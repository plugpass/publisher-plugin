# Python scaffolding template

The publisher's server becomes an **OAuth-protected resource server** using the official MCP SDK's built-in auth: a `TokenVerifier` + `AuthSettings` make FastMCP serve the RFC 9728 PRM document itself and answer `/mcp` requests without a valid bearer with the `401` + `WWW-Authenticate` challenge (`error="invalid_token"`, `resource_metadata="…"` — the exact shape MCP clients key OAuth discovery off). Standalone source, no platform package. Version pins: **`mcp>=1.28.1,<2`** (v2 is imminent and breaks FastMCP/auth surfaces — the `<2` pin is load-bearing), `pyjwt[crypto]>=2.13.0`, `httpx`.

**Two structural decisions carry the whole design — never undo them:**

1. **`stateless_http=True, json_response=True` on FastMCP, always.** In JSON mode the transport writes the HTTP response only after the tool handler finished, so an outer ASGI middleware can replace it with a `401` when a tool discovers mid-call that the bearer is revoked (`reauth_required` from the Entitlement API). In SSE mode the `200` headers hit the wire before the tool runs — the swap is impossible. Stateless also makes per-request auth context reliable.
2. **The audience is the baked `RESOURCE_URL`, never the request host** — it is the JWT `aud` pin, the `AuthSettings.resource_server_url` (which becomes both the PRM route path and the PRM `resource`), and the challenge's `resource_metadata` base. This is what lets a locally-listening server accept real bearers minted for its public URL.

