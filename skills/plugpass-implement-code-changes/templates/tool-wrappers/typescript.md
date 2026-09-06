# TypeScript scaffolding template

The publisher's server becomes an **OAuth-protected resource server**: every `/mcp` request carries `Authorization: Bearer …`, validated locally against Plugpass's JWKS with `jose`; requests without a valid bearer get the `401` + `WWW-Authenticate` challenge; the RFC 9728 PRM document serves at `/.well-known/oauth-protected-resource/mcp`. Emitted as standalone source in the publisher's repo (no platform-package dependency). Version pins: `@modelcontextprotocol/sdk` **^1.29.0 (stay on 1.x — v2 splits the packages and breaks this surface)**, `jose` ^6.2.3 (Node ≥ 20 or Workers), `@hono/mcp` ^0.3.0 for Hono servers, `zod` (v3.25+ or v4).

**Two structural decisions carry the whole design — never undo them:**

1. **`enableJsonResponse: true` on the transport, always.** In JSON mode the transport's `handleRequest` resolves only *after* every tool handler finished, with a not-yet-sent `Response` — which is what lets the route swap in a `401` when a tool discovered mid-call that the bearer is revoked (`reauth_required` from the Entitlement API). In SSE mode (the default) the `200` is committed before the tool runs and the swap is impossible. Cost: request-scoped progress notifications are dropped — fine for tool servers.
2. **The audience is the baked `RESOURCE_URL`, never the request host.** The PRM `resource`, the JWT `aud` pin, and the challenge's `resource_metadata` all derive from it. This is what makes a locally-listening server accept real bearers minted for its public URL (the local test loop), and it can't drift behind proxies.

```ts
// premium-feature-access-check.ts — written once per server. Standalone.
import { createRemoteJWKSet, jwtVerify, type JWTVerifyGetKey } from 'jose';
import type { CallToolResult } from '@modelcontextprotocol/sdk/types.js';

// Plugpass endpoints for this server.
const ISSUER = '<plugpass_issuer>';
const JWKS_URL = '<plugpass_jwks_url>';
const ENTITLEMENT_API_ORIGIN = '<entitlement_api_origin>';
// This server's own public MCP URL — the JWT `aud` pin and the PRM `resource`.
const RESOURCE_URL = '<this server's RESOURCE_URL>';

export interface PlugpassConfig {
  issuer: string;
  jwksUrl: string;
  entitlementApiOrigin: string;
  resourceUrl: string;
}

export function plugpassConfig(): PlugpassConfig {
  return {
    issuer: ISSUER,
    jwksUrl: JWKS_URL,
    entitlementApiOrigin: ENTITLEMENT_API_ORIGIN,
    resourceUrl: RESOURCE_URL,
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
export function nonAuthorizedToolResponse(r: Extract<EntitlementResult, { status: 'non_authorized' }>): CallToolResult {
  return { content: [{ type: 'text', text: r.result_text }] };
}

// The check proxy's unavailable grant — Plugpass could not answer, so the check
// grants and the paid skill runs. A wrapped tool needs no equivalent: it just
// runs its body.
export function unavailableCheckGrant(): CallToolResult {
  return { content: [{ type: 'text', text: '<the unavailable grant text from TOOLS.md>' }] };
}
```

**Server composition — Hono (Workers / Bun / Node).** Per-request `McpServer` + transport (the stateless pattern); the middleware verifies once and stashes `AuthInfo` under the Hono variable `auth`, which `@hono/mcp` forwards to every tool handler as `extra.authInfo`:

```ts
import { StreamableHTTPTransport } from '@hono/mcp';
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import type { AuthInfo } from '@modelcontextprotocol/sdk/server/auth/types.js';

type PlugpassEnv = { Bindings: { /* …env vars… */ }; Variables: { auth: AuthInfo } };
const app = new Hono<PlugpassEnv>();

app.get(prmPath, (c) => c.json(prmDocument(plugpassConfig())));

app.all('/mcp', async (c) => {
  const cfg = plugpassConfig();
  // The gate: every method (POST, GET, DELETE) requires a valid bearer.
  const header = c.req.header('authorization');
  if (!header?.toLowerCase().startsWith('bearer ')) return unauthorized(cfg, 'Missing bearer token');
  const token = header.slice(7).trim();
  let sub: string;
  try {
    ({ sub } = await verifyBearer(token, cfg));
  } catch {
    return unauthorized(cfg, 'Token invalid or expired');
  }
  c.set('auth', { token, clientId: sub, scopes: [], extra: { sub } });

  const parsedBody = c.req.method === 'POST' ? await c.req.json().catch(() => undefined) : undefined;
  const reauth: ReauthSignal = { triggered: false };
  const server = new McpServer({ name: '<server name>', version: '<version>' });
  /* …registerXxx(server, cfg, reauth, …) for every tool + the check proxy… */

  const transport = new StreamableHTTPTransport({ enableJsonResponse: true }); // required: lets a revoked-bearer tool swap in a 401
  await server.connect(transport);
  const res = await transport.handleRequest(c, parsedBody); // JSON mode: resolves AFTER tools ran, Response not yet sent
  if (reauth.triggered) return unauthorized(cfg, 'Token no longer valid; re-authenticate');
  return res;
});
```

