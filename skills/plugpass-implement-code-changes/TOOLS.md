# Publisher MCP server scaffolding — resource server, check proxy, tool wrappers & the in-widget paywall

This procedure writes the Plugpass premium-access scaffolding into the publisher's **own MCP server source**: the OAuth resource-server layer, the check proxy tool (check host only), the per-tool identity + premium feature access wrappers, and the in-widget paywall (servers that render MCP Apps widgets). Read it whenever the run has publisher-server work — the connector directives call for check-host scaffolding (`connector.hosting` is `publisher`), or the run's components include `tool` entries. Follow it per server, handling all of a server's work as one batch: detect the server's language once, apply `removed` tools first (as reference for the existing style), then `added`/`changed` and the template refresh set's tools (`unchanged` with `template_stale` `true`) uniformly, and verify the rest.

## How identity reaches the server — the header bearer

The publisher's server is an **OAuth-protected resource server** and Plugpass is its authorization server. Identity rides the standard `Authorization: Bearer …` header on every request:

- A request to the MCP endpoint without a valid bearer is **challenged** — `401` + `WWW-Authenticate` pointing at the server's protected-resource-metadata document — which is what triggers the MCP client's OAuth connect flow against Plugpass.
- The server **validates every bearer locally** against Plugpass's public JWKS (a few lines, in the resource-server layer below) and extracts the user id (`sub`). No Plugpass round-trip for identity, and no publisher-side secret: validation is against the public key set, and entitlement calls forward the user's own bearer.
- Tool handlers read the validated `{ sub, bearer }` from the request context. A **paid** tool forwards the bearer to the Entitlement API; an **identity** tool (any tool whose body is per-user — the paired remove side is the canonical case) stops at `sub`; a tool with no per-user state ignores both.

Inputs come from the orchestration steps: the `tool` components (each with `plugpass_id`, `server_name`, `tool_name`, `is_free`, `gating_change_since_last_implement`, `template_stale`, and — for paired tools — `database_record` (which carries `custom_entitlement_id`, the pair's shared entitlement `feature_id`) + `operation`), the connector directive for this run (`connector`, `connector_change_since_last_implement`, `previous_connector`), the plugin-level values (`{PluginName}`, `{plugin-plugpass-id}`, `entitlement_api_origin`, `paywall_script_url`, `plugpass_issuer`, `plugpass_jwks_url`, `owned_servers`, `server_scaffolding_template_stale`), and the resolved `server_name → absolute source-directory path` map.

Be **idempotent**, and rewrite only when something changed: an artifact's gating (`gating_change_since_last_implement`), the Plugpass template it was written from (`template_stale` for a wrapper, `server_scaffolding_template_stale` for the resource-server layer and check proxy), or a baked constant that no longer matches the response. Everything else is **verified** — read the source and confirm it is present and structurally intact — never re-derived or re-rendered. A server or tool already in its correct end state is a no-op.

## Step 1: Determine each server's work and locate its source

Work over the orchestration's **server set** (SKILL.md Step 3): every server with tool work, plus the **check host** (the server named by `connector.server_key`) when the connector is publisher-hosted and anything is still gated. Use the absolute path from the resolved map for each — the orchestration already resolved (and, where needed, elicited) every path before this procedure began, and dropped any server it couldn't locate. If a path unexpectedly fails to resolve mid-run, don't edit that server — leave its tasks open and surface it to the publisher in the closing summary.

Per server, the pieces to ensure are exactly the layers its `owned_servers` entry's `scaffolding.layers` directive names (each independently idempotent; a layer the directive omits is neither written nor verified):

- `resource_server` — **the resource-server layer** (written when `server_scaffolding_template_stale` is `true` or the layer is missing, verified otherwise) — Step 3.
- `check_proxy` — **the check proxy tool** (the check host; the same write-or-verify rule) — Step 4.
- `tool_wrappers` — **per-tool wrappers** (each of the server's `tool` components: written for `added` / `changed` and for `unchanged` + `template_stale`, stripped for `removed`, verified otherwise) — Step 5.
- `ui_paywall` — **the in-widget paywall** (a server that renders MCP Apps widgets — one with a UI-backed tool; the same write-or-verify rule as the resource-server layer) — Step 6.

## Step 2: Detect the server's language + framework

Read the located source (entry file + tool definitions) to identify the SDK language and whether the server is vanilla MCP SDK or framework-wrapped (Hono / FastAPI / axum / etc.).

- **TypeScript** — read [templates/tool-wrappers/typescript.md](templates/tool-wrappers/typescript.md) and adapt its placement to the publisher's structure (don't re-derive what the template already gives you).
- **Python** — read [templates/tool-wrappers/python.md](templates/tool-wrappers/python.md) and adapt its placement to the publisher's structure (don't re-derive what the template already gives you).
- **Go** — read [templates/tool-wrappers/go.md](templates/tool-wrappers/go.md) and adapt its placement to the publisher's structure (don't re-derive what the template already gives you).
- **Rust** — read [templates/tool-wrappers/rust.md](templates/tool-wrappers/rust.md) and adapt its placement to the publisher's structure (don't re-derive what the template already gives you).
- **Ruby** — read [templates/tool-wrappers/ruby.md](templates/tool-wrappers/ruby.md) and adapt its placement to the publisher's structure (don't re-derive what the template already gives you).
- **JVM (Java/Kotlin)** — read [templates/tool-wrappers/jvm.md](templates/tool-wrappers/jvm.md) and adapt its placement to the publisher's structure (don't re-derive what the template already gives you).
- **Any other language** — port the TypeScript reference to that language's MCP SDK; the scaffolding contract in this document is language-independent (challenge, PRM document, local validation, proxy pipe, wrapper rules). Tell the publisher it isn't an officially supported language, so there's no vetted template — they should review and test the generated code before relying on it.