```python
# premium_feature_access_check.py — written once per server. Standalone.
import os
from dataclasses import dataclass

import anyio
import httpx
import jwt as pyjwt
from jwt import PyJWKClient
from mcp.server.auth.provider import AccessToken

MCP_PATH = "/mcp"

# Plugpass endpoints for this server.
ISSUER = "<plugpass_issuer>"
JWKS_URL = "<plugpass_jwks_url>"
ENTITLEMENT_API_ORIGIN = "<entitlement_api_origin>"
# This server's own public MCP URL — the JWT `aud` pin and the PRM `resource`.
RESOURCE_URL = "<this server's RESOURCE_URL>"


@dataclass(frozen=True)
class PlugpassConfig:
    issuer: str
    jwks_url: str
    entitlement_api_origin: str
    resource_url: str

    @property
    def prm_url(self) -> str:
        # RFC 9728 path-aware form: origin + /.well-known/oauth-protected-resource + /mcp
        origin = self.resource_url.removesuffix(MCP_PATH)
        return f"{origin}/.well-known/oauth-protected-resource{MCP_PATH}"


def plugpass_config() -> PlugpassConfig:
    return PlugpassConfig(
        issuer=ISSUER,
        jwks_url=JWKS_URL,
        entitlement_api_origin=ENTITLEMENT_API_ORIGIN,
        resource_url=RESOURCE_URL,
    )


class PlugpassTokenVerifier:
    """mcp.server.auth.provider.TokenVerifier — local JWKS validation, no
    Plugpass round-trip. EdDSA-pinned, issuer exact, audience = the baked
    RESOURCE_URL, exp required. PyJWKClient caches the key set (lifespan=4h)
    and refetches once on an unknown kid (key rotation)."""

    def __init__(self, cfg: PlugpassConfig) -> None:
        self._cfg = cfg
        self._client = PyJWKClient(cfg.jwks_url, lifespan=14_400)

    def _decode(self, token: str) -> dict:
        signing_key = self._client.get_signing_key_from_jwt(token)
        return pyjwt.decode(
            token,
            signing_key.key,
            algorithms=["EdDSA"],
            audience=self._cfg.resource_url,
            issuer=self._cfg.issuer,
            options={"require": ["exp", "aud", "iss"]},
        )

    async def verify_token(self, token: str) -> AccessToken | None:
        try:
            # PyJWKClient's fetch is blocking urllib — keep it off the event loop.
            payload = await anyio.to_thread.run_sync(self._decode, token)
        except Exception:
            return None  # any failure → the SDK's 401 challenge (fail closed)
        sub = payload.get("sub")
        if not isinstance(sub, str) or not sub:
            return None
        return AccessToken(
            token=token, client_id=sub, scopes=[],
            expires_at=payload.get("exp"), subject=sub, claims=payload,
        )


# ---------------------------------------------------------------------------
# Entitlement API client (paid tools + the check proxy).
# ---------------------------------------------------------------------------

# Result: {"status": "ok", "remaining": int|None}
#       | {"status": "reauth_required"}
#       | {"status": "non_authorized", "result_text": str}
#       | {"status": "unavailable"}   ← Plugpass could not answer; the caller GRANTS
# 5s timeout PER ATTEMPT, with ONE retry after a 2s backoff on a transport
# error or a 5xx; 4xx are terminal and never retried. Every failure past that
# maps to "unavailable": the call is server-to-server from this host, so the end
# user cannot have caused it.
async def entitlement(bearer: str, body: dict, op: str, cfg: PlugpassConfig) -> dict:
    for attempt in range(2):
        try:
            async with httpx.AsyncClient(timeout=5.0) as client:
                res = await client.post(
                    f"{cfg.entitlement_api_origin}/entitlement/{op}",
                    headers={"authorization": f"Bearer {bearer}"},
                    json=body,
                )
            if res.status_code >= 500 and attempt == 0:
                raise httpx.HTTPStatusError(
                    f"entitlement {op}: {res.status_code}", request=res.request, response=res
                )
        except (httpx.TransportError, httpx.HTTPStatusError):
            if attempt == 0:
                await asyncio.sleep(2)
                continue
            return {"status": "unavailable"}
        except Exception:
            return {"status": "unavailable"}
        if res.status_code == 401:
            return {"status": "reauth_required"}
        if res.status_code != 200:
            return {"status": "unavailable"}
        try:
            return res.json()
        except Exception:
            return {"status": "unavailable"}
    return {"status": "unavailable"}


def non_authorized_text(result: dict) -> str:
    """The server-composed check result verbatim — a single-field pipe (the
    trigger keys inside it auto-fire the plugin's access-handler skill).
    Never parse, reformat, or re-serialize it."""
    return str(result["result_text"])


# The check proxy's unavailable grant — Plugpass could not answer, so the check
# grants and the paid skill runs. A wrapped tool needs no equivalent: it just
# runs its body.
UNAVAILABLE_CHECK_GRANT = "<the unavailable grant text from TOOLS.md>"


class ReauthTo401Middleware:
    """Pure ASGI. When a tool set `plugpass_reauth_required` in the request
    scope state (the Entitlement API reported the bearer revoked), replace the
    whole response with the 401 challenge — same shape as the initial one, so
    the client re-authorizes and retries. Requires json_response=True."""

    def __init__(self, app, resource_metadata_url: str) -> None:
        self.app, self.url = app, resource_metadata_url

    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            return await self.app(scope, receive, send)
        replaced = False

        async def wrapped_send(message):
            nonlocal replaced
            if message["type"] == "http.response.start" and scope.get("state", {}).get(
                "plugpass_reauth_required"
            ):
                replaced = True
                body = b'{"error": "invalid_token", "error_description": "Access token no longer valid"}'
                await send({"type": "http.response.start", "status": 401, "headers": [
                    (b"content-type", b"application/json"),
                    (b"content-length", str(len(body)).encode()),
                    (b"www-authenticate",
                     f'Bearer error="invalid_token", error_description="Access token no longer valid", resource_metadata="{self.url}"'.encode()),
                ]})
                await send({"type": "http.response.body", "body": body})
                return
            if replaced:
                return  # discard the inner app's response
            await send(message)

        await self.app(scope, receive, wrapped_send)
```

**Server entry** (`server.py`) — the SDK serves the PRM document and the challenge itself; the only custom ASGI piece is the reauth middleware:

```python
import uvicorn
from mcp.server.fastmcp import FastMCP

cfg = plugpass_config()
mcp = FastMCP(
    "<server name>",
    host="0.0.0.0",
    port=int(os.environ.get("PORT", "8000")),
    stateless_http=True, json_response=True,   # required: lets a revoked-bearer tool swap in a 401
    token_verifier=PlugpassTokenVerifier(cfg),
    auth=AuthSettings(
        issuer_url=cfg.issuer,                 # → PRM authorization_servers
        resource_server_url=cfg.resource_url,  # MUST include /mcp — drives the PRM path AND its `resource`
        required_scopes=None,
    ),
)

# ...tool registrations (below)...

app = mcp.streamable_http_app()  # /mcp (401-gated) + /.well-known/oauth-protected-resource/mcp
app.add_middleware(ReauthTo401Middleware, resource_metadata_url=cfg.prm_url)
uvicorn.run(app, host=mcp.settings.host, port=mcp.settings.port)
# NOT mcp.run(...) — it rebuilds the app internally and would drop the middleware.
```

