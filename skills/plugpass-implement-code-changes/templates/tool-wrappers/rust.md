# Rust scaffolding template

The publisher's server becomes an **OAuth-protected resource server**: an axum middleware gates `/mcp` (validating the bearer against Plugpass's JWKS with `jsonwebtoken`), an axum route serves the RFC 9728 PRM document, and rmcp's streamable-HTTP service runs behind them. Standalone source, no platform package. Version pins: `rmcp = { version = "2.2", features = ["server", "transport-streamable-http-server"] }`, `jsonwebtoken = { version = "10", default-features = false, features = ["rust_crypto"] }` (**the feature pin is load-bearing — bare `jsonwebtoken = "10"` has no crypto provider and panics at runtime on the first verify**), `axum = "0.8"`, `http = "1"`, `reqwest = { version = "0.13", features = ["json"] }`, `schemars = "1"` (must be 1.x to match rmcp's), `serde`, `serde_json`, `tokio`.

**Two structural decisions carry the whole design — never undo them:**

1. **`.with_stateful_mode(false).with_json_response(true)` on the transport config, always — as a pair.** In stateless+JSON mode the HTTP response object isn't built until the tool handler finished, so the gate middleware holds the complete response and can swap in a `401` when a tool discovered mid-call that the bearer is revoked. `json_response` has **no effect in stateful mode** (SSE streams the `200` before the tool runs) — dropping either flag silently kills the reauth mechanism.
2. **The audience is the baked `RESOURCE_URL`, never the request host** — the JWT `aud` pin, the PRM `resource`, and the challenge's `resource_metadata` all derive from it. This is what lets a locally-listening server accept real bearers minted for its public URL.

```rust
// premium_feature_access_check.rs — written once per server. Standalone.
use std::collections::HashMap;
use std::sync::atomic::{AtomicBool, Ordering};
use std::sync::{Arc, Mutex, OnceLock};
use std::time::{Duration, Instant};

use jsonwebtoken::jwk::JwkSet;
use jsonwebtoken::{decode, decode_header, Algorithm, DecodingKey, Validation};

pub const MCP_PATH: &str = "/mcp";

// Plugpass endpoints for this server.
const ISSUER: &str = "<plugpass_issuer>";
const JWKS_URL: &str = "<plugpass_jwks_url>";
const ENTITLEMENT_API_ORIGIN: &str = "<entitlement_api_origin>";
// This server's own public MCP URL — the JWT `aud` pin and the PRM `resource`.
const RESOURCE_URL: &str = "<this server's RESOURCE_URL>";
// Where the Plugpass paywall for MCP Apps widgets loads from (the `ui_paywall`
// layer — only on a server whose directive names it).
const PAYWALL_SCRIPT_URL: &str = "<paywall_script_url>";

#[derive(Clone)]
pub struct PlugpassConfig {
    pub issuer: String,
    pub jwks_url: String,
    pub entitlement_api_origin: String,
    pub resource_url: String,
    pub paywall_script_url: String,
}

pub fn plugpass_config() -> PlugpassConfig {
    PlugpassConfig {
        issuer: ISSUER.to_string(),
        jwks_url: JWKS_URL.to_string(),
        entitlement_api_origin: ENTITLEMENT_API_ORIGIN.to_string(),
        resource_url: RESOURCE_URL.to_string(),
        paywall_script_url: PAYWALL_SCRIPT_URL.to_string(),
    }
}

// RFC 9728 path-aware PRM URL: origin + /.well-known/oauth-protected-resource + /mcp.
pub fn prm_url(cfg: &PlugpassConfig) -> String {
    let origin = cfg.resource_url.trim_end_matches(MCP_PATH);
    format!("{origin}/.well-known/oauth-protected-resource{MCP_PATH}")
}

// Per-request values the gate inserts into the request extensions; rmcp carries
// them (inside http::request::Parts) into every tool handler. Arc-shared, so
// the tool's clone and the middleware's clone see the same flag.
#[derive(Clone)]
pub struct Identity { pub sub: String, pub bearer: String }
#[derive(Clone)]
pub struct ReauthSignal(pub Arc<AtomicBool>);

// JWKS cache: 4h TTL, refetch on an unknown kid rate-limited to one per 10s
// (key rotation), retry the lookup once after the refetch. Locks are never
// held across an await.
struct CachedJwks { set: JwkSet, fetched_at: Instant, last_refetch: Instant }
static JWKS_CACHE: Mutex<Option<HashMap<String, CachedJwks>>> = Mutex::new(None);
const JWKS_TTL: Duration = Duration::from_secs(4 * 60 * 60);

fn http_client() -> &'static reqwest::Client {
    static CLIENT: OnceLock<reqwest::Client> = OnceLock::new();
    CLIENT.get_or_init(|| reqwest::Client::builder().timeout(Duration::from_secs(5)).build().unwrap())
}

async fn decoding_key_for(jwks_url: &str, kid: &str) -> Option<DecodingKey> {
    /* …fetch/cache logic: fresh cache hit → find(kid); miss or stale →
       (rate-limited) refetch via http_client().get(jwks_url) → JwkSet json →
       find(kid) → DecodingKey::from_jwk(jwk).ok() (handles OKP/Ed25519)… */
}

// EdDSA-pinned, issuer exact, audience = the baked RESOURCE_URL, exp+sub
// required. Returns the user id, or None on any failure.
pub async fn verify_bearer(token: &str, cfg: &PlugpassConfig) -> Option<String> {
    let header = decode_header(token).ok()?;
    let key = decoding_key_for(&cfg.jwks_url, header.kid.as_deref()?).await?;
    let mut validation = Validation::new(Algorithm::EdDSA);
    validation.set_issuer(&[&cfg.issuer]);
    validation.set_audience(&[&cfg.resource_url]);
    validation.set_required_spec_claims(&["exp", "aud", "iss", "sub"]);
    let data = decode::<serde_json::Map<String, serde_json::Value>>(token, &key, &validation).ok()?;
    data.claims.get("sub")?.as_str().map(str::to_owned)
}
```