**Server composition — node/express (or bare `node:http`).** Same gate as middleware (set the challenge via `res.set('WWW-Authenticate', …).status(401).json(…)`, stash `req.auth`); the MCP route uses the SDK's web-standard transport through `getRequestListener` so the reauth swap happens **before** anything is written to `res` (the plain Node `StreamableHTTPServerTransport.handleRequest(req, res, body)` pumps the response before resolving — the swap must live inside the fetch callback):

```ts
import { getRequestListener } from '@hono/node-server'; // already a dependency of the SDK
import { WebStandardStreamableHTTPServerTransport } from '@modelcontextprotocol/sdk/server/webStandardStreamableHttp.js';

app.all('/mcp', requireAuthExpress, async (req, res) => {
  const cfg = plugpassConfig();
  const reauth: ReauthSignal = { triggered: false };
  const server = buildServer(cfg, reauth);
  const transport = new WebStandardStreamableHTTPServerTransport({
    sessionIdGenerator: undefined, // stateless
    enableJsonResponse: true,
  });
  await server.connect(transport);
  const listener = getRequestListener(async (webReq) => {
    const mcpRes = await transport.handleRequest(webReq, { authInfo: req.auth, parsedBody: req.body });
    if (reauth.triggered) return unauthorized(cfg, 'Token no longer valid; re-authenticate');
    return mcpRes;
  });
  await listener(req, res);
  res.on('close', () => { void transport.close(); void server.close(); });
});
```

**The check proxy tool (check host only).** A pure pipe — never parse or reformat `result_text`:

```ts
export function registerCheckPremiumAccess(server: McpServer, cfg: PlugpassConfig, reauth: ReauthSignal) {
  server.registerTool(
    '{CheckToolName}',
    {
      title: 'Check premium access',
      description: CHECK_PREMIUM_ACCESS_DESCRIPTION,
      inputSchema: {
        plugin_id: z.string(),
        feature_id: z.string(),
        plugin_version: z.string(),
      },
      annotations: {
        readOnlyHint: false,
        destructiveHint: false,
        idempotentHint: false,
        openWorldHint: false,
      },
      // No outputSchema.
    },
    async (args, extra) => {
      const bearer = extra.authInfo?.token;
      if (!bearer) return unavailableCheckGrant(); // unreachable behind the gate
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
server.registerTool(
  'paid_tool',
  {
    /* …existing config… */
    /* a wrapped tool declares no outputSchema — paywall/reauth replies are text */
    // Metering makes this tool neither read-only nor idempotent, whatever it was
    // before the wrap. The other two hints keep the tool's own values.
    annotations: { readOnlyHint: false, destructiveHint: <the tool's own value>, idempotentHint: false, openWorldHint: <the tool's own value> },
    _meta: { plugpass_component_id: '<plugpass_id>' },
  },
  async (args, extra) => {
    const auth = extra.authInfo;
    const sub = auth?.extra?.sub as string;
    const r: EntitlementResult = auth?.token
      ? await entitlement(auth.token, { plugin_id: '<plugin-plugpass-id>', feature_id: '<plugpass_id>' }, 'track_usage', cfg)
      : { status: 'unavailable' };
    if (r.status === 'reauth_required') { reauth.triggered = true; return { content: [{ type: 'text', text: 'Re-authentication required.' }] }; }
    if (r.status === 'non_authorized') return nonAuthorizedToolResponse(r);
    /* …existing tool body — keyed / scoped to `sub`; `unavailable` consumed nothing and grants… */
  },
);
```

**Paired-tool add side** (`operation: add`): same shape, but `feature_id: '<custom-entitlement-plugpass-id>'` (the add tool's `database_record.custom_entitlement_id` — its `custom_` prefix carries the feature type; **not** `database_record.plugpass_id`, and **not** the tool's own `_meta` id), always **`check_remaining`**, passing the user's current count from the publisher's own store (scoped to `sub`) as `current_count` — see TOOLS.md → Reading the user's current count.

**Identity tool** (the paired remove side, or any per-user free tool): no Entitlement API call at all — read `extra.authInfo.extra.sub` and scope the body to it. Zero Plugpass round-trips.

**Placement guidance.** Adapt to the publisher's structure: a Hono/Workers server follows the reference shape above; an express or bare-`node:http` server uses the `getRequestListener` variant. Whatever the layout, the invariants are: the gate covers every `/mcp` method; the PRM route is unauthenticated; `enableJsonResponse: true` on every transport construction; the reauth check sits between `handleRequest` resolving and the response being returned; wrapped tools keep `_meta.plugpass_component_id` and lose any `outputSchema`.
