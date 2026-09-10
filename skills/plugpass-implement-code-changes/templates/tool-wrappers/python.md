# Python scaffolding template

The publisher's server becomes an **OAuth-protected resource server** using the official MCP SDK's built-in auth: a `TokenVerifier` + `AuthSettings` make the server serve the RFC 9728 PRM document itself and answer `/mcp` requests without a valid bearer with the `401` + `WWW-Authenticate` challenge (`error="invalid_token"`, `resource_metadata="…"` — the exact shape MCP clients key OAuth discovery off). Standalone source, no platform package. Version pins: **`mcp>=2.2.0,<3`** (the 2.x line — it is what implements protocol 2026-07-28; on 1.x the server serves the legacy era only), `pyjwt[crypto]>=2.13.0`, `httpx`.

**Three structural decisions carry the whole design — never undo them:**

1. **The server serves BOTH protocol eras from the one app.** The SDK's session manager routes each request by its `MCP-Protocol-Version` header: 2026-07-28 to the modern, envelope-based leg (which owns `server/discover`), the handshake versions to their own. The client's UI capability rides the modern envelope — that is what the paywall-UI marker reads — and a host that gets only the legacy era never sends it. Nothing extra is wired for it: the SDK line does the era routing.
2. **`stateless_http=True, json_response=True` on the app, always.** In JSON mode BOTH legs write the HTTP response only after the tool handler finished, so an outer ASGI middleware can replace it with a `401` when a tool discovers mid-call that the bearer is revoked (`reauth_required` from the Entitlement API). In SSE mode the `200` headers hit the wire before the tool runs — the swap is impossible. Stateless also makes per-request auth context reliable.
3. **The audience is the baked `RESOURCE_URL`, never the request host** — it is the JWT `aud` pin, the `AuthSettings.resource_server_url` (which becomes both the PRM route path and the PRM `resource`), and the challenge's `resource_metadata` base. This is what lets a locally-listening server accept real bearers minted for its public URL.

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
PLUGIN_ID = "<the plugin's Plugpass id>"
ISSUER = "<plugpass_issuer>"
JWKS_URL = "<plugpass_jwks_url>"
ENTITLEMENT_API_ORIGIN = "<entitlement_api_origin>"
# This server's own public MCP URL — the JWT `aud` pin and the PRM `resource`.
RESOURCE_URL = "<this server's RESOURCE_URL>"
# Where the Plugpass paywall for MCP Apps widgets loads from (the `ui_paywall`
# layer — only on a server whose directive names it).
PAYWALL_SCRIPT_URL = "<paywall_script_url>"
# This server's own check tool, named on a denial so the paywall can ask it
# whether the user became entitled. None on a server that hosts no check tool
# (the check proxy is registered on the plugin's check host only), in which case
# a denial carries no probe and the paywall says less, never something untrue.
CHECK_TOOL_NAME: str | None = "<check_tool_name, or None off the check host>"


@dataclass(frozen=True)
class PlugpassConfig:
    plugin_id: str
    issuer: str
    jwks_url: str
    entitlement_api_origin: str
    resource_url: str
    paywall_script_url: str
    check_tool_name: str | None

    @property
    def prm_url(self) -> str:
        # RFC 9728 path-aware form: origin + /.well-known/oauth-protected-resource + /mcp
        origin = self.resource_url.removesuffix(MCP_PATH)
        return f"{origin}/.well-known/oauth-protected-resource{MCP_PATH}"