The entitlement client (`entitlement(bearer, body, op, cfg)` POSTing `{origin}/entitlement/{op}` via `http_client()` — 5s timeout per attempt with ONE retry after a 2s backoff on a transport error or a 5xx (4xx terminal, never retried); HTTP 401 → `ReauthRequired`, other non-200 or a failure after the retry → `Unavailable`), the `EntitlementResult` enum (its `NonAuthorized` variant carries `result_text: String`; its `Unavailable` variant carries nothing), the `non_authorized` rendering below, and the unavailable-grant `CallToolResult` follow the wire contract in TOOLS.md.

```rust
use rmcp::model::{CallToolResult, ContentBlock, JsonObject, Meta};
use serde_json::Value;

// The call a wrapper denied: the tool's name and the arguments it was called
// with (the handler's parsed args, serialized as the caller sent them).
pub struct DeniedCall {
    pub name: String,
    pub arguments: Value,
}

// The marker a UI-backed tool's denial carries when the client renders widgets.
pub const PAYWALL_UI_MARKER: &str = "PLUGPASS_PAYWALL_UI=true";

// non_authorized → result_text verbatim — a single-field pipe (the trigger keys
// inside it auto-fire the plugin's access-handler skill). Never parse, reformat,
// or re-serialize it.
//
// Two things ride beside it for the Plugpass paywall an MCP Apps widget shows
// (both inert everywhere else):
//   - The denial NAMES the call it denied, on the result's `_meta` — which
//     hosts pass to a widget and never show the model — so the paywall can
//     replay the same call once the user has upgraded, and says whether a
//     widget may call the tool at all (widget_callable below): a host refuses
//     a widget's call to a model-only tool, so the paywall then hands the
//     retry to the conversation instead.
//   - The marker: when the denied tool is UI-backed (it renders a widget) AND
//     the client renders widgets (paywall_ui below), a second text block
//     carries PLUGPASS_PAYWALL_UI=true. The widget's paywall is then the one
//     asking the user, and the plugin's access-handler skill posts nothing
//     beside it. The composed text stays untouched in its own block.
// Both read the tool's own registered `Meta` and the request's, at runtime.
pub fn non_authorized_tool_response(result_text: &str, call: DeniedCall, tool_meta: &Meta, request_meta: &Meta) -> CallToolResult {
    let mut content = vec![ContentBlock::text(result_text)];
    if paywall_ui(tool_meta, request_meta) {
        content.push(ContentBlock::text(PAYWALL_UI_MARKER));
    }
    let mut response = CallToolResult::success(content);
    let mut meta = JsonObject::new();
    meta.insert(
        "plugpass_denied_call".to_string(),
        serde_json::json!({ "name": call.name, "arguments": call.arguments, "widget_callable": widget_callable(tool_meta) }),
    );
    response.meta = Some(Meta(meta));
    response
}

// Whether the calling client renders MCP Apps widgets: it declared the UI
// extension among the client capabilities every request carries in its `_meta`
// (`io.modelcontextprotocol/clientCapabilities` — the stateless protocol's
// per-request declaration; this server keeps no session, so the initialize
// handshake is not a source). Read off the request `Meta` a tool handler
// extracts. A client that declares nothing renders nothing, and gets no marker.
const CLIENT_CAPABILITIES_META_KEY: &str = "io.modelcontextprotocol/clientCapabilities";
const UI_EXTENSION_ID: &str = "io.modelcontextprotocol/ui";

pub fn client_renders_widgets(meta: &Meta) -> bool {
    meta.0
        .get(CLIENT_CAPABILITIES_META_KEY)
        .and_then(|capabilities| capabilities.get("extensions"))
        .and_then(|extensions| extensions.get(UI_EXTENSION_ID))
        .is_some_and(Value::is_object)
}

// Whether the widget's paywall is the one asking on this denial: the tool
// renders a widget (its own registration's `Meta` declares `ui.resourceUri`)
// AND the client renders widgets. Read off the registration at runtime, so a
// tool that gains or loses its widget changes nothing here.
pub fn paywall_ui(tool_meta: &Meta, request_meta: &Meta) -> bool {
    tool_meta.0.get("ui").and_then(|ui| ui.get("resourceUri")).is_some_and(Value::is_string)
        && client_renders_widgets(request_meta)
}

// Who may call the tool, off the same registration: an undeclared visibility
// means the model and a widget both may; a declared list means exactly its
// members. A widget's paywall replays a denied call itself only when it may.
pub fn widget_callable(tool_meta: &Meta) -> bool {
    match tool_meta.0.get("ui").and_then(|ui| ui.get("visibility")).and_then(Value::as_array) {
        Some(visibility) => visibility.iter().any(|v| v.as_str() == Some("app")),
        None => true,
    }
}
```