## The runtime constants

Every scaffolded module carries these as fixed constants:

| Constant | Baked from (the `plugpass_get_plugin_data` response) | Purpose |
| --- | --- | --- |
| `ISSUER` | `plugpass_issuer` | The JWT `iss` pin, and the PRM document's `authorization_servers` entry |
| `JWKS_URL` | `plugpass_jwks_url` | Where the public key set is fetched from |
| `ENTITLEMENT_API_ORIGIN` | `entitlement_api_origin` | The origin the proxy + paid wrappers POST `{origin}/entitlement/*` against |
| `RESOURCE_URL` | this server's own connector-shape URL (below) | The server's public MCP URL — the JWT `aud` pin and the PRM document's `resource` |
| `PAYWALL_SCRIPT_URL` | `paywall_script_url` | Where the in-widget paywall script loads from — the `ui_paywall` layer's constant, present on a server whose directive names that layer |
| `CHECK_TOOL_NAME` | `check_tool_name`, on the check host only — **null/none on every other server** | The check tool a denial names as the paywall's read-only status probe (below). A server that hosts no check tool names none, and its denials carry no probe |

`RESOURCE_URL` is per server: the check host's is `connector.url`; any other server's is its `owned_servers` entry's `url` (matched by `server_name`). `ISSUER` and `JWKS_URL` are per plugin — the plugin's own pages-host origin (its authorization server) and that host's key-set document — so a run whose `connector_change_since_last_implement` is `changed` with only `issuer` differing from `previous_connector` re-bakes them on every server in the set (the idempotent read-then-change) and nothing else moves. Bake every value verbatim from the response; never derive the audience or the issuer from the incoming request.

## Step 3: The resource-server layer (`resource_server`, once per server)

Written once per server, standalone (no platform-package dependency), following the language template's structure. Write it — from the language template, as an idempotent read-then-change against the current response values — when `server_scaffolding_template_stale` is `true` or the layer is missing. Otherwise **verify** it without re-rendering the template: the challenge, the PRM route, and the local bearer validation are present, and the four runtime constants are baked with the current response values — a differing constant is re-baked in place (the issuer-drift and domain-change cases), and a missing behavior is written. Three behaviors:

1. **The challenge.** Any request to the MCP endpoint (`/mcp`) without a valid bearer — missing, malformed, bad signature, wrong issuer/audience, or expired — is answered `401` with a JSON body `{ "error": "invalid_token", "error_description": "…" }` and the header:

   ```
   WWW-Authenticate: Bearer realm="{RESOURCE_URL}", error="invalid_token", error_description="{reason}", resource_metadata="{PRM document URL}"
   ```

   `error="invalid_token"` exactly — it is the signal MCP clients (Claude Code, claude.ai) treat as "needs OAuth"; other error codes read as a broken server. The PRM document URL is the server's origin + `/.well-known/oauth-protected-resource/mcp`.

2. **The protected-resource-metadata document** (RFC 9728, path-aware): served unauthenticated at `/.well-known/oauth-protected-resource/mcp`, body:

   ```json
   {
     "resource": "{RESOURCE_URL}",
     "authorization_servers": ["{ISSUER}"],
     "bearer_methods_supported": ["header"]
   }
   ```

