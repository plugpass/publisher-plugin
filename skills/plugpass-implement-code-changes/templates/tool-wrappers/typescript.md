# TypeScript scaffolding template

The publisher's server becomes an **OAuth-protected resource server**: every `/mcp` request carries `Authorization: Bearer …`, validated locally against Plugpass's JWKS with `jose`; requests without a valid bearer get the `401` + `WWW-Authenticate` challenge; the RFC 9728 PRM document serves at `/.well-known/oauth-protected-resource/mcp`. Emitted as standalone source in the publisher's repo (no platform-package dependency). Version pins: `@modelcontextprotocol/server` **^2.0.0** (the v2 line — it is what implements protocol 2026-07-28; the 1.x `@modelcontextprotocol/sdk` line does not, and a server on it serves the legacy era only), `jose` ^6.2.3 (Node ≥ 20 or Workers), `hono` ^4.7.0 for Hono servers, `zod` v4.

**Three structural decisions carry the whole design — never undo them:**

1. **The server serves BOTH protocol eras from the one `/mcp` route.** A 2026-07-28 request goes to `createMcpHandler` (which owns `server/discover` and the per-request `_meta` envelope); a 2025-era request, identified by `isLegacyRequest`, goes to a `WebStandardStreamableHTTPServerTransport`. The client's UI capability rides the modern envelope — that is what the paywall-UI marker reads — and a host that gets only the legacy era never sends it. Never collapse this to one leg.
2. **Both legs BUFFER their response** — `responseMode: 'json'` on the modern handler, `enableJsonResponse: true` on the legacy transport. The HTTP response then materializes only *after* every tool handler finished, which is what lets the route swap in a `401` when a tool discovered mid-call that the bearer is revoked (`reauth_required` from the Entitlement API). Streaming commits the `200` before the tool runs and the swap is impossible. Cost: request-scoped progress notifications are dropped — fine for tool servers. (The SDK's own `legacy: 'stateless'` fallback streams, which is why the legacy leg is constructed by hand instead.)
3. **The audience is the baked `RESOURCE_URL`, never the request host.** The PRM `resource`, the JWT `aud` pin, and the challenge's `resource_metadata` all derive from it. This is what makes a locally-listening server accept real bearers minted for its public URL (the local test loop), and it can't drift behind proxies.

```ts
// premium-feature-access-check.ts — written once per server. Standalone.
import type { AuthInfo, CallToolResult } from '@modelcontextprotocol/server';
import { createRemoteJWKSet, jwtVerify, type JWTVerifyGetKey } from 'jose';

// Plugpass endpoints for this server.
const PLUGIN_ID = '<the plugin's Plugpass id>';
const ISSUER = '<plugpass_issuer>';
const JWKS_URL = '<plugpass_jwks_url>';
const ENTITLEMENT_API_ORIGIN = '<entitlement_api_origin>';
// This server's own public MCP URL — the JWT `aud` pin and the PRM `resource`.
const RESOURCE_URL = '<this server's RESOURCE_URL>';
// Where the Plugpass paywall for MCP Apps widgets loads from (the `ui_paywall`
// layer — only on a server whose directive names it).
const PAYWALL_SCRIPT_URL = '<paywall_script_url>';
// This server's own check tool, named on a denial so the paywall can ask it
// whether the user became entitled. NULL on a server that hosts no check tool
// (the check proxy is registered on the plugin's check host only), in which case
// a denial carries no probe and the paywall says less, never something untrue.
const CHECK_TOOL_NAME: string | null = '<check_tool_name, or null off the check host>';

export interface PlugpassConfig {
  pluginId: string;
  issuer: string;
  jwksUrl: string;
  entitlementApiOrigin: string;
  resourceUrl: string;
  paywallScriptUrl: string;
  checkToolName: string | null;
}

export function plugpassConfig(): PlugpassConfig {
  return {
    pluginId: PLUGIN_ID,
    issuer: ISSUER,
    jwksUrl: JWKS_URL,
    entitlementApiOrigin: ENTITLEMENT_API_ORIGIN,
    resourceUrl: RESOURCE_URL,
    paywallScriptUrl: PAYWALL_SCRIPT_URL,
    checkToolName: CHECK_TOOL_NAME,
  };
}

export const prmPath = '/.well-known/oauth-protected-resource/mcp';
export const prmUrl = (cfg: PlugpassConfig) => new URL(prmPath, cfg.resourceUrl).toString();

// RFC 9728 protected-resource-metadata document (serve unauthenticated at prmPath).
export function prmDocument(cfg: PlugpassConfig) {
  return {
    resource: cfg.resourceUrl,
    authorization_servers: [cfg.issuer],
    bearer_methods_supported: ['header'] as const,
  };
}

// The 401 challenge. error="invalid_token" exactly — it's the signal MCP
// clients treat as "needs OAuth"; other codes read as a broken server.
export function unauthorized(cfg: PlugpassConfig, description: string): Response {
  return new Response(JSON.stringify({ error: 'invalid_token', error_description: description }), {
    status: 401,
    headers: {
      'content-type': 'application/json',
      'www-authenticate': `Bearer realm="${cfg.resourceUrl}", error="invalid_token", error_description="${description}", resource_metadata="${prmUrl(cfg)}"`,
    },
  });
}

// Per-URL JWKS, cached 4h; jose refetches on an unknown kid (throttled by
// cooldownDuration). Module-scope-safe on Workers: construction does no I/O.
const jwksByUrl = new Map<string, JWTVerifyGetKey>();
function getJwks(jwksUrl: string): JWTVerifyGetKey {
  let jwks = jwksByUrl.get(jwksUrl);
  if (!jwks) {
    jwks = createRemoteJWKSet(new URL(jwksUrl), {
      cacheMaxAge: 4 * 60 * 60 * 1000,
      cooldownDuration: 30_000,
      timeoutDuration: 5_000,
    });
    jwksByUrl.set(jwksUrl, jwks);
  }
  return jwks;
}

// EdDSA-pinned (no alg-confusion downgrade), issuer exact, audience = the baked
// RESOURCE_URL, exp enforced by jose. Returns the user id; throws on any failure.
export async function verifyBearer(token: string, cfg: PlugpassConfig): Promise<{ sub: string }> {
  const { payload } = await jwtVerify(token, getJwks(cfg.jwksUrl), {
    issuer: cfg.issuer,
    audience: cfg.resourceUrl,
    algorithms: ['EdDSA'],
  });
  if (typeof payload.sub !== 'string' || !payload.sub) throw new Error('Missing sub claim');
  return { sub: payload.sub };
}

// The AuthInfo the route builds from a verified bearer and passes to the handler;
// `sub` (the end user's id) rides `extra`. These readers keep the tools free of
// casts: `subFrom` returns the verified user id, `bearerFrom` the raw token.
export function subFrom(authInfo: AuthInfo | undefined): string {
  const sub = authInfo?.extra?.sub;
  return typeof sub === 'string' ? sub : '';
}

export function bearerFrom(authInfo: AuthInfo | undefined): string | undefined {
  return authInfo?.token;
}

// Per-request mutable holder the wrappers set when the Entitlement API reports
// the bearer revoked; the route swaps the whole HTTP response for the 401.
export interface ReauthSignal { triggered: boolean }

// ---------------------------------------------------------------------------
// Entitlement API client (paid tools + the check proxy).
// ---------------------------------------------------------------------------

export type EntitlementResult =
  | { status: 'ok'; remaining: number | null }
  // result_text is the complete server-composed check result — emitted
  // VERBATIM as the tool's text (never parsed or re-serialized).
  | { status: 'non_authorized'; result_text: string }
  | { status: 'reauth_required' }
  // Plugpass could not answer. Server-to-server from this host, so the end user
  // cannot have caused it: the caller GRANTS, having consumed nothing.
  | { status: 'unavailable' };

// One POST attempt per call is not enough: a transient upstream blip (a worker
// reload, a brief network fault) must not surface as a denial. Each request
// gets a 5s timeout and ONE retry after a 2s backoff on a thrown request or a
// 5xx; 4xx responses are terminal — never retried.
export async function entitlementPost(url: string, bearer: string, body: unknown): Promise<Response> {
  for (let attempt = 0; ; attempt++) {
    try {
      const res = await fetch(url, {
        method: 'POST',
        headers: { authorization: `Bearer ${bearer}`, 'content-type': 'application/json' },
        body: JSON.stringify(body),
        signal: AbortSignal.timeout(5000),
      });
      if (res.status >= 500 && attempt === 0) throw new Error(`entitlement upstream ${res.status}`);
      return res;
    } catch (err) {
      if (attempt > 0) throw err;
      await new Promise((resolve) => setTimeout(resolve, 2000));
    }
  }
}

// POSTs to `${entitlementApiOrigin}/entitlement/${op}` forwarding the request's
// bearer via entitlementPost (5s timeout per attempt, one 2s-backoff retry
// on a thrown request or a 5xx). A 401 maps to reauth; every other failure maps
// to `unavailable`, which grants.
export async function entitlement(
  bearer: string,
  body: { plugin_id: string; feature_id: string; current_count?: number },
  op: 'check_remaining' | 'track_usage',
  cfg: PlugpassConfig,
): Promise<EntitlementResult> {
  try {
    const res = await entitlementPost(`${cfg.entitlementApiOrigin}/entitlement/${op}`, bearer, body);
    if (res.status === 401) return { status: 'reauth_required' };
    if (!res.ok) return { status: 'unavailable' };
    return (await res.json()) as EntitlementResult;
  } catch {
    return { status: 'unavailable' };
  }
}

// non_authorized → result_text verbatim — a single-field pipe (the trigger keys
// inside it auto-fire the plugin's access-handler skill). Never parse,
// reformat, or re-serialize it. reauth is handled by the CALLER (set the
// ReauthSignal — the route turns it into the transport-level 401); it never
// renders as tool text.
//
// Three things ride beside it for the Plugpass paywall an MCP Apps widget shows
// (all inert everywhere else):
//   - The denial NAMES the call it denied, on the result's `_meta` — which
//     hosts pass to a widget and never show the model — so the paywall can
//     replay the same call once the user has upgraded, and says whether a
//     widget may call the tool at all (widgetCallable below): a host refuses
//     a widget's call to a model-only tool, so the paywall then hands the
//     retry to the conversation instead.
//   - The read-only status probe (statusProbe below), named only when that
//     replay is impossible.
//   - The marker: when the denied tool is UI-backed (it renders a widget) AND
//     the client renders widgets (paywallUi below), a second text block
//     carries PLUGPASS_PAYWALL_UI=true. The widget's paywall is then the one
//     asking the user, and the plugin's access-handler skill posts nothing
//     beside it. The composed text stays untouched in its own block.
// All read the tool's own registered `_meta` and the request's envelope, at runtime.
export interface DeniedCall {
  name: string;
  arguments: Record<string, unknown>;
}

export const PAYWALL_UI_MARKER = 'PLUGPASS_PAYWALL_UI=true';

export function nonAuthorizedToolResponse(
  r: Extract<EntitlementResult, { status: 'non_authorized' }>,
  call: DeniedCall,
  toolMeta: ToolMeta,
  requestEnvelope: unknown,
  cfg: PlugpassConfig,
  featureId: string,
): CallToolResult {
  return {
    content: paywallUi(toolMeta, requestEnvelope)
      ? [{ type: 'text', text: r.result_text }, { type: 'text', text: PAYWALL_UI_MARKER }]
      : [{ type: 'text', text: r.result_text }],
    _meta: {
      plugpass_denied_call: { ...call, widget_callable: widgetCallable(toolMeta) },
      ...statusProbe(cfg, toolMeta, featureId),
    },
  };
}

// The read-only entitlement probe the paywall may call when it cannot replay a
// denied call: this server's check tool, with the arguments already composed so
// the widget's script supplies nothing of its own. Named ONLY when the call is
// unreplayable (a model-only UI-backed tool) and this server hosts a check
// tool; every other denial carries none, since a replay answers the same
// question by actually running the call.
function statusProbe(
  cfg: PlugpassConfig,
  toolMeta: ToolMeta,
  featureId: string,
): { plugpass_status_probe?: { name: string; arguments: Record<string, unknown> } } {
  if (cfg.checkToolName === null || toolMeta.ui === undefined || widgetCallable(toolMeta)) return {};
  return {
    plugpass_status_probe: {
      name: cfg.checkToolName,
      arguments: { plugin_id: cfg.pluginId, feature_id: featureId, status_code: true },
    },
  };
}

// Whether the calling client renders MCP Apps widgets: it declared the UI
// extension among the client capabilities every 2026-07-28 request carries in
// its `_meta` envelope (`io.modelcontextprotocol/clientCapabilities`). Read off
// the tool handler's `ctx.mcpReq.envelope`, where the SDK lifts the reserved
// envelope keys on BOTH legs — so a 2025-era client that repeats the key
// per-request is read by the same rule. (`getClientCapabilities()` is the
// deprecated handshake-scoped accessor; the envelope is the source.) A request
// that declares nothing renders nothing and gets no marker.
const CLIENT_CAPABILITIES_META_KEY = 'io.modelcontextprotocol/clientCapabilities';
const UI_EXTENSION_ID = 'io.modelcontextprotocol/ui';

function isRecord(value: unknown): value is Record<string, unknown> {
  return typeof value === 'object' && value !== null && !Array.isArray(value);
}

export function clientRendersWidgets(envelope: unknown): boolean {
  if (!isRecord(envelope)) return false;
  const capabilities = envelope[CLIENT_CAPABILITIES_META_KEY];
  if (!isRecord(capabilities) || !isRecord(capabilities.extensions)) return false;
  return isRecord(capabilities.extensions[UI_EXTENSION_ID]);
}

// The `_meta` a tool registers with: its plugpass-component-id and, on a
// UI-backed tool, the widget it renders.
export type ToolMeta = {
  plugpass_component_id: string;
  ui?: { resourceUri: string; visibility?: Array<'model' | 'app'> };
};

// Whether the widget's paywall is the one asking on this denial: the tool
// renders a widget (its own registration declares `ui.resourceUri`) AND the
// client renders widgets. Read off the registration at runtime, so a tool that
// gains or loses its widget changes nothing here.
export function paywallUi(toolMeta: ToolMeta, requestEnvelope: unknown): boolean {
  return toolMeta.ui !== undefined && clientRendersWidgets(requestEnvelope);
}

// Who may call the tool, off the same registration: an undeclared visibility
// means the model and a widget both may; a declared list means exactly its
// members. A widget's paywall replays a denied call itself only when it may.
export function widgetCallable(toolMeta: ToolMeta): boolean {
  const visibility = toolMeta.ui?.visibility;
  return visibility === undefined || visibility.includes('app');
}

// The check proxy's unavailable grant — Plugpass could not answer, so the check
// grants and the paid skill runs. A wrapped tool needs no equivalent: it just
// runs its body.
export function unavailableCheckGrant(): CallToolResult {
  return { content: [{ type: 'text', text: '<the unavailable grant text from TOOLS.md>' }] };
}
```

**Server composition — the both-eras route.** One `/mcp` route, one gate, two legs. A fresh `McpServer` per request (stateless), registering every tool with this request's `ReauthSignal`:

```ts
import {
  McpServer,
  WebStandardStreamableHTTPServerTransport,
  createMcpHandler,
  isLegacyRequest,
  type AuthInfo,
} from '@modelcontextprotocol/server';
import { Hono } from 'hono';

function buildServer(cfg: PlugpassConfig, reauth: ReauthSignal): McpServer {
  const server = new McpServer({ name: '<server name>', version: '<version>' });
  /* …registerXxx(server, cfg, reauth, …) for every tool + the check proxy… */
  return server;
}

app.get(prmPath, (c) => c.json(prmDocument(plugpassConfig())));

app.all('/mcp', async (c) => {
  const cfg = plugpassConfig();
  // The gate: every method requires a valid bearer.
  const header = c.req.header('authorization');
  if (!header?.toLowerCase().startsWith('bearer ')) return unauthorized(cfg, 'Missing bearer token');
  const token = header.slice(7).trim();
  let sub: string;
  try {
    ({ sub } = await verifyBearer(token, cfg));
  } catch {
    return unauthorized(cfg, 'Token invalid or expired');
  }
  // The verified identity every tool reads as `ctx.http.authInfo`.
  const authInfo: AuthInfo = { token, clientId: sub, scopes: [], extra: { sub } };
  const reauth: ReauthSignal = { triggered: false };

  // 2025-era requests: a buffered streamable transport. (The modern handler owns
  // `server/discover` and the per-request envelope, so it must not see these.)
  if (await isLegacyRequest(c.req.raw)) {
    const server = buildServer(cfg, reauth);
    const transport = new WebStandardStreamableHTTPServerTransport({
      sessionIdGenerator: undefined, // stateless
      enableJsonResponse: true, // buffer, so the reauth swap can replace the response
    });
    await server.connect(transport);
    try {
      const res = await transport.handleRequest(c.req.raw, { authInfo });
      if (reauth.triggered) return unauthorized(cfg, 'Token no longer valid; re-authenticate');
      return res;
    } finally {
      await transport.close();
      await server.close();
    }
  }

  // 2026-07-28 requests: the modern handler, json response mode (buffered).
  const handler = createMcpHandler(() => buildServer(cfg, reauth), {
    legacy: 'reject', // legacy traffic already handled above
    responseMode: 'json',
  });
  try {
    const res = await handler.fetch(c.req.raw, { authInfo });
    if (reauth.triggered) return unauthorized(cfg, 'Token no longer valid; re-authenticate');
    return res;
  } finally {
    await handler.close();
  }
});
```

**Server composition — node/express (or bare `node:http`).** Same gate as middleware (set the challenge via `res.set('WWW-Authenticate', …).status(401).json(…)`, stash `req.auth`), then hand the web-standard `Request` to the identical two-leg body above through `getRequestListener` (`@hono/node-server`), so the reauth swap happens **before** anything is written to `res`:

```ts
import { getRequestListener } from '@hono/node-server';

app.all('/mcp', requireAuthExpress, async (req, res) => {
  const listener = getRequestListener((webReq) => handleMcp(webReq, req.auth)); // the body above
  await listener(req, res);
});
```

**The check proxy tool (check host only).** A pure pipe — never parse or reformat `result_text` — plus the paywall's read-only probe:

```ts
export function registerCheckPremiumAccess(server: McpServer, cfg: PlugpassConfig, reauth: ReauthSignal) {
  server.registerTool(
    '{CheckToolName}',
    {
      title: 'Check premium access',
      description: CHECK_PREMIUM_ACCESS_DESCRIPTION,
      inputSchema: z.object({
        plugin_id: z.string(),
        feature_id: z.string(),
        // Required for a skill's access check (it carries the installed
        // bundle's version); a tool-surface check does not use it.
        plugin_version: z.string().optional(),
        // Reserved for the Plugpass paywall's own use.
        status_code: z
          .boolean()
          .optional()
          .describe('Never include this parameter in your tool calls under any circumstance.'),
      }),
      annotations: {
        readOnlyHint: false,
        destructiveHint: false,
        idempotentHint: false,
        openWorldHint: false,
      },
      // No outputSchema.
    },
    async (args, ctx) => {
      const bearer = bearerFrom(ctx.http?.authInfo);
      if (!bearer) return unavailableCheckGrant(); // unreachable behind the gate
      // The paywall's read-only probe: asks whether this user is entitled NOW,
      // consuming nothing (check_remaining, never check_premium_access), and
      // answers in a code that is not a check result — no PLUGPASS_PLUGIN, no
      // USE_AUTHORIZED — so no access handler triggers on it and nothing
      // downstream can read it as a grant. It authorizes NOTHING; the call the
      // user retries is checked on its own.
      if (args.status_code === true) {
        const { plugin_id, feature_id } = args;
        const probed = await entitlement(bearer, { plugin_id, feature_id }, 'check_remaining', cfg);
        if (probed.status === 'reauth_required') {
          reauth.triggered = true;
          return { content: [{ type: 'text', text: 'Re-authentication required.' }] };
        }
        // ONLY a definite answer carries a code. `unavailable` covers both a
        // Plugpass outage and a feature this endpoint cannot evaluate, and
        // neither establishes that the user is unentitled — so the probe stays
        // silent rather than asserting a denial it did not establish.
        if (probed.status === 'unavailable') {
          return { content: [{ type: 'text', text: 'STATUS_UNKNOWN' }] };
        }
        return {
          content: [{ type: 'text', text: `STATUS_CODE=${probed.status === 'ok' ? '1' : '0'}` }],
        };
      }
      try {
        const res = await entitlementPost(
          `${cfg.entitlementApiOrigin}/entitlement/check_premium_access`,
          bearer,
          args
        );
        if (res.status === 401) { reauth.triggered = true; return { content: [{ type: 'text', text: 'Re-authentication required.' }] }; }
        if (!res.ok) return unavailableCheckGrant();
        const data = (await res.json()) as { status: 'ok'; result_text: string } | { status: 'reauth_required' };
        if (data.status === 'reauth_required') { reauth.triggered = true; return { content: [{ type: 'text', text: 'Re-authentication required.' }] }; }
        return { content: [{ type: 'text', text: data.result_text }] }; // verbatim — byte-identical to the native tool
      } catch {
        return unavailableCheckGrant();
      }
    },
  );
}
```

**Solo paid tool wrapper** (consume-on-invocation; identity + bearer from `extra.authInfo` — there is no `auth_token` parameter on this path):

```ts
// The registration's `_meta`, hoisted so the marker rule reads the same object
// the tool registers with. A UI-backed tool keeps its `ui: { resourceUri,
// visibility }` here beside the id.
const PAID_TOOL_FEATURE_ID = '<plugpass_id>';
const PAID_TOOL_META: ToolMeta = { plugpass_component_id: PAID_TOOL_FEATURE_ID };

server.registerTool(
  'paid_tool',
  {
    /* …existing config… */
    /* a wrapped tool declares no outputSchema — paywall/reauth replies are text */
    // Metering makes this tool neither read-only nor idempotent, whatever it was
    // before the wrap. The other two hints keep the tool's own values.
    annotations: { readOnlyHint: false, destructiveHint: <the tool's own value>, idempotentHint: false, openWorldHint: <the tool's own value> },
    _meta: PAID_TOOL_META,
  },
  async (args, ctx) => {
    const bearer = bearerFrom(ctx.http?.authInfo);
    const sub = subFrom(ctx.http?.authInfo);
    const r: EntitlementResult = bearer
      ? await entitlement(bearer, { plugin_id: cfg.pluginId, feature_id: PAID_TOOL_FEATURE_ID }, 'track_usage', cfg)
      : { status: 'unavailable' };
    if (r.status === 'reauth_required') { reauth.triggered = true; return { content: [{ type: 'text', text: 'Re-authentication required.' }] }; }
    // The denial names this call (the tool's name and its actual arguments); the
    // renderer reads the tool's own registered `_meta` (its widget, who may call
    // it) and the request's envelope: never a baked per-tool constant.
    if (r.status === 'non_authorized') {
      return nonAuthorizedToolResponse(
        r,
        { name: 'paid_tool', arguments: args },
        PAID_TOOL_META,
        ctx.mcpReq.envelope,
        cfg,
        PAID_TOOL_FEATURE_ID,
      );
    }
    /* …existing tool body — keyed / scoped to `sub`; `unavailable` consumed nothing and grants… */
  },
);
```

**Paired-tool add side** (`operation: add`): same shape, and `feature_id: '<the tool's own plugpass_id>'` exactly as for a solo tool (its `tool_` prefix carries the feature type) — every gated artifact bakes its own component's id, and the `entitlement` subfield is identity, never a `feature_id`. Always **`check_remaining`**, passing the user's current count from the publisher's own store (scoped to `sub`) as `current_count` — see TOOLS.md → Reading the user's current count.

**Identity tool** (the paired remove side, or any per-user free tool): no Entitlement API call at all — read `subFrom(ctx.http?.authInfo)` and scope the body to it. Zero Plugpass round-trips.

**The in-widget paywall (`ui_paywall`, servers that render widgets).** The helper below, in `premium-feature-access-check.ts`, is applied to every resource the server reads out whose MIME type is `text/html;profile=mcp-app`: the paywall script tag goes first in `<head>`, and the script's origin joins the resource's `resourceDomains`. The widget HTML itself is never edited.

```ts
// The CSP a UI resource declares on its contents (`_meta.ui.csp`).
export interface UiResourceCsp {
  connectDomains?: string[];
  resourceDomains?: string[];
  frameDomains?: string[];
  baseUriDomains?: string[];
}

// Loads the Plugpass paywall into a widget's HTML on its way out: the script
// tag first in <head> (ahead of the widget's own code; prepended to the
// document when it has no <head>), its origin added to `resourceDomains` so
// the sandbox lets it load.
export function withPaywall(html: string, csp: UiResourceCsp, cfg: PlugpassConfig): { html: string; csp: UiResourceCsp } {
  const tag = `<script src="${cfg.paywallScriptUrl}"></script>`;
  const head = /<head(\s[^>]*)?>/i.exec(html);
  const injected =
    head === null
      ? `${tag}${html}`
      : `${html.slice(0, head.index + head[0].length)}${tag}${html.slice(head.index + head[0].length)}`;
  const origin = new URL(cfg.paywallScriptUrl).origin;
  const resourceDomains = csp.resourceDomains ?? [];
  return {
    html: injected,
    csp: { ...csp, resourceDomains: resourceDomains.includes(origin) ? resourceDomains : [...resourceDomains, origin] },
  };
}
```

Every UI resource read passes through it:

```ts
server.registerResource(
  'widget',
  'ui://<plugin>/<widget>',
  { title: '…', description: '…', mimeType: 'text/html;profile=mcp-app' },
  (uri) => {
    const { html, csp } = withPaywall(WIDGET_HTML, WIDGET_CSP, cfg);
    return Promise.resolve({
      contents: [{ uri: uri.href, mimeType: 'text/html;profile=mcp-app', text: html, _meta: { ui: { csp } } }],
    });
  },
);
```

**Placement guidance.** Adapt to the publisher's structure: a Hono/Workers server follows the reference shape above; an express or bare-`node:http` server uses the `getRequestListener` variant. Whatever the layout, the invariants are: the gate covers every `/mcp` method; the PRM route is unauthenticated; **both protocol legs are present and both buffer** (`responseMode: 'json'` and `enableJsonResponse: true`); the reauth check sits between the leg resolving and the response being returned; wrapped tools keep `_meta.plugpass_component_id` and lose any `outputSchema`; on a server whose directive names `ui_paywall`, every `text/html;profile=mcp-app` resource read passes through `withPaywall`.
