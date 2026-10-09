# Python scaffolding template

The publisher's server becomes an **OAuth-protected resource server**: a pure-ASGI gate in front of the official MCP SDK's streamable-HTTP app validates bearers locally against Plugpass's JWKS, serves the RFC 9728 PRM documents at both paths, and answers requests without a valid bearer with the `401` + `WWW-Authenticate` challenge (`error="invalid_token"`, `resource_metadata="…"` — the exact shape MCP clients key OAuth discovery off) — at `/mcp` once the plugin is published, at `/mcp/test` from the first deploy. The gate hands the verified identity to the SDK's request context exactly as the SDK's own auth would (`request.user.access_token`); the SDK's `AuthSettings` are **not** used, since they challenge every request whatever the enforcement state. Standalone source, no platform package. Version pins: **`mcp>=2.2.0,<3`** (the 2.x line — it is what implements protocol 2026-07-28; on 1.x the server serves the legacy era only), `pyjwt[crypto]>=2.13.0`, `httpx`.

**Four structural decisions carry the whole design — never undo them:**

1. **The server serves BOTH protocol eras from the one app.** The SDK's session manager routes each request by its `MCP-Protocol-Version` header: 2026-07-28 to the modern, envelope-based leg (which owns `server/discover`), the handshake versions to their own. The client's UI capability rides the modern envelope — that is what the paywall-UI marker reads — and a host that gets only the legacy era never sends it. Nothing extra is wired for it: the SDK line does the era routing.
2. **`stateless_http=True, json_response=True` on the app, always.** In JSON mode BOTH legs write the HTTP response only after the tool handler finished, so an outer ASGI middleware can replace it with a `401` when a tool discovers mid-call that the bearer is revoked (`reauth_required` from the Entitlement API). In SSE mode the `200` headers hit the wire before the tool runs — the swap is impossible. Stateless also makes per-request auth context reliable.
3. **Every accepted audience comes from Plugpass, never from the request.** A bearer's `aud` must be the path's own resource — the baked `RESOURCE_URL` at `/mcp`, `{RESOURCE_URL}/test` at `/mcp/test` — or one of the retired URLs Plugpass reports for this server (the addresses it moved off, which old installs still call) in the same form. The two never cross. The PRM `resource` and the challenge's `resource_metadata` name the URL the request was addressed to (`X-Forwarded-Host`, else `Host`) only when that URL is an accepted audience, else `RESOURCE_URL` — with the `/test` suffix at the test path. This is what lets a locally-listening server accept real bearers minted for its public URL, and what keeps a moved server working for installs of its old address.
4. **The `/mcp` gate is armed by publish, and only by publish.** `enforced()` answers whether the plugin has a published version — fetched from Plugpass before the first request is handled, cached five minutes and refreshed in the background, keyed by `RESOURCE_URL` and nothing else, final once `True`, strict while there is no answer. Off, `/mcp` challenges nobody: a valid bearer is handled as when on, a missing (or malformed, or expired) bearer means no identity, and every tool states what it does with none. The test path never consults it.