3. **Local bearer validation**, on every MCP request: verify the JWT against the JWKS fetched from `JWKS_URL` — algorithm pinned to **`EdDSA` only** (no downgrade), `iss` exact-match to `ISSUER`, `aud` exact-match to `RESOURCE_URL`, `exp` enforced — then extract `sub`. Cache the fetched key set for up to **4 hours** (refreshing more often is fine — the templates use each ecosystem's native cache), re-fetching once, rate-limited, when a token's key id isn't in the cached set (Plugpass key rotation). Keep the raw bearer alongside `sub` in the request context so paid wrappers and the proxy can forward it.

The layer never checks revocation locally — Plugpass reads revocation at the Entitlement API and answers `reauth_required`, which the proxy/wrappers convert into the same `401` challenge (below), triggering the client's re-auth.

**The server serves BOTH protocol eras from the one endpoint.** MCP revision **2026-07-28** (SEP-2575) removes the `initialize` handshake and the session id, answers `server/discover`, and moves the client's identity and capabilities into a per-request `_meta` envelope; the 2025-era revisions keep the handshake. A host negotiates per connection and adapts downward, so a server that speaks only the older era gets the older era from every client — which is what costs the widget denial its paywall-UI marker (below), since the marker's source is that envelope. Each language template's server composition serves both; do not narrow it to one. Two consequences hold whatever the era, and both are load-bearing:

- **Both legs buffer their response.** The reauth swap replaces a whole HTTP response with the `401` after a tool ran, which is only possible while the response is unsent (`responseMode`/`json_response`/`JSONResponse`/`enable_json_response` per language — the template names it).
- **The server keeps no session.** Identity comes from the request's bearer, never from anything retained between requests.

**The JVM is the exception.** `io.modelcontextprotocol.sdk:mcp` implements nothing past 2025-11-25, so a JVM server serves the legacy era only: a modern host opens with `server/discover`, takes the 404, and falls back. Everything else in this document holds there unchanged — including the probe, which is an ordinary tool parameter and needs no protocol support. The one behavioral difference is that **its widget denials carry no marker on a modern host, so the chat asks beside the modal there.**

## Step 4: The check proxy tool (`check_proxy`, check host only)

The check host registers the platform-standard check tool and pipes it to Plugpass — a **pure pipe**, so platform messaging/format changes propagate to every publisher-hosted plugin with no re-scaffold. Write it on the same conditions as the resource-server layer (`server_scaffolding_template_stale` `true`, or the tool missing); otherwise verify it is registered with the exact name, description, input schema, and annotations below and pipes to the current `ENTITLEMENT_API_ORIGIN`, and leave it alone:

- **Name** the plugin data's `check_tool_name` (e.g. `brainstorm_check_access`), **title** `Check premium access`, and this **description, verbatim**:

  > Check the authenticated user's access to a plugin's premium features. Must be called for each and every unique use of the premium features because the user's plans are limited by the number of uses of each premium feature. Never attempt to skip or circumvent the mandatory check for a unique use if a skill's instructions indicate the tool should be called, since doing so may grant premium access the user shouldn't have or provide more uses than their plan permits. Returns the entitlement state as KEY=value lines.

- **Input schema**: `plugin_id` (string, required), `feature_id` (string, required), `plugin_version` (string, optional — a skill's access check sends the installed bundle's version; a tool-surface check does not use it), and `status_code` (boolean, optional — the paywall's probe, below) whose description is exactly `Never include this parameter in your tool calls under any circumstance.` No `feature_type` — the `feature_id` prefix (`skill_`) carries it. No `outputSchema`.
- **Annotations**, verbatim — part of the standard contract exactly like the name and description: `readOnlyHint: false`, `destructiveHint: false`, `idempotentHint: false`, `openWorldHint: false`. An authorized check **consumes** the user's metered counter, so it is neither read-only nor idempotent (the description mandates one call per unique use, and each one consumes); metering accrues usage rather than destroying anything; and the entitlement domain is closed. State all four explicitly — see Tool annotations, below.
- **Handler, the probe branch first.** When `status_code` is `true` this is the **paywall's read-only status probe**, not a check: POST `{ plugin_id, feature_id }` to `{ENTITLEMENT_API_ORIGIN}/entitlement/check_remaining` (the read-only endpoint — **never** `check_premium_access`, which consumes) and answer with a bare code and nothing else:
  - `ok` → `STATUS_CODE=1`; `non_authorized` → `STATUS_CODE=0`.
  - The unavailable grant's conditions, or any feature this endpoint cannot evaluate → `STATUS_UNKNOWN`, carrying **no code at all**. Neither an outage nor an unevaluable feature establishes that the user is unentitled, and answering `0` there would tell an upgraded user they had not upgraded.
  - `reauth_required` → the transport-level `401`, exactly as below.

  The answer is deliberately not a check result: no `PLUGPASS_PLUGIN`, no `USE_AUTHORIZED`, nothing an access-handler skill triggers on and nothing downstream can read as a grant. The probe authorizes **nothing** — the call the user retries is checked on its own.
- **Handler, the check.** POST the input object unchanged to `{ENTITLEMENT_API_ORIGIN}/entitlement/check_premium_access` with `Authorization: Bearer {the request's bearer}` and a 5-second timeout per attempt — on a thrown request (network error / timeout) or a `5xx`, retry ONCE after a 2-second backoff; `4xx` responses are terminal, never retried. Response handling:
  - `200 { "status": "ok", "result_text": "…" }` → return `result_text` **verbatim** as the tool's text result. Never parse, reformat, or wrap it — it is byte-identical to what the platform's own check tool returns.
  - `200 { "status": "reauth_required" }`, or an HTTP `401` → the bearer is revoked or stale: respond at the **transport level** with the same `401` + `WWW-Authenticate` challenge as an invalid bearer (the template's mechanism), so the client re-authorizes and retries. Never return this as tool text.
  - Any other status, or the timeout → the unavailable grant (below).

## Step 5: Per-tool wrappers (`tool_wrappers`; added / changed / template refresh)

Write the wrapper for an `added` or `changed` tool, and for an `unchanged` tool whose `template_stale` is `true` — the latter replaces its existing wrapper with one rendered from the current template, its role unchanged. For every other `unchanged` tool, **verify only**: confirm the handler still carries the gate for its role (the `track_usage` / `check_remaining` call with its `feature_id`, or the `sub` read for an identity tool), its `_meta` id, and — on a paid tool — the denial's call echo and marker rule below; a tool missing any of them is treated as `added`. Never rewrite a verified wrapper — it is a contextual edit into the publisher's own code, and leaving working code alone is the point of the rule.

What a tool's handler does with the request context's `{ sub, bearer }` depends on its role:

- **Truly-free tool** (no per-user state) → untouched. It runs behind the connect gate like everything else, but reads neither `sub` nor the bearer.
- **Identity tool** — any tool whose body is per-user; the **paired-tool remove side** (`operation: remove`) is the canonical case (removing a record is always allowed, but the handler must know **whose** record to remove) → read `sub` from the request context and scope the body to it. **No Entitlement API call** — zero Plugpass round-trips.
- **Solo paid tool** → consume-on-invocation: call the Entitlement API **`track_usage`** at tool entry; run the tool body only after an `ok` response.
- **Paired-tool add side** (`operation: add`) → gate-the-add: because the publisher's own DB is the source of truth, gate with **`check_remaining` at tool entry**, read-only (it never consumes a Plugpass counter). Read the user's **current count** of that record from the publisher's store (scoped to `sub`) and pass it as `current_count` (see Reading the user's current count, below); the API authorizes a **limited/total** record iff `current_count < limit`, and an **unlimited** record on entitlement alone (no cap, so the count is ignored). Run the tool body only after an `ok` response; a non-authorized `result_text` rejects with no record added.

A database-record add **never consumes** — there is no `track_usage`, no shared-pool draw, and no after-the-fact op on a success branch — so the gate is wholly at tool entry, with no success handler to locate, whether the tool's own work is synchronous or asynchronous. (The shared unlimited pool is only a `/plans` budget-calc abstraction for an unlimited record, never a runtime touch.)

**Watch for `outputSchema`.** A wrapped tool's non-authorized responses are the **text-only** composed check result — they carry no structured payload. If the tool declares an MCP `outputSchema`, the SDK rejects those responses (it demands matching `structuredContent`), so the paywall never reaches the agent. When you wrap a tool that declares an `outputSchema`, **remove it** (its success path can still return `structuredContent`, which the SDK ignores without a schema) — a tool that can paywall can't guarantee structured output.

### Tool annotations

Every tool on the server must declare all four MCP `ToolAnnotations` hints — `readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint`. This is not optional polish:

- **The defaults are pessimistic.** An omitted hint falls back to `readOnlyHint: false`, `destructiveHint: true`, `idempotentHint: false`, `openWorldHint: true`. ChatGPT applies exactly those and renders an undeclared tool with a write/destructive badge and a confirmation prompt — so a read-only tool that simply forgets them looks dangerous to every end user.
- **OpenAI requires three of them** (`readOnlyHint`, `destructiveHint`, `openWorldHint`) on every tool descriptor submitted to the ChatGPT app directory. A server missing them can't be submitted.
- **State all four even when the spec says one is redundant.** The spec calls `destructiveHint` meaningless when `readOnlyHint` is true; ChatGPT does not honor that precedence. Never rely on an implied value.

So: **if a tool you touch declares no annotations, add them** — the tool's own behavior decides the values. And because wrapping changes behavior, **re-derive the hints of every tool you wrap**:

- **Solo paid tool** (`track_usage`) → the wrapper consumes a counter on every call, so the tool is no longer read-only and no longer idempotent, whatever it was before. Force `readOnlyHint: false` and `idempotentHint: false`. This is the common trap: a read-only "search"/"generate" tool honestly annotated `readOnlyHint: true` becomes a lie the moment it meters.
- **Paired-tool add side** (`check_remaining`) → the gate is read-only and consumes nothing, so the gate itself changes no hint. Annotate the tool for what its own body does (a record-adding tool is a non-destructive write; idempotent if the store dedupes).
- **Identity tool and truly-free tool** → no Entitlement API call, so nothing changes. Annotate for the tool's own behavior — and note the paired **remove** side deletes user data, which is the genuine `destructiveHint: true` case.

Judging the values, in the tool's own terms: `readOnlyHint` — does the handler modify any state? `destructiveHint` — can it irreversibly remove or damage something (deleting a record), as opposed to an additive or metering write? `idempotentHint` — does a second identical call leave the state the first one produced? `openWorldHint` — does it reach an open-ended external world (the public internet, a third-party API), or a bounded domain (the publisher's own store and services, the caller's own records)?

**Judge what the handler does, not how the description reads.** Descriptions carry marketing flavor that hints must not inherit — a tool described as drawing on "external inspiration" but implemented as a call to the plugin's own AI is a **closed** world (`openWorldHint: false`); a plugin's own backend, however remote, is still its own bounded domain. Words like *external*, *web*, or *live* in the copy are not evidence. Read the handler, then annotate. The one place the description does bind is the entitlement call the wrapper adds — that's real behavior, and it's why metering forces `readOnlyHint`/`idempotentHint` to false above.

Ensure each paid or identity tool's `_meta` field carries its `plugpass_component_id` (`{plugpass_id}`) at registration (requires MCP protocol revision 2025-06-18+) — the re-run identity `plugpass_sync_plugin` / future runs match on. `plugpass-sync-plugin` already stamps it when the tool is first registered, so normally you're **reconciling**: leave it if it already matches, write it only if absent or stale, and keep it intact when you wrap the handler.

### The Entitlement API call

For each **paid** tool call, forward the request's bearer as `Authorization: Bearer …` and POST the resolved component descriptor to `{ENTITLEMENT_API_ORIGIN}/entitlement/{check_remaining|track_usage}` with a 5-second timeout per attempt — on a thrown request (network error / timeout) or a `5xx`, retry ONCE after a 2-second backoff; `4xx` responses are terminal, never retried. The body is `{ plugin_id, feature_id, current_count? }` (no `feature_type` — the `feature_id` prefix carries it; the server derives the user from the bearer's `sub`):

- Solo tool → `feature_id: "{tool plugpass-component-id}"` (the `tool_`-prefixed id baked into the tool's `_meta.plugpass_component_id`); `track_usage`.
- Paired-tool add side → `feature_id: "{custom_entitlements plugpass-id}"` (a `custom_`-prefixed id) — read it from the add tool's `database_record.custom_entitlement_id` in the `plugpass_get_plugin_data` response (the shared paired-tool entitlement's plugpass-id; **not** `database_record.plugpass_id`, which is the database-record id and resolves to no entitlement). Always `check_remaining`, passing the user's `current_count`. The remove side makes no Entitlement API call at all (identity only).

Response handling (the `status`-discriminated JSON — see the Wire contract):

- `ok` → authorized; run the tool body.
- `non_authorized` → return the response's `result_text` **verbatim** as the tool's text result — a single-field pipe, never parsed, reformatted, or re-serialized (the server composed the complete block, and per-key re-serialization can't carry it faithfully). The trigger keys inside it auto-fire the plugin's access-handler skill via its description, which then owns the flow. The result also carries three things for the in-widget paywall (the language template's denial renderer takes all three), inert everywhere else:
  - **The denied-call echo** — `_meta.plugpass_denied_call`, `{ name, arguments, widget_callable }`: the tool's name, the arguments this call was made with, and whether a widget may call the tool at all (its registered `_meta.ui.visibility`; undeclared means the model and a widget both may). Hosts pass a result's `_meta` to an MCP Apps widget and never show the model; the paywall replays the call from it once the user has upgraded — or, for a tool a widget may not call, hands the retry to the conversation as a user turn.
  - **The status probe** — `_meta.plugpass_status_probe`, `{ name, arguments }`: this server's `CHECK_TOOL_NAME` with the arguments already composed (`{ plugin_id, feature_id, status_code: true }`), so the widget's script supplies nothing of its own. Present **only** when the denial cannot be replayed — a UI-backed tool a widget may not call — and only when this server hosts a check tool; every other denial carries none, since a replay answers the same question by actually running the call. It is how the paywall confirms an upgrade for a call it cannot make itself.
  - **The paywall-UI marker** — a second text block, exactly `PLUGPASS_PAYWALL_UI=true`, when the tool is **UI-backed** (its registration declares `_meta.ui.resourceUri`) and the client renders widgets: the widget's paywall is then the one asking the user, and the access-handler skill posts nothing beside it. The client renders widgets when it declares the `io.modelcontextprotocol/ui` extension among its client capabilities — on 2026-07-28 in every request's `_meta["io.modelcontextprotocol/clientCapabilities"]` envelope, on a 2025-era connection at the handshake. Each language template reads it through whatever its SDK exposes for that (an envelope accessor, a typed capabilities reader, or both where the SDK parses only one source), so the rule is one rule whatever the era. The template's denial renderer reads all of it off the tool's own registered `_meta` (its widget, who may call it) and the request at runtime — the wrapper bakes no per-tool constant, so a tool that gains or loses its widget, or changes who may call it, needs no rewrite, and a tool that renders no widget of its own never carries the marker, whatever the client.
- `reauth_required` (the JSON envelope, or an HTTP `401` from the API) → respond at the **transport level** with the `401` + `WWW-Authenticate` challenge (the same mechanism as the proxy tool), so the client re-authorizes and retries.
- Any other non-200, or a thrown request — each after the single retry above → the unavailable grant (below).

**The unavailable grant.** A failing or unreachable Entitlement API **grants**: this call is server-to-server from the server's own host, so a Plugpass failure is never the end user's doing and never costs them access. Nothing was consumed. Never deny on it, and never surface it as an error.

- A wrapped tool runs its body, exactly as on `ok`.
- The check proxy returns this exact text as its tool result (the templates' `<the unavailable grant text from TOOLS.md>` placeholder resolves to it):

  > USE_AUTHORIZED=true

## The wire contract

`POST {ENTITLEMENT_API_ORIGIN}/entitlement/{op}` — request `{ plugin_id: string, feature_id: string, current_count?: number }` (the `feature_id` prefix carries the feature type — no separate field), response:

```
{ "status": "ok", "remaining": number | null }
| { "status": "reauth_required" }
| { "status": "non_authorized", "result_text": string }
```

`result_text` is the complete server-composed check result — the same text block the plugin's native check tool returns — which the wrapper emits verbatim as its tool result.

`POST {ENTITLEMENT_API_ORIGIN}/entitlement/check_premium_access` (the proxy's upstream) — request `{ plugin_id: string, feature_id: string, plugin_version?: string }` (the `feature_id` prefix carries the `skill` type; an omitted `plugin_version` is absent from the body, never sent empty), response `{ "status": "ok", "result_text": string } | { "status": "reauth_required" }`. This endpoint **consumes**; the proxy's `status_code` probe therefore uses `check_remaining` instead, which does not.

Non-cascade statuses on either endpoint: `401` missing/invalid bearer (treat as reauth); `400`, `403`, `5xx` and every other non-200 (the unavailable grant).

### Reading the user's current count

A limited (total) record authorizes iff `current_count < limit`, so the add wrapper must read the user's current count of that record — scoped to `sub` — and pass it as `current_count`. **Always read it** — an unlimited plan simply ignores the value, so a later total↔unlimited plan change needs no wrapper change. That count lives in the **publisher's own store**, not ours, so determining how to read it is part of the implementation — approached like any change to a codebase you don't already understand:

- **Work it out yourself first.** Read the add/remove tools' source to find where the record is stored and how it's scoped to the user, and write the count read against that store. Never ask the publisher for something you can find yourself.
- **Confirm what counts.** The count must match exactly what the limit governs — confirm the definition with the publisher whenever it's ambiguous (active vs. archived / soft-deleted / expired rows, per-user scoping, and the like). Getting it wrong silently mis-enforces the paywall, with no error to surface it, so confirm rather than guess.
- **Research / safe experiment.** Web-research the publisher's storage layer or ORM when it helps, and you may ask permission to run a safe experiment (e.g. "may I add then immediately remove a test record so I can see how it's stored and counted?").
- **Bound the effort** — a genuine, reasonable search, then stop.

If you can't confidently determine how to read the count, don't guess — handle it per **Don't guess, don't leave markers** below.

### Don't guess, don't leave markers

Never edit a repository other than the plugin repo and the located MCP server source without the publisher's explicit permission and a plain-English explanation of why; any approved outside repo is recorded for `edited_repos` in the final report.

**When a tool or server can't be brought to its correct end state** (e.g. its source can't be confidently located, or you can't confidently determine how to read the user's current count): do **not** guess, and do **not** leave a marker or `TODO` in the publisher's source — a comment only a human reader would act on is a silent failure. Instead, explain in plain English what remains and why, leave the relevant task(s) **open**, and exclude the affected component(s) from the implementation record — the dashboard's publish gate then holds until a re-run verifies the wiring. The explanation is plain, dense, assumes no knowledge of the entitlement model, and never uses internal terms (no "paired tools").

## Step 6: The in-widget paywall (`ui_paywall`, servers that render widgets)

A tool call a widget makes itself returns its result to the widget, never to the model, so a non-authorized `result_text` there fires no access-handler skill. The platform's paywall script renders it in place instead: loaded into every widget the server renders, it watches for a denial, shows the composed message with its call to action, and replays the denied call once the user confirms they've upgraded. Write the layer — from the language template — when `server_scaffolding_template_stale` is `true` or the layer is missing; otherwise **verify** it (the constant is baked with the current `paywall_script_url`, every UI resource read passes through the helper) and leave it alone. Two pieces:

- **`PAYWALL_SCRIPT_URL`**, baked beside the other runtime constants from `paywall_script_url`.
- **The injection helper**, applied to every resource the server reads out whose MIME type is `text/html;profile=mcp-app` (every resource a tool's `_meta.ui.resourceUri` names): it inserts `<script src="{PAYWALL_SCRIPT_URL}"></script>` as the first child of the document's `<head>`, ahead of the widget's own scripts (prepended to the document when it has no `<head>`), and adds the script's origin to the resource's `_meta.ui.csp.resourceDomains` (the sandbox's allowlist for scripts — deduplicated, the widget's own origins kept). The read passes through the helper on its way out; the publisher's widget HTML itself is never edited.

The three wrapper-side pieces the paywall reads — the denied-call echo, the status probe, and the paywall-UI marker — are part of every paid wrapper (Step 5), on every server, whether or not the server renders widgets.

## Step 7: Removals

For each `removed` tool, restore its plain handler: strip the entitlement gate (the `track_usage` / `check_remaining` call and its branching); keep the `sub` read only if the tool's body genuinely still needs per-user scoping (a de-gated per-user tool is now an identity tool, not a plain one); leave its `_meta` id (identity is harmless and cheap to keep). Do **not** tear down the resource-server layer or the proxy tool on removals — the server remains the plugin's connector (or a still-registered owned server) and stale bundles may still call it; leftover scaffolding is inert and harmless. The only proxy removal this skill ever performs is the connector directive's publisher→publisher case (the previous check host, when its repo is in the run). Already-stripped → no-op.

## Reconcile runs

On a `--reconcile` run, additionally re-audit what normal runs only check for presence — the audit is **probe-based** (re-read the scaffolded code and re-run the local verification below), not server-side state:

- Each in-set server's **resource-server layer** (when its directive names `resource_server`): present, constants baked with the current response values, challenge + PRM + validation intact.
- The check host's **proxy tool**: registered with the exact name/description/schema, piping to the current `ENTITLEMENT_API_ORIGIN`.
- Every **`unchanged`** tool's wrapper: the gate call, descriptor (`feature_id` — its prefix carries the type), `current_count` read, `_meta` id, and the denial's call echo + marker rule present and correct.
- Each in-set server's **in-widget paywall** (when its directive names `ui_paywall`): `PAYWALL_SCRIPT_URL` baked with the current `paywall_script_url`, and every `text/html;profile=mcp-app` resource read passing through the injection helper.

Regenerate anything missing or drifted. This is the explicit escape hatch for MCP-server source that drifted out of band (hand-edits, a git revert, a previously-failed run).

## Step 8: Per-server verification

Before the run reports a server's work done, verify it locally — drive it yourself when the repo gives you a runnable dev command; otherwise ask the publisher to start the server and tell you the local port:

1. The code **compiles / imports cleanly**, and the server **starts and serves** (probes 2–4 exercise it). Registration itself is a source-level read: confirm every tool this run touched — plus the check proxy tool, on the check host — has its registration call present and reached from the server's setup path. A compile check never establishes registration, and runtime registration is only observable via `tools/list`, which carries the same bearer requirement as item 7. On a server that builds its MCP instance per request, the registration code doesn't even execute until an authenticated call arrives, so a clean start proves nothing about it either.
2. When the server's directive names `resource_server` (probes 2–4): `GET /.well-known/oauth-protected-resource/mcp` serves the PRM document with `resource` = the server's `RESOURCE_URL` and `authorization_servers` = `[ISSUER]`.
3. A bare request to `/mcp` (no `Authorization` header) returns `401` with the `WWW-Authenticate` challenge carrying `error="invalid_token"` and a `resource_metadata` parameter.

   **Assert the challenge's shape, not its host, when probing locally.** Some dev servers proxy the response and rewrite the host in outgoing headers to the local listen address, so `realm` and `resource_metadata` can come back as `127.0.0.1:{port}` instead of the baked `RESOURCE_URL` — an artifact of the dev server, not drift in the written source. Item 2 is the authoritative check on that value: it reads a response *body*, which is not rewritten. To confirm directly, re-send this request with `Host:` set to the `RESOURCE_URL`'s host — the baked value then passes through untouched. A local scheme mismatch (`https://` in the header over a plain-HTTP local request) is the same artifact and is likewise not a finding.
4. A garbage bearer (`Authorization: Bearer not-a-jwt`) is rejected the same way: `401`, `error="invalid_token"`, and a `resource_metadata` parameter — the same three assertions as item 3, and subject to the same local-host caveat. `error_description` is **expected to differ** between the two probes, since it carries the reason; a different description is not a finding. What this probe checks is that a malformed bearer takes the invalid-token path rather than some other error code — as every invalid-bearer case must (missing, malformed, bad signature, wrong issuer or audience, expired).
5. **Cross-check the baked values the probes can't see** — read them back from the written source against this run's data: every wrapper's entitlement descriptor (`feature_id` — its prefix carries the type; a paired add must carry the shared `custom_entitlements` `custom_`-prefixed id from `database_record.custom_entitlement_id`, never the database-record id and never the tool's own `_meta` id), its **operation against its role** (`track_usage` for a solo paid tool, `check_remaining` for a paired add — a solo tool wrongly calling `check_remaining` authorizes without ever consuming, a silent under-metering nothing downstream catches), its denial's call echo (the tool's own name and its actual arguments), its status probe (named only on an unreplayable denial, and dispatching to `check_remaining` — a probe wired to `check_premium_access` would consume the user's allowance every time the paywall asked) and marker rule (the renderer given the tool's own registered `_meta` and the request — never a baked constant), and the baked `ENTITLEMENT_API_ORIGIN` default (proxy + wrappers) against the response's `entitlement_api_origin`. (The PRM probe above already pins `RESOURCE_URL` and `ISSUER` on the wire.)
   When the directive names `ui_paywall`, read back the layer the same way — `resources/read` carries the same bearer requirement as `tools/list`, so this is source-level too: the baked `PAYWALL_SCRIPT_URL` equals the response's `paywall_script_url`, and every resource read whose MIME type is `text/html;profile=mcp-app` passes through the injection helper (the script tag first in `<head>`, the origin in `resourceDomains`).
6. **The server answers both protocol eras** (every language but the JVM, which has no SDK support — see Step 3). Read it back from the written source: the composition serves the modern leg (`server/discover` and the per-request `_meta` envelope) alongside the 2025-era leg, and **both** buffer their responses so the reauth swap works on either. A `/mcp` request carrying `MCP-Protocol-Version: 2026-07-28` must not be refused as unknown. Source-level for the same reason as item 5: the endpoint is bearer-gated, so no local probe can call it.
7. **Every tool declares all four annotations in its registration** — `readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint` — and each solo paid tool's read the consuming shape (`readOnlyHint: false`, `idempotentHint: false`). Read this back from the written source, tool by tool, the same way item 5 reads back the baked values. A missing hint is invisible on this server and only shows up as a wrong destructive/confirm badge in the end user's client, so it is worth asserting even though the assertion is source-level. See Tool annotations.

   **Do not try to read this off the `tools/list` wire.** The resource-server layer gates every `/mcp` request, so `tools/list` needs a valid bearer — which no local probe has, and which deploying does not provide either (a deployed server is gated identically). The residual this leaves: an annotation block that is declared in source but never reaches the descriptor — a mis-shaped field, or an SDK/framework that drops it silently — is not caught here. That case belongs to the pre-publish connect loop below, which is the only surface with a real token.

The full bearer round-trip (mint → call → cascade) needs a real Plugpass-minted token and is covered by the pre-publish connect loop (the publisher connects a real client against the deployed server) — that loop also owns the wire-level annotation read described in item 7. These local probes are what this run owes.

## Outcome

For each server and tool brought to or confirmed in its correct end state (scaffolded, wrapped, identity-only, stripped, or verified), mark its task completed and include the tool components in the implementation record. Record each edited server-source repo for `edited_repos` (role `mcp_server`, with `mcp_server_names`). Any item that couldn't be completed — e.g. an unlocatable server source — keeps its task open, is excluded from the record, and is explained plainly in the closing summary.