**Per-request identity inside a tool**: declare `ctx: Context` and read the validated principal off the request — `request = ctx.request_context.request`; `sub = request.user.access_token.subject`; the raw bearer to forward is `request.user.access_token.token`. On reauth: `request.state.plugpass_reauth_required = True` (the scope-state flag the middleware reads — reliable in both session modes, unlike contextvars).

**The check proxy tool (check host only).** A pure pipe — never parse or reformat `result_text`:

```python
from mcp import types
from mcp.server.fastmcp import Context

CHECK_PREMIUM_ACCESS_DESCRIPTION = "<the VERBATIM description from TOOLS.md § The check proxy tool>"

@mcp.tool(name="{CheckToolName}", title="Check premium access",
          description=CHECK_PREMIUM_ACCESS_DESCRIPTION,
          annotations=types.ToolAnnotations(
              readOnlyHint=False, destructiveHint=False,
              idempotentHint=False, openWorldHint=False),
          structured_output=False)
async def check_premium_access(
    plugin_id: str, feature_id: str, plugin_version: str, ctx: Context
) -> str:
    request = ctx.request_context.request
    bearer = request.user.access_token.token
    result = await entitlement(
        bearer,
        {"plugin_id": plugin_id,
         "feature_id": feature_id, "plugin_version": plugin_version},
        "check_premium_access", cfg,
    )
    if result["status"] == "unavailable":
        return UNAVAILABLE_CHECK_GRANT
    if result["status"] == "reauth_required":
        request.state.plugpass_reauth_required = True   # → transport-level 401 via the middleware
        return "Re-authentication required."            # discarded by the swap
    return result["result_text"]  # verbatim — byte-identical to the native tool
```

**Solo paid tool wrapper** (consume-on-invocation; no `auth_token` parameter on this path):

```python
@mcp.tool(name="paid_tool", description="...", structured_output=False,
          # Metering makes this tool neither read-only nor idempotent, whatever it
          # was before the wrap. The other two hints keep the tool's own values.
          annotations=types.ToolAnnotations(
              readOnlyHint=False, destructiveHint=<the tool's own value>,
              idempotentHint=False, openWorldHint=<the tool's own value>),
          meta={"plugpass_component_id": "<plugpass_id>"})
async def paid_tool(..., ctx: Context) -> str:
    request = ctx.request_context.request
    sub = request.user.access_token.subject
    result = await entitlement(
        request.user.access_token.token,
        {"plugin_id": "<plugin-plugpass-id>", "feature_id": "<plugpass_id>"},
        "track_usage", cfg,
    )
    if result["status"] == "reauth_required":
        request.state.plugpass_reauth_required = True
        return "Re-authentication required."
    if result["status"] == "non_authorized":
        return non_authorized_text(result)
    # ...existing tool body — keyed / scoped to `sub`; "unavailable" consumed
    # nothing and grants...
```

**Paired-tool add side** (`operation: add`): same shape but `feature_id: "<custom-entitlement-plugpass-id>"` (the add tool's `database_record.custom_entitlement_id` — its `custom_` prefix carries the feature type; **not** `database_record.plugpass_id`, and **not** the tool's own `_meta` id), always **`check_remaining`**, passing the user's current count from the publisher's own store (scoped to `sub`) as `current_count` — see TOOLS.md → Reading the user's current count.

**Identity tool** (the paired remove side, or any per-user free tool): no Entitlement API call — read `ctx.request_context.request.user.access_token.subject` and scope the body to it.

**Placement guidance.** A FastMCP server keeps its existing tool modules; the verifier + middleware live in `premium_feature_access_check.py`; the entry file gains the `FastMCP(...)` auth kwargs + the two-line app assembly. When the publisher mounts the MCP app inside a larger Starlette/FastAPI app instead, `Mount("/…", mcp.streamable_http_app())` works but the **parent** app's lifespan must run `mcp.session_manager.run()`, and the reauth middleware wraps the parent. Invariants regardless of layout: `stateless_http=True, json_response=True`; `resource_server_url` includes `/mcp`; wrapped tools keep `meta={"plugpass_component_id": …}`, declare `structured_output=False`, and never declare an output schema.