```python
# premium_feature_access_check.py — written once per server. Standalone.
import asyncio
import json
import re
import time
import uuid
from collections.abc import Callable
from dataclasses import dataclass
from typing import Any
from urllib.parse import urlsplit

import anyio
import httpx
import jwt as pyjwt
from jwt import PyJWKClient
from jwt.exceptions import PyJWKClientConnectionError, PyJWKSetError
from mcp import types
from mcp.server.auth.middleware.bearer_auth import AuthenticatedUser
from mcp.server.auth.provider import AccessToken
from mcp.server.mcpserver import Context
from starlette.authentication import AuthCredentials, UnauthenticatedUser

MCP_PATH = "/mcp"
# The test path: the same app, strictly gated from the first deploy, for a
# resource of its own — `{RESOURCE_URL}/test` — that Plugpass mints only to the
# plugin's test users.
TEST_PATH_SUFFIX = "/test"
TEST_MCP_PATH = f"{MCP_PATH}{TEST_PATH_SUFFIX}"

# Plugpass endpoints for this server.
PLUGIN_ID = "<the plugin's Plugpass id>"
ISSUER = "<plugpass_issuer>"
JWKS_URL = "<plugpass_jwks_url>"
ENTITLEMENT_API_ORIGIN = "<entitlement_api_origin>"
# This server's own public MCP URL — its current bearer audience and the PRM's
# default `resource`.
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


PRM_PATH = f"/.well-known/oauth-protected-resource{MCP_PATH}"
TEST_PRM_PATH = f"{PRM_PATH}{TEST_PATH_SUFFIX}"


def prm_url(resource: str) -> str:
    # RFC 9728 path-aware form: origin + /.well-known/oauth-protected-resource +
    # the resource's path — the test document for a test resource.
    if resource.endswith(TEST_MCP_PATH):
        return f"{resource.removesuffix(TEST_MCP_PATH)}{TEST_PRM_PATH}"
    return f"{resource.removesuffix(MCP_PATH)}{PRM_PATH}"


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


# The URLs this server moved off, which Plugpass reports so old installs keep
# working. Fetched only when a bearer or a request names another address;
# cached 5 minutes (1 minute after a failed fetch, which accepts nothing extra).
RETIRED_TTL_S = 300
RETIRED_FAILURE_TTL_S = 60
_retired: tuple[frozenset[str], float] | None = None
_retired_lock = anyio.Lock()


async def retired_audiences(cfg: PlugpassConfig) -> frozenset[str]:
    global _retired
    if _retired is not None and _retired[1] > time.monotonic():
        return _retired[0]
    async with _retired_lock:
        if _retired is not None and _retired[1] > time.monotonic():
            return _retired[0]
        urls: frozenset[str] = frozenset()
        ttl = RETIRED_FAILURE_TTL_S
        try:
            async with httpx.AsyncClient(timeout=5.0) as client:
                res = await client.get(
                    f"{cfg.entitlement_api_origin}/entitlement/retired-audiences",
                    params={"resource": cfg.resource_url},
                )
            retired = res.json().get("retired") if res.status_code == 200 else None
            if isinstance(retired, list):
                urls = frozenset(u for u in retired if isinstance(u, str))
                ttl = RETIRED_TTL_S
        except Exception:
            pass  # unreachable: accept only RESOURCE_URL until the retry
        _retired = (urls, time.monotonic() + ttl)
        return urls


# The enforcement state — whether the plugin is published, the one input that
# turns the /mcp gate on. Fetched before the first request a process serves is
# handled (one attempt at a time; a concurrent caller waits on the lock and takes
# its answer), cached 5 minutes and refreshed in the background after that;
# keyed by RESOURCE_URL and by nothing in any request. True is final for the
# process. A failed fetch keeps the last answer; with no answer yet the gate is
# strict, and the fetch is retried after 1 minute.
ENFORCEMENT_TTL_S = 300
ENFORCEMENT_RETRY_S = 60
_enforcement: tuple[bool, float] | None = None  # (enforced, expires_at (monotonic))
_enforcement_attempt_at = 0.0
_enforcement_lock = anyio.Lock()
_enforcement_tasks: set[asyncio.Task[None]] = set()


async def _refresh_enforcement(cfg: PlugpassConfig) -> None:
    global _enforcement, _enforcement_attempt_at
    async with _enforcement_lock:
        now = time.monotonic()
        if now < _enforcement_attempt_at:
            return
        _enforcement_attempt_at = now + ENFORCEMENT_RETRY_S
        try:
            async with httpx.AsyncClient(timeout=5.0) as client:
                res = await client.get(
                    f"{cfg.entitlement_api_origin}/entitlement/enforcement",
                    params={"resource": cfg.resource_url},
                )
            enforced_answer = res.json().get("enforced") if res.status_code == 200 else None
            if isinstance(enforced_answer, bool):
                _enforcement = (enforced_answer, time.monotonic() + ENFORCEMENT_TTL_S)
        except Exception:
            pass  # unreachable: the last answer stands (strict while there is none) until the retry


async def enforced(cfg: PlugpassConfig) -> bool:
    answer = _enforcement
    if answer is not None and answer[0]:
        return True  # final
    if answer is None:
        # No answer yet: learn it before handling the request; strict until it arrives.
        await _refresh_enforcement(cfg)
        return _enforcement is None or _enforcement[0]
    if time.monotonic() >= answer[1] and time.monotonic() >= _enforcement_attempt_at:
        # Off and stale: refresh in the background, the cached answer serving meanwhile.
        task = asyncio.create_task(_refresh_enforcement(cfg))
        _enforcement_tasks.add(task)
        task.add_done_callback(_enforcement_tasks.discard)
    return answer[0]


async def addressed_resource(cfg: PlugpassConfig, scope, test: bool = False) -> str:
    """The URL a request was addressed to — `X-Forwarded-Host` (a proxy on an
    old address sets it), else its own host, on RESOURCE_URL's scheme and path —
    when that URL is an accepted audience; otherwise RESOURCE_URL. At the test
    path, the same with the `/test` suffix."""
    suffix = TEST_PATH_SUFFIX if test else ""
    headers = dict(scope.get("headers") or [])
    raw = headers.get(b"x-forwarded-host") or headers.get(b"host") or b""
    host = raw.decode("latin-1").split(",")[0].strip().lower()
    scheme, rest = cfg.resource_url.split("://", 1)
    current_host, _, path = rest.partition("/")
    if not host or host == current_host.lower():
        return cfg.resource_url + suffix
    candidate = f"{scheme}://{host}/{path}"
    accepted = candidate if candidate in await retired_audiences(cfg) else cfg.resource_url
    return accepted + suffix


class KeySetUnavailable(Exception):
    """The signing key set couldn't be fetched: no verdict on the bearer, so the
    gate answers 503, never the 401 a client reads as signed out."""


class PlugpassTokenVerifier:
    """Local JWKS validation, no Plugpass round-trip — the gate's verifier.
    EdDSA-pinned, issuer exact, exp required, audience = the path's own resource
    (the baked RESOURCE_URL, or its `/test` form at the test path) or one of this
    server's retired URLs in the same form; a canonical token is refused at the
    test path and a test token at the canonical path. PyJWKClient caches the key
    set (lifespan=4h) and refetches once on an unknown kid (key rotation)."""

    def __init__(self, cfg: PlugpassConfig) -> None:
        self._cfg = cfg
        # An explicit User-Agent: Plugpass's edge refuses urllib's default one.
        self._client = PyJWKClient(
            cfg.jwks_url, lifespan=14_400, headers={"User-Agent": "plugpass-resource-server"}
        )

    def _decode(self, token: str) -> dict:
        try:
            signing_key = self._client.get_signing_key_from_jwt(token)
        except (PyJWKClientConnectionError, PyJWKSetError, json.JSONDecodeError) as e:
            # A fetch that failed, or a body that isn't a key set; a key the
            # fetched set lacks is the token's fault (PyJWKClientError).
            raise KeySetUnavailable() from e
        return pyjwt.decode(
            token,
            signing_key.key,
            algorithms=["EdDSA"],
            issuer=self._cfg.issuer,
            # The audience is checked below, against RESOURCE_URL and the
            # retired set.
            options={"require": ["exp", "aud", "iss"], "verify_aud": False},
        )

    async def verify_token(self, token: str, test: bool = False) -> AccessToken | None:
        try:
            # PyJWKClient's fetch is blocking urllib — keep it off the event loop.
            payload = await anyio.to_thread.run_sync(self._decode, token)
        except KeySetUnavailable:
            raise
        except Exception:
            return None  # any other failure → no identity (the gate challenges when strict)
        suffix = TEST_PATH_SUFFIX if test else ""
        aud = payload.get("aud")
        auds = [aud] if isinstance(aud, str) else [a for a in aud or [] if isinstance(a, str)]
        if self._cfg.resource_url + suffix not in auds:
            retired = await retired_audiences(self._cfg)
            if not any(a.endswith(suffix) and a.removesuffix(suffix) in retired for a in auds):
                return None
        sub = payload.get("sub")
        if not isinstance(sub, str) or not sub:
            return None
        return AccessToken(
            token=token, client_id=sub, scopes=[],
            expires_at=payload.get("exp"), subject=sub, claims=payload,
        )


def identity(ctx: Context) -> AccessToken | None:
    """The verified identity the request carries — `subject` is the user id, `token`
    the bearer to forward — or None on a request with none (the /mcp gate off,
    before the plugin is published), which every tool handles explicitly."""
    user = ctx.request_context.request.user
    return user.access_token if isinstance(user, AuthenticatedUser) else None


# ---------------------------------------------------------------------------
# Entitlement API client (paid tools + the check proxy).
# ---------------------------------------------------------------------------

# Body: {"plugin_id", "feature_id"} plus a solo paid tool's "call_id" and
# "quantities", or a database-record add's "current_count" and "adding".
# Result: {"status": "ok", "remaining": int|None,
#          "settle": "none"|"on_failure"|"always",   ← what follows the body
#          "output": {"key": str, "max": int|None}|None}   ← cap the size argument at max
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
                await anyio.sleep(2)
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


async def settle(bearer: str, body: dict, cfg: PlugpassConfig) -> dict:
    """The settle closing a solo paid tool's call — body {"plugin_id",
    "feature_id", "call_id", "outcome": "success"|"failure", "quantities"?}: what
    a failed call's entry took is given back; a success reports its output count
    and is answered {"status": "ok", "deliver": int|None, "note": str|None,
    "partial": str|None}, or a denial when nothing could be delivered. The
    entry's timeout and retry; every other outcome (reauth included) is
    "unavailable", which delivers everything."""
    result = await entitlement(bearer, body, "settle", cfg)
    return {"status": "unavailable"} if result["status"] == "reauth_required" else result


def deliver_settled(
    result: types.CallToolResult,
    settled: dict,
    trim: Callable[[int], types.CallToolResult],
    call: dict[str, Any],
    tool_meta: dict[str, Any],
) -> types.CallToolResult:
    """A success settle applied to the tool's result: trimmed to `deliver` items
    in every representation (`trim` is the tool's own — its text and its
    structured content alike), the note added as a text block of its own and
    under the structured content's `limit_note` (a host may give the model the
    structured content alone), and the note in the composed check result's
    shape on `_meta.plugpass_partial`, beside the call echo, for the in-widget
    paywall."""
    delivered = result if settled["deliver"] is None else trim(settled["deliver"])
    if settled["note"] is None:
        return delivered
    meta = dict(delivered.meta or {})
    if settled["partial"] is not None:
        meta["plugpass_partial"] = settled["partial"]
        meta["plugpass_denied_call"] = {**call, "widget_callable": widget_callable(tool_meta)}
    structured = delivered.structured_content
    return types.CallToolResult(
        content=[*delivered.content, types.TextContent(type="text", text=settled["note"])],
        structuredContent=None if structured is None else {**structured, "limit_note": settled["note"]},
        _meta=meta,
    )


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
# grants and the paid skill runs; and its answer to a request with no identity
# (the /mcp gate off), composed here without reaching Plugpass. A wrapped tool
# needs no equivalent: it just runs its body.
UNAVAILABLE_CHECK_GRANT = "<the unavailable grant text from TOOLS.md>"


async def _send_json(send, status: int, body: dict, headers: tuple[tuple[bytes, bytes], ...] = ()) -> None:
    payload = json.dumps(body).encode()
    await send({"type": "http.response.start", "status": status, "headers": [
        (b"content-type", b"application/json"),
        (b"content-length", str(len(payload)).encode()),
        *headers,
    ]})
    await send({"type": "http.response.body", "body": payload})


async def _send_challenge(send, resource: str, description: str) -> None:
    # The 401 challenge for the addressed resource. error="invalid_token" exactly
    # — the signal MCP clients treat as "needs OAuth"; other codes read as a
    # broken server.
    header = (
        f'Bearer realm="{resource}", error="invalid_token", '
        f'error_description="{description}", resource_metadata="{prm_url(resource)}"'
    )
    await _send_json(
        send, 401, {"error": "invalid_token", "error_description": description},
        ((b"www-authenticate", header.encode()),),
    )


async def _send_key_set_unavailable(send) -> None:
    # The signing key set couldn't be fetched: no verdict on the bearer.
    await _send_json(
        send, 503,
        {"error": "temporarily_unavailable", "error_description": "The signing key set could not be resolved"},
        ((b"retry-after", b"30"), (b"cache-control", b"no-store")),
    )


class PlugpassGate:
    """Pure ASGI, outermost. Serves both PRM documents, gates /mcp and /mcp/test,
    and hands the verified identity to the SDK's request context — the same
    `request.user.access_token` the SDK's own auth populates. The test path is
    the same app: after the gate has read it, its path is rewritten to MCP_PATH
    on the way in. The gate: the test path is strict from the first deploy;
    /mcp challenges only while the plugin is published. Off, a request with no
    valid bearer is handled with no identity — never challenged."""

    def __init__(self, app, cfg: PlugpassConfig) -> None:
        self.app, self.cfg = app, cfg
        self.verifier = PlugpassTokenVerifier(cfg)

    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            return await self.app(scope, receive, send)
        path = scope.get("path", "")
        if path in (PRM_PATH, TEST_PRM_PATH):
            if scope.get("method") != "GET":
                return await _send_json(send, 405, {"error": "method_not_allowed"})
            resource = await addressed_resource(self.cfg, scope, test=path == TEST_PRM_PATH)
            return await _send_json(send, 200, {
                "resource": resource,
                "authorization_servers": [self.cfg.issuer],
                "bearer_methods_supported": ["header"],
            })
        if path not in (MCP_PATH, TEST_MCP_PATH):
            return await self.app(scope, receive, send)
        test = path == TEST_MCP_PATH
        resource = await addressed_resource(self.cfg, scope, test=test)
        strict = test or await enforced(self.cfg)
        headers = dict(scope.get("headers") or [])
        header = headers.get(b"authorization", b"").decode("latin-1")
        access: AccessToken | None = None
        if header[:7].lower() == "bearer ":
            try:
                access = await self.verifier.verify_token(header[7:].strip(), test=test)
            except KeySetUnavailable:
                return await _send_key_set_unavailable(send)
            if access is None and strict:
                return await _send_challenge(send, resource, "Token invalid or expired")
            # Off: a malformed or expired bearer is no bearer.
        elif strict:
            return await _send_challenge(send, resource, "Missing bearer token")
        # What the SDK's AuthenticationMiddleware would have set: the tools read
        # `request.user`, and the SDK's AuthContextMiddleware its contextvar.
        scope["user"] = AuthenticatedUser(access) if access is not None else UnauthenticatedUser()
        scope["auth"] = AuthCredentials([])
        state = scope.setdefault("state", {})
        state["plugpass_resource"] = resource  # the reauth swap's challenge names it
        if test:
            scope = {**scope, "path": MCP_PATH, "raw_path": MCP_PATH.encode()}
        await self.app(scope, receive, send)


class ReauthTo401Middleware:
    """Pure ASGI. When a tool set `plugpass_reauth_required` in the request
    scope state (the Entitlement API reported the bearer revoked), replace the
    whole response with the 401 challenge for the addressed resource — same
    shape as the initial one, so the client re-authorizes and retries. Requires
    json_response=True."""

    def __init__(self, app) -> None:
        self.app = app

    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            return await self.app(scope, receive, send)
        replaced = False

        async def wrapped_send(message):
            nonlocal replaced
            state = scope.get("state", {})
            if message["type"] == "http.response.start" and state.get("plugpass_reauth_required"):
                replaced = True
                return await _send_challenge(
                    send, state.get("plugpass_resource", RESOURCE_URL), "Access token no longer valid"
                )
            if replaced:
                return  # discard the inner app's response
            await send(message)

        await self.app(scope, receive, wrapped_send)
```