**`main.rs` composition** — PRM route public; the gate layered on the nested MCP service:

```rust
use rmcp::transport::streamable_http_server::{
    session::never::NeverSessionManager, StreamableHttpServerConfig, StreamableHttpService,
};

let cfg = plugpass_config();
let mcp_service = StreamableHttpService::new(
    move || Ok(MyServer::new(cfg.clone(), /* …shared state via Arc… */)), // fresh handler per POST
    Arc::new(NeverSessionManager::default()),
    StreamableHttpServerConfig::default()   // #[non_exhaustive] — builders only, never a struct literal
        .with_stateful_mode(false)          // required: lets a revoked-bearer tool swap in a 401
        .with_json_response(true)
        .with_sse_keep_alive(None)
        .disable_allowed_hosts(),           // default allows loopback Hosts only — a public deploy 403s
                                            // without this (defensible: every request is bearer-gated)
);
let protected = Router::new()
    .nest_service(MCP_PATH, mcp_service)
    .layer(middleware::from_fn_with_state(cfg.clone(), bearer_gate));
let app = Router::new()
    .route("/.well-known/oauth-protected-resource/mcp", get(prm_document))
    .merge(protected);
axum::serve(tokio::net::TcpListener::bind(("0.0.0.0", port)).await?, app).await?;
```

```rust
// The PRM document (unauthenticated) and the gate.
async fn prm_document(State(cfg): State<PlugpassConfig>) -> Json<serde_json::Value> {
    Json(serde_json::json!({
        "resource": cfg.resource_url,
        "authorization_servers": [cfg.issuer],
        "bearer_methods_supported": ["header"],
    }))
}

fn challenge_401(cfg: &PlugpassConfig, description: &str) -> Response {
    (
        StatusCode::UNAUTHORIZED,
        [(header::WWW_AUTHENTICATE, format!(
            "Bearer realm=\"{}\", error=\"invalid_token\", error_description=\"{description}\", resource_metadata=\"{}\"",
            cfg.resource_url, prm_url(cfg)
        ))],
        Json(serde_json::json!({ "error": "invalid_token", "error_description": description })),
    ).into_response()
}

async fn bearer_gate(State(cfg): State<PlugpassConfig>, mut request: Request, next: Next) -> Response {
    let Some(token) = bearer_from(request.headers()) else {
        return challenge_401(&cfg, "Missing bearer token");
    };
    let Some(sub) = verify_bearer(&token, &cfg).await else {
        return challenge_401(&cfg, "Token invalid or expired");
    };
    let signal = ReauthSignal(Arc::new(AtomicBool::new(false)));
    request.extensions_mut().insert(Identity { sub, bearer: token });
    request.extensions_mut().insert(signal.clone());
    let response = next.run(request).await; // stateless+json: resolves only after the tool finished
    if signal.0.load(Ordering::Relaxed) {
        return challenge_401(&cfg, "Access token no longer valid");
    }
    response
}
```