def plugpass_config() -> PlugpassConfig:
    return PlugpassConfig(
        plugin_id=PLUGIN_ID,
        issuer=ISSUER,
        jwks_url=JWKS_URL,
        entitlement_api_origin=ENTITLEMENT_API_ORIGIN,
        resource_url=RESOURCE_URL,
        paywall_script_url=PAYWALL_SCRIPT_URL,
        check_tool_name=CHECK_TOOL_NAME,
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


# The marker a UI-backed tool's denial carries when the client renders widgets.
PAYWALL_UI_MARKER = "PLUGPASS_PAYWALL_UI=true"


def non_authorized_tool_response(
    result: dict,
    call: dict[str, Any],
    tool_meta: dict[str, Any],
    ctx: Context,
    cfg: PlugpassConfig,
    feature_id: str,
) -> types.CallToolResult:
    """non_authorized → result_text verbatim as the tool's text. Three things
    ride beside it for the Plugpass paywall an MCP Apps widget shows (all inert
    everywhere else): the denial NAMES the call it denied (`call`: the tool's
    name and the arguments it was called with) on the result's `_meta` — which
    hosts pass to a widget and never show the model — so the paywall can replay
    it once the user has upgraded, and says whether a widget may call the tool
    at all (widget_callable below; a host refuses a widget's call to a
    model-only tool, so the paywall then hands the retry to the conversation);
    the read-only status probe (status_probe below), named only when that replay
    is impossible; and the marker (a UI-backed tool AND a client that renders
    widgets, paywall_ui below) — a second text block carrying
    PLUGPASS_PAYWALL_UI=true, so the widget's paywall is the one asking the user
    and the access-handler skill posts nothing beside it. All read the tool's own
    registered meta and the request, at runtime."""
    content: list[types.ContentBlock] = [
        types.TextContent(type="text", text=non_authorized_text(result))
    ]
    if paywall_ui(tool_meta, ctx):
        content.append(types.TextContent(type="text", text=PAYWALL_UI_MARKER))
    return types.CallToolResult(
        content=content,
        _meta={
            "plugpass_denied_call": {**call, "widget_callable": widget_callable(tool_meta)},
            **status_probe(cfg, tool_meta, feature_id),
        },
    )


def status_probe(cfg: PlugpassConfig, tool_meta: dict[str, Any], feature_id: str) -> dict[str, Any]:
    """The read-only entitlement probe the paywall may call when it cannot replay
    a denied call: this server's check tool, with the arguments already composed
    so the widget's script supplies nothing of its own. Named ONLY when the call
    is unreplayable (a model-only UI-backed tool) and this server hosts a check
    tool; every other denial carries none, since a replay answers the same
    question by actually running the call."""
    ui = tool_meta.get("ui")
    if cfg.check_tool_name is None or not isinstance(ui, dict) or widget_callable(tool_meta):
        return {}
    return {
        "plugpass_status_probe": {
            "name": cfg.check_tool_name,
            "arguments": {
                "plugin_id": cfg.plugin_id,
                "feature_id": feature_id,
                "status_code": True,
            },
        }
    }


# Whether the calling client renders MCP Apps widgets: it declared the UI
# extension among its client capabilities. Two sources, because this server
# answers both protocol eras: on 2026-07-28 the capabilities ride every request's
# `_meta` envelope and the SDK parses them onto `ctx.client_capabilities`; a
# 2025-era client declares them at the initialize handshake, which reaches the
# same property, or repeats the key in a bare per-request `_meta`, which the SDK
# leaves unparsed. Read both, so the rule is one rule whatever the era.
CLIENT_CAPABILITIES_META_KEY = "io.modelcontextprotocol/clientCapabilities"
UI_EXTENSION_ID = "io.modelcontextprotocol/ui"


def _declares_ui_extension(capabilities: Any) -> bool:
    extensions = capabilities.get("extensions") if isinstance(capabilities, dict) else None
    return isinstance(extensions, dict) and UI_EXTENSION_ID in extensions


def client_renders_widgets(ctx: Context) -> bool:
    parsed = ctx.client_capabilities
    if parsed is not None and isinstance(parsed.extensions, dict):
        if UI_EXTENSION_ID in parsed.extensions:
            return True
    meta = ctx.request_context.meta  # a RequestParamsMeta TypedDict, or None
    return _declares_ui_extension(
        meta.get(CLIENT_CAPABILITIES_META_KEY) if meta is not None else None
    )


# Whether the widget's paywall is the one asking on this denial: the tool renders
# a widget (its own registration's meta declares `ui.resourceUri`) AND the client
# renders widgets. Read off the registration at runtime, so a tool that gains or
# loses its widget changes nothing here.
def paywall_ui(tool_meta: dict[str, Any], ctx: Context) -> bool:
    ui = tool_meta.get("ui")
    return (
        isinstance(ui, dict)
        and isinstance(ui.get("resourceUri"), str)
        and client_renders_widgets(ctx)
    )


# Who may call the tool, off the same registration: an undeclared visibility
# means the model and a widget both may; a declared list means exactly its
# members. A widget's paywall replays a denied call itself only when it may.
def widget_callable(tool_meta: dict[str, Any]) -> bool:
    ui = tool_meta.get("ui")
    visibility = ui.get("visibility") if isinstance(ui, dict) else None
    return not isinstance(visibility, list) or "app" in visibility


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
from mcp.server.mcpserver import MCPServer

cfg = plugpass_config()
host, port = "0.0.0.0", int(os.environ.get("PORT", "8000"))
mcp = MCPServer(
    "<server name>",
    token_verifier=PlugpassTokenVerifier(cfg),
    auth=AuthSettings(
        issuer_url=cfg.issuer,                 # → PRM authorization_servers
        resource_server_url=cfg.resource_url,  # MUST include /mcp — drives the PRM path AND its `resource`
        required_scopes=None,
        # The verifier pins `aud` to the same RESOURCE_URL itself, so the SDK's
        # own audience check would be a second copy of one rule.
        validate_token_resource=False,
    ),
)

# ...tool registrations (below)...

# /mcp (401-gated, BOTH protocol eras) + /.well-known/oauth-protected-resource/mcp
app = mcp.streamable_http_app(
    stateless_http=True,
    json_response=True,  # required: both legs buffer, so a revoked-bearer tool can swap in a 401
    host=host,
)
app.add_middleware(ReauthTo401Middleware, resource_metadata_url=cfg.prm_url)
uvicorn.run(app, host=host, port=port)
# NOT mcp.run(...) — it rebuilds the app internally and would drop the middleware.
```

**Per-request identity inside a tool**: declare `ctx: Context` and read the validated principal off the request — `request = ctx.request_context.request`; `sub = request.user.access_token.subject`; the raw bearer to forward is `request.user.access_token.token`. On reauth: `request.state.plugpass_reauth_required = True` (the scope-state flag the middleware reads — reliable in both session modes, unlike contextvars).

**The check proxy tool (check host only).** A pure pipe — never parse or reformat `result_text`:

```python
from typing import Annotated

from mcp import types
from mcp.server.mcpserver import Context
from pydantic import Field

CHECK_PREMIUM_ACCESS_DESCRIPTION = "<the VERBATIM description from TOOLS.md § The check proxy tool>"

@mcp.tool(name="{CheckToolName}", title="Check premium access",
          description=CHECK_PREMIUM_ACCESS_DESCRIPTION,
          annotations=types.ToolAnnotations(
              readOnlyHint=False, destructiveHint=False,
              idempotentHint=False, openWorldHint=False),
          structured_output=False)
async def check_premium_access(
    plugin_id: str,
    feature_id: str,
    ctx: Context,
    # Required for a skill's access check (it carries the installed bundle's
    # version); a tool-surface check does not use it.
    plugin_version: str | None = None,
    # Reserved for the Plugpass paywall's own use.
    status_code: Annotated[bool, Field(
        description="Never include this parameter in your tool calls under any circumstance."
    )] = False,
) -> str:
    request = ctx.request_context.request
    bearer = request.user.access_token.token
    # The paywall's read-only probe: asks whether this user is entitled NOW,
    # consuming nothing (check_remaining, never check_premium_access), and
    # answers in a code that is not a check result — no PLUGPASS_PLUGIN, no
    # USE_AUTHORIZED — so no access handler triggers on it and nothing
    # downstream can read it as a grant. It authorizes NOTHING; the call the
    # user retries is checked on its own.
    if status_code:
        probed = await entitlement(
            bearer, {"plugin_id": plugin_id, "feature_id": feature_id}, "check_remaining", cfg
        )
        if probed["status"] == "reauth_required":
            request.state.plugpass_reauth_required = True
            return "Re-authentication required."
        # ONLY a definite answer carries a code. "unavailable" covers both a
        # Plugpass outage and a feature this endpoint cannot evaluate, and
        # neither establishes that the user is unentitled — so the probe stays
        # silent rather than asserting a denial it did not establish.
        if probed["status"] == "unavailable":
            return "STATUS_UNKNOWN"
        return f"STATUS_CODE={'1' if probed['status'] == 'ok' else '0'}"
    body: dict[str, str] = {"plugin_id": plugin_id, "feature_id": feature_id}
    if plugin_version is not None:
        body["plugin_version"] = plugin_version
    result = await entitlement(bearer, body, "check_premium_access", cfg)
    if result["status"] == "unavailable":
        return UNAVAILABLE_CHECK_GRANT
    if result["status"] == "reauth_required":
        request.state.plugpass_reauth_required = True   # → transport-level 401 via the middleware
        return "Re-authentication required."            # discarded by the swap
    return result["result_text"]  # verbatim — byte-identical to the native tool
```

**Solo paid tool wrapper** (consume-on-invocation; no `auth_token` parameter on this path):

```python
# The registration's meta, hoisted so the marker rule reads the same object the
# tool registers with. A UI-backed tool keeps its "ui": {"resourceUri": …,
# "visibility": …} here beside the id.
PAID_TOOL_FEATURE_ID = "<plugpass_id>"
PAID_TOOL_META: dict[str, Any] = {"plugpass_component_id": PAID_TOOL_FEATURE_ID}

@mcp.tool(name="paid_tool", description="...", structured_output=False,
          # Metering makes this tool neither read-only nor idempotent, whatever it
          # was before the wrap. The other two hints keep the tool's own values.
          annotations=types.ToolAnnotations(
              readOnlyHint=False, destructiveHint=<the tool's own value>,
              idempotentHint=False, openWorldHint=<the tool's own value>),
          meta=PAID_TOOL_META)
async def paid_tool(..., ctx: Context) -> str:
    request = ctx.request_context.request
    sub = request.user.access_token.subject
    result = await entitlement(
        request.user.access_token.token,
        {"plugin_id": cfg.plugin_id, "feature_id": PAID_TOOL_FEATURE_ID},
        "track_usage", cfg,
    )
    if result["status"] == "reauth_required":
        request.state.plugpass_reauth_required = True
        return "Re-authentication required."
    # The denial names this call (the tool's name and its actual arguments); the
    # renderer reads the tool's own registered meta (its widget, who may call it)
    # and the request: never a baked per-tool constant.
    if result["status"] == "non_authorized":
        return non_authorized_tool_response(
            result, {"name": "paid_tool", "arguments": {...the call's arguments...}},
            PAID_TOOL_META, ctx, cfg, PAID_TOOL_FEATURE_ID,
        )
    # ...existing tool body — keyed / scoped to `sub`; "unavailable" consumed
    # nothing and grants...
```

**Paired-tool add side** (`operation: add`): same shape, and `feature_id: "<the tool's own plugpass_id>"` exactly as for a solo tool (its `tool_` prefix carries the feature type) — every gated artifact bakes its own component's id, and the `entitlement` subfield is identity, never a `feature_id`. Always **`check_remaining`**, passing the user's current count from the publisher's own store (scoped to `sub`) as `current_count` — see TOOLS.md → Reading the user's current count.

**Identity tool** (the paired remove side, or any per-user free tool): no Entitlement API call — read `ctx.request_context.request.user.access_token.subject` and scope the body to it.

**The in-widget paywall (`ui_paywall`, servers that render widgets).** The helper below, in `premium_feature_access_check.py`, is applied to every resource the server reads out whose MIME type is `text/html;profile=mcp-app`: the paywall script tag goes first in `<head>`, and the script's origin joins the resource's `resourceDomains`. The widget HTML itself is never edited.

```python
import re
from urllib.parse import urlsplit

_HEAD_OPEN = re.compile(r"<head(\s[^>]*)?>", re.IGNORECASE)


def with_paywall(html: str, csp: dict[str, list[str]], cfg: PlugpassConfig) -> tuple[str, dict[str, list[str]]]:
    """The script tag first in <head> (ahead of the widget's own code; prepended
    to the document when it has no <head>), its origin added to `resourceDomains`
    so the sandbox lets it load. Returns the injected HTML and the widened CSP."""
    tag = f'<script src="{cfg.paywall_script_url}"></script>'
    head = _HEAD_OPEN.search(html)
    injected = f"{tag}{html}" if head is None else f"{html[: head.end()]}{tag}{html[head.end() :]}"
    parts = urlsplit(cfg.paywall_script_url)
    origin = f"{parts.scheme}://{parts.netloc}"
    resource_domains = list(csp.get("resourceDomains", []))
    if origin not in resource_domains:
        resource_domains.append(origin)
    return injected, {**csp, "resourceDomains": resource_domains}
```

Every UI resource read passes through it. The SDK carries a resource's `meta` onto the contents it reads out (as `_meta`), so a static resource applies the helper at registration and declares the widened CSP there:

```python
html, csp = with_paywall(WIDGET_HTML, WIDGET_CSP, cfg)

@mcp.resource("ui://<plugin>/<widget>", name="widget", title="…", description="…",
              mime_type="text/html;profile=mcp-app", meta={"ui": {"csp": csp}})
def widget() -> str:
    return html
```

**Placement guidance.** An `MCPServer` keeps its existing tool modules; the verifier + middleware live in `premium_feature_access_check.py`; the entry file gains the `MCPServer(...)` auth kwargs + the app assembly. When the publisher mounts the MCP app inside a larger Starlette/FastAPI app instead, `Mount("/…", mcp.streamable_http_app())` works but the **parent** app's lifespan must run `mcp.session_manager.run()`, and the reauth middleware wraps the parent. Invariants regardless of layout: `stateless_http=True, json_response=True` (both legs buffered); `resource_server_url` includes `/mcp`; wrapped tools keep their hoisted `meta=` (the `plugpass_component_id`, plus `"ui"` on a UI-backed tool), declare `structured_output=False`, and never declare an output schema; on a server whose directive names `ui_paywall`, every `text/html;profile=mcp-app` resource read passes through `with_paywall`.