**Server entry** (`server.py`) — the SDK app without `AuthSettings`; the custom ASGI pieces are the SDK's `AuthContextMiddleware` (its contextvar off the gate's `scope["user"]`), the reauth middleware, and, outermost, the gate:

```python
import os

import uvicorn
from mcp.server.auth.middleware.auth_context import AuthContextMiddleware
from mcp.server.mcpserver import MCPServer

cfg = plugpass_config()
host, port = "0.0.0.0", int(os.environ.get("PORT", "8000"))
mcp = MCPServer("<server name>")  # no AuthSettings: the gate below is the auth

# ...tool registrations (below)...

# /mcp and /mcp/test (BOTH protocol eras) + both PRM documents, all through the gate
app = mcp.streamable_http_app(
    stateless_http=True,
    json_response=True,  # required: both legs buffer, so a revoked-bearer tool can swap in a 401
    host=host,
)
app.add_middleware(AuthContextMiddleware)
app.add_middleware(ReauthTo401Middleware)
app.add_middleware(PlugpassGate, cfg=cfg)  # added last = outermost
uvicorn.run(app, host=host, port=port)
# NOT mcp.run(...) — it rebuilds the app internally and would drop the middleware.
```

**Per-request identity inside a tool**: declare `ctx: Context` and read `access = identity(ctx)` — `access.subject` is the user id, `access.token` the raw bearer to forward, and `None` a request with no identity (the /mcp gate off, before the plugin is published), which every tool handles explicitly. On reauth: `ctx.request_context.request.state.plugpass_reauth_required = True` (the scope-state flag the middleware reads — reliable in both session modes, unlike contextvars).

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
    access = identity(ctx)
    # No identity — the /mcp gate off, before the plugin is published: the one
    # answer the proxy composes itself, without reaching Plugpass.
    if access is None:
        return UNAVAILABLE_CHECK_GRANT
    bearer = access.token
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