**Per-request identity inside a tool**: extract `Extension(parts): Extension<http::request::Parts>` (the `rmcp::handler::server::common::Extension` extractor — it moved out of `::tool::` in rmcp 2.2) and read the gate's values **nested inside** the parts — `parts.extensions.get::<Identity>()` / `parts.extensions.get::<ReauthSignal>()`. rmcp injects the whole `http::request::Parts` as one context extension, so a top-level `Extension<Identity>` will NOT find them.

**The check proxy tool (check host only).** A pure pipe — never parse or reformat `result_text`:

```rust
#[derive(serde::Deserialize, schemars::JsonSchema)]
struct CheckPremiumAccessArgs {
    plugin_id: String,
    feature_id: String,
    plugin_version: String,
}

// `description` must be a STRING LITERAL — rmcp's `#[tool]` macro rejects a const
// path ("Unexpected type `path`") — inline the description as a literal.
#[tool(
    name = "{CheckToolName}",
    description = "<verbatim from TOOLS.md, inline as a literal>",
    annotations(
        read_only_hint = false,
        destructive_hint = false,
        idempotent_hint = false,
        open_world_hint = false
    )
)]
async fn check_premium_access(
    &self,
    Parameters(args): Parameters<CheckPremiumAccessArgs>,
    Extension(parts): Extension<http::request::Parts>,
) -> Result<CallToolResult, ErrorData> {
    let Some(identity) = parts.extensions.get::<Identity>().cloned() else {
        return Ok(unavailable_check_grant()); // unreachable behind the gate
    };
    match check_premium_access_upstream(&identity.bearer, &args, &self.cfg).await {
        Ok(Upstream::Ok { result_text }) =>
            Ok(CallToolResult::success(vec![ContentBlock::text(result_text)])), // verbatim
        Ok(Upstream::ReauthRequired) => {
            if let Some(sig) = parts.extensions.get::<ReauthSignal>() { sig.0.store(true, Ordering::Relaxed); }
            Ok(CallToolResult::error(vec![ContentBlock::text("Re-authentication required.")])) // discarded by the swap
        }
        Err(_) => Ok(unavailable_check_grant()), // Plugpass could not answer → the check grants
    }
}
```

**Solo paid tool wrapper** (consume-on-invocation; no `auth_token` parameter on this path): read `Identity` from the parts extensions, call `track_usage` (`feature_id` = the tool's own plugpass-component-id, matching its `_meta`; the id's `tool_` prefix carries the feature type), branch: `Ok` → run the body scoped to `identity.sub`; `NonAuthorized` → `non_authorized_tool_response(&result_text, DeniedCall { name: "<tool name>".into(), arguments: serde_json::to_value(&args).unwrap_or(Value::Null) }, &paid_tool_meta(), &meta)` — the args struct derives `serde::Serialize` beside `Deserialize` (`#[serde(skip_serializing_if = "Option::is_none")]` on each optional field, so the echo is the call as sent), every wrapped tool's handler extracts `meta: Meta` (rmcp's request-`_meta` extractor, beside `Parameters` and `Extension`), and the renderer reads the tool's own registered `Meta` (its widget, who may call it) and the request's — never a baked per-tool constant; `ReauthRequired` → set the signal + placeholder; `Unavailable` → run the body (it consumed nothing and grants). Stamp `_meta` with the macro's `meta` attribute, `#[tool(name = "…", description = "…", meta = paid_tool_meta())]`, where `fn paid_tool_meta() -> Meta` builds the `Meta` carrying `plugpass_component_id` (and, on a UI-backed tool, its `ui: { resourceUri, visibility }`) — the one function both the attribute and the marker rule read. Wrapped tools return `Result<CallToolResult, ErrorData>` — never a `Json<T>` typed result (no output schema: paywall/reauth responses are text-only). Re-derive the macro's `annotations(…)`: metering makes the tool neither read-only nor idempotent, whatever it was before the wrap — `read_only_hint = false, idempotent_hint = false`, the other two keeping the tool's own values; see TOOLS.md § Tool annotations.

**Paired-tool add side** (`operation: add`): same shape but `feature_id` = the add tool's `database_record.custom_entitlement_id` (its `custom_` prefix carries the feature type; **not** `database_record.plugpass_id`, and **not** the tool's own `_meta` id), always **`check_remaining`**, passing the user's current count from the publisher's own store (scoped to `sub`) as `current_count` — see TOOLS.md → Reading the user's current count.

