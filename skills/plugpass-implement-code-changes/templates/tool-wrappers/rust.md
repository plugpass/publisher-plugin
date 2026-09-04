# Rust scaffolding template

The developer's server becomes an **OAuth-protected resource server**: an axum middleware gates `/mcp` (validating the bearer against Plugpass's JWKS with `jsonwebtoken`), an axum route serves the RFC 9728 PRM document, and rmcp's streamable-HTTP service runs behind them. Standalone source, no platform package. Version pins: `rmcp = { version = "2.2", features = ["server", "transport-streamable-http-server"] }`, `jsonwebtoken = { version = "10", default-features = false, features = ["rust_crypto"] }` (**the feature pin is load-bearing — bare `jsonwebtoken = "10"` has no crypto provider and panics at runtime on the first verify**), `axum = "0.8"`, `http = "1"`, `reqwest = { version = "0.13", features = ["json"] }`, `schemars = "1"` (must be 1.x to match rmcp's), `serde`, `serde_json`, `tokio`.

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

#[derive(Clone)]
pub struct PlugpassConfig {
    pub issuer: String,
    pub jwks_url: String,
    pub entitlement_api_origin: String,
    pub resource_url: String,
}

pub fn plugpass_config() -> PlugpassConfig {
    PlugpassConfig {
        issuer: ISSUER.to_string(),
        jwks_url: JWKS_URL.to_string(),
        entitlement_api_origin: ENTITLEMENT_API_ORIGIN.to_string(),
        resource_url: RESOURCE_URL.to_string(),
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

The entitlement client (`entitlement(bearer, body, op, cfg)` POSTing `{origin}/entitlement/{op}` via `http_client()` — 5s timeout per attempt with ONE retry after a 2s backoff on a transport error or a 5xx (4xx terminal, never retried); HTTP 401 → `ReauthRequired`, other non-200 or a failure after the retry → error), the `EntitlementResult` enum (its `NonAuthorized` variant carries `result_text: String`), the `non_authorized` rendering (the `result_text` emitted verbatim as the tool's text — a single-field pipe, never parsed or re-serialized), and the unavailable-deny `CallToolResult` follow the wire contract in TOOLS.md — never fail open.

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
        return Ok(unavailable_deny()); // unreachable behind the gate; fail closed
    };
    match check_premium_access_upstream(&identity.bearer, &args, &self.cfg).await {
        Ok(Upstream::Ok { result_text }) =>
            Ok(CallToolResult::success(vec![ContentBlock::text(result_text)])), // verbatim
        Ok(Upstream::ReauthRequired) => {
            if let Some(sig) = parts.extensions.get::<ReauthSignal>() { sig.0.store(true, Ordering::Relaxed); }
            Ok(CallToolResult::error(vec![ContentBlock::text("Re-authentication required.")])) // discarded by the swap
        }
        Err(_) => Ok(unavailable_deny()),
    }
}
```

**Solo paid tool wrapper** (consume-on-invocation; no `auth_token` parameter on this path): read `Identity` from the parts extensions, call `track_usage` (`feature_id` = the tool's own plugpass-component-id, matching its `_meta`; the id's `tool_` prefix carries the feature type), branch: `Ok` → run the body scoped to `identity.sub`; `NonAuthorized` → its `result_text` verbatim as the text result; `ReauthRequired` → set the signal + placeholder; error → the unavailable deny. Stamp `_meta` with the macro's `meta` attribute (`#[tool(name = "…", description = "…", meta = component_meta("<plugpass_id>"))]` where `component_meta` builds `Meta` carrying `plugpass_component_id`). Wrapped tools return `Result<CallToolResult, ErrorData>` — never a `Json<T>` typed result (no output schema: paywall/reauth responses are text-only). Re-derive the macro's `annotations(…)`: metering makes the tool neither read-only nor idempotent, whatever it was before the wrap — `read_only_hint = false, idempotent_hint = false`, the other two keeping the tool's own values; see TOOLS.md § Tool annotations.

**Paired-tool add side** (`operation: add`): same shape but `feature_id` = the add tool's `database_record.custom_entitlement_id` (its `custom_` prefix carries the feature type; **not** `database_record.plugpass_id`, and **not** the tool's own `_meta` id), always **`check_remaining`**, passing the user's current count from the developer's own store (scoped to `sub`) as `current_count` — see TOOLS.md → Reading the user's current count.

**Identity tool** (the paired remove side, or any per-user free tool): no Entitlement API call — read `Identity.sub` from the parts extensions and scope the body to it.

**Placement guidance.** The handler struct keeps rmcp's macro requirements: a `tool_router: ToolRouter<Self>` field initialized with `Self::tool_router()`, `#[tool_router]` on the impl block, `#[tool_handler]` on the `ServerHandler` impl; args structs derive `serde::Deserialize + schemars::JsonSchema` (schemars 1.x). rmcp's config/model types are `#[non_exhaustive]` — always builders or mutate-a-`Default`. Invariants regardless of layout: the stateless+JSON pair on the transport config; the gate layered on the MCP service covering every method; the PRM route outside the gate; extensions read through `Parts`, never top-level.