**Solo paid tool wrapper** (entry, body, settle; no `auth_token` parameter on this path). The example tool returns a list of leads, limited by `limit`, and is recorded with one output quantity, `leads`; report every recorded quantity this way (TOOLS.md → Reading a tool's quantities):

```python
# The registration's meta, hoisted so the marker rule reads the same object the
# tool registers with. A UI-backed tool keeps its "ui": {"resourceUri": …,
# "visibility": …} here beside the id.
PAID_TOOL_FEATURE_ID = "<plugpass_id>"
PAID_TOOL_META: dict[str, Any] = {"plugpass_component_id": PAID_TOOL_FEATURE_ID}
DEFAULT_LIMIT = 25  # the tool's own default for its size argument

@mcp.tool(name="find_leads", description="...", structured_output=False,
          # Metering makes this tool neither read-only nor idempotent, whatever it
          # was before the wrap. The other two hints keep the tool's own values.
          annotations=types.ToolAnnotations(
              readOnlyHint=False, destructiveHint=<the tool's own value>,
              idempotentHint=False, openWorldHint=<the tool's own value>),
          meta=PAID_TOOL_META)
async def find_leads(query: str, ctx: Context, limit: int | None = None) -> types.CallToolResult:
    request = ctx.request_context.request
    access = identity(ctx)
    sub = access.subject if access is not None else ""

    async def run(size: int) -> list[Lead]:
        ...  # existing tool body — keyed / scoped to `sub` (empty with no identity)

    def render(leads: list[Lead]) -> types.CallToolResult:
        # Every representation of the result, from its items.
        items = [lead.model_dump() for lead in leads]
        return types.CallToolResult(
            content=[types.TextContent(type="text", text=json.dumps(items))],
            structuredContent={"leads": items},
        )

    requested = limit if limit is not None else DEFAULT_LIMIT
    # No identity is the /mcp gate off (the plugin unpublished): the tool runs as
    # it did before Plugpass, with no entitlement call.
    if access is None:
        return render(await run(requested))

    call = {"name": "find_leads", "arguments": {"query": query, "limit": limit}}
    call_id = str(uuid.uuid4())
    result = await entitlement(
        access.token,
        {
            "plugin_id": cfg.plugin_id,
            "feature_id": PAID_TOOL_FEATURE_ID,
            "call_id": call_id,
            # Each recorded quantity: an input's count, an output's requested size.
            "quantities": {"leads": requested},
        },
        "track_usage", cfg,
    )
    if result["status"] == "reauth_required":
        request.state.plugpass_reauth_required = True
        return types.CallToolResult(content=[types.TextContent(type="text", text="Re-authentication required.")])
    # The denial names this call (the tool's name and its actual arguments); the
    # renderer reads the tool's own registered meta (its widget, who may call it)
    # and the request: never a baked per-tool constant.
    if result["status"] == "non_authorized":
        return non_authorized_tool_response(result, call, PAID_TOOL_META, ctx, cfg, PAID_TOOL_FEATURE_ID)
    # "unavailable" consumed nothing and grants: nothing to settle.
    settle_mode = result["settle"] if result["status"] == "ok" else "none"
    output = result.get("output") if result["status"] == "ok" else None
    cap = output["max"] if output is not None else None
    ids = {"plugin_id": cfg.plugin_id, "feature_id": PAID_TOOL_FEATURE_ID, "call_id": call_id}

    try:
        leads = await run(requested if cap is None else min(requested, cap))
    except Exception:
        # A failed call gets back what its entry took.
        if settle_mode != "none":
            await settle(access.token, {**ids, "outcome": "failure"}, cfg)
        raise
    full = render(leads)
    if settle_mode != "always":
        return full
    settled = await settle(
        access.token, {**ids, "outcome": "success", "quantities": {"leads": len(leads)}}, cfg
    )
    if settled["status"] == "non_authorized":
        return non_authorized_tool_response(settled, call, PAID_TOOL_META, ctx, cfg, PAID_TOOL_FEATURE_ID)
    # The unavailable grant delivers everything, with no note.
    if settled["status"] != "ok":
        return full
    return deliver_settled(full, settled, lambda deliver: render(leads[:deliver]), call, PAID_TOOL_META)
```

A tool that returns an error result (`is_error`) rather than raising settles the same failure before returning it. A tool with no output quantity settles only on a failure (`on_failure`) and returns its result as it is; an input quantity is counted from the arguments (`"quantities": {"enrichments": len(leads)}`). An output quantity of a tool with no size argument is reported only at the settle, its body uncapped.

**Paired-tool add side** (`operation: add`): same shape as the entry, and `feature_id: "<the tool's own plugpass_id>"` exactly as for a solo tool (its `tool_` prefix carries the feature type) — every gated artifact bakes its own component's id, and the `entitlement` subfield is identity, never a `feature_id`. Always **`check_remaining`**, passing the user's current count from the publisher's own store (scoped to `sub`) as `current_count` — see TOOLS.md → Reading the user's current count — and `adding`, how many records the call adds (`1`, or the list's length for a batch). No call id and no settle: a record add never consumes.

**Identity tool** (the paired remove side, or any per-user free tool): no Entitlement API call — read `identity(ctx)` and scope the body to its `subject`; with none (the /mcp gate off), answer that nobody is signed in and touch no record:

```python
access = identity(ctx)
if access is None:
    return "No user is signed in."
# ...existing tool body, scoped to access.subject...
```

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