**Identity tool** (the paired remove side, or any per-user free tool): no Entitlement API call — read `Identity.sub` from the parts extensions and scope the body to it.

**The in-widget paywall (`ui_paywall`, servers that render widgets).** The helper below, in `premium_feature_access_check.rs`, is applied to every resource the server reads out whose MIME type is `text/html;profile=mcp-app`: the paywall script tag goes first in `<head>`, and the script's origin joins the resource's `resourceDomains`. The widget HTML itself is never edited.

```rust
// The CSP a UI resource declares on its contents (`_meta.ui.csp`).
#[derive(Clone, Default, serde::Serialize)]
#[serde(rename_all = "camelCase")]
pub struct UiResourceCsp {
    #[serde(skip_serializing_if = "Vec::is_empty")]
    pub connect_domains: Vec<String>,
    #[serde(skip_serializing_if = "Vec::is_empty")]
    pub resource_domains: Vec<String>,
    #[serde(skip_serializing_if = "Vec::is_empty")]
    pub frame_domains: Vec<String>,
    #[serde(skip_serializing_if = "Vec::is_empty")]
    pub base_uri_domains: Vec<String>,
}

// Loads the Plugpass paywall into a widget's HTML on its way out: the script
// tag first in <head> (ahead of the widget's own code; prepended to the
// document when it has no <head>), its origin added to `resourceDomains` so
// the sandbox lets it load.
pub fn with_paywall(html: &str, csp: &UiResourceCsp, cfg: &PlugpassConfig) -> (String, UiResourceCsp) {
    let tag = format!("<script src=\"{}\"></script>", cfg.paywall_script_url);
    let injected = match head_open_end(html) { // the byte offset just past the opening <head …> tag, if any
        Some(end) => format!("{}{}{}", &html[..end], tag, &html[end..]),
        None => format!("{tag}{html}"),
    };
    let origin = origin_of(&cfg.paywall_script_url); // scheme + host (+ port), no path
    let mut widened = csp.clone();
    if !widened.resource_domains.iter().any(|d| *d == origin) {
        widened.resource_domains.push(origin);
    }
    (injected, widened)
}
```

Every UI resource read passes through it — `ServerHandler` overrides beside `get_info`, which enables resources (`ServerCapabilities::builder().enable_tools().enable_resources().build()`):

```rust
async fn list_resources(&self, _request: Option<PaginatedRequestParams>, _context: RequestContext<RoleServer>) -> Result<ListResourcesResult, ErrorData> {
    Ok(ListResourcesResult::with_all_items(vec![
        Resource::new("ui://<plugin>/<widget>", "<widget>")
            .with_title("…")
            .with_description("…")
            .with_mime_type("text/html;profile=mcp-app"),
    ]))
}

async fn read_resource(&self, request: ReadResourceRequestParams, _context: RequestContext<RoleServer>) -> Result<ReadResourceResult, ErrorData> {
    if request.uri != "ui://<plugin>/<widget>" {
        return Err(ErrorData::resource_not_found(format!("Unknown resource: {}", request.uri), None));
    }
    let (html, csp) = with_paywall(WIDGET_HTML, &widget_csp(), &self.cfg);
    let mut meta = JsonObject::new();
    meta.insert("ui".to_string(), serde_json::json!({ "csp": csp }));
    Ok(ReadResourceResult::new(vec![ResourceContents::TextResourceContents {
        uri: request.uri,
        mime_type: Some("text/html;profile=mcp-app".to_string()),
        text: html,
        meta: Some(Meta(meta)),
    }]))
}
```

**Placement guidance.** The handler struct keeps rmcp's macro requirements: a `tool_router: ToolRouter<Self>` field initialized with `Self::tool_router()`, `#[tool_router]` on the impl block, `#[tool_handler(router = self.tool_router)]` on the `ServerHandler` impl (the macro's default rebuilds a router per call and leaves the field unread); args structs derive `serde::Deserialize + schemars::JsonSchema` (schemars 1.x). rmcp's config/model types are `#[non_exhaustive]` — always builders or mutate-a-`Default`. Invariants regardless of layout: the stateless+JSON pair on the transport config; the gate layered on the MCP service covering every method; the PRM route outside the gate; extensions read through `Parts`, never top-level; on a server whose directive names `ui_paywall`, every `text/html;profile=mcp-app` resource read passes through `with_paywall`.
