# Go scaffolding template

The developer's server becomes an **OAuth-protected resource server** using the official SDK's `auth` package: `auth.RequireBearerToken` gates `/mcp` (emitting the `WWW-Authenticate` challenge with `resource_metadata`), `auth.ProtectedResourceMetadataHandler` serves the RFC 9728 PRM document, and a custom `TokenVerifier` does the local JWKS validation. Standalone source, no platform package. Version pins: `github.com/modelcontextprotocol/go-sdk` **v1.6.1** (v1.7.0 is pre-release for the next protocol — stay on stable), `github.com/golang-jwt/jwt/v5` v5.3.x, `github.com/MicahParks/keyfunc/v3` v3.8.x.

**Two structural decisions carry the whole design — never undo them:**

1. **`StreamableHTTPOptions{Stateless: true, JSONResponse: true}`, always.** In JSON mode the SDK buffers the response and `ServeHTTP` returns only after the tool handler finished — so a wrapping middleware can discard the buffered response and write a `401` when a tool discovered mid-call that the bearer is revoked. In SSE mode events flush immediately (the `200` is committed before the handler runs). And **stateless mode is what makes middleware context values visible inside tool handlers** — in stateful mode handler contexts descend from the *initialize* request, not the current POST, and the reauth flag silently never fires.

> **The gate's challenge needs a header fixup.** `auth.RequireBearerToken` emits `WWW-Authenticate: Bearer resource_metadata="…"` with **no** `error="invalid_token"` (RFC 6750 permits omitting the error on a missing token) — but the Plugpass contract requires `error="invalid_token"` (the signal clients key OAuth discovery off). The `challengeWriter`/`withFullChallenge` wrapper below fills it in on the gate's 401. Every other language SDK emits the full challenge itself; only the Go SDK needs this one wrapper.
2. **The audience is the baked `RESOURCE_URL`, never the request host** — the JWT `aud` pin, the PRM `resource`, and the challenge's `resource_metadata` all derive from it. This is what lets a locally-listening server accept real bearers minted for its public URL.

```go
// premium_feature_access_check.go — written once per server. Standalone.
package main

import (
	"cmp"
	"context"
	"fmt"
	"net/http"
	"net/url"
	"os"
	"strings"
	"sync"
	"sync/atomic"

	"github.com/MicahParks/keyfunc/v3"
	"github.com/golang-jwt/jwt/v5"
	"github.com/modelcontextprotocol/go-sdk/auth"
)

const mcpPath = "/mcp"

// Plugpass endpoints for this server.
const (
	issuer               = "<plugpass_issuer>"
	jwksURL              = "<plugpass_jwks_url>"
	entitlementAPIOrigin = "<entitlement_api_origin>"
	// This server's own public MCP URL — the JWT aud pin and the PRM resource.
	resourceURL = "<this server's RESOURCE_URL>"
)

type plugpassConfig struct {
	issuer, jwksURL, entitlementAPIOrigin, resourceURL string
}

func loadPlugpassConfig() plugpassConfig {
	return plugpassConfig{
		issuer:               issuer,
		jwksURL:              jwksURL,
		entitlementAPIOrigin: entitlementAPIOrigin,
		resourceURL:          resourceURL,
	}
}

// RFC 9728 path-aware PRM URL: origin + /.well-known/oauth-protected-resource + /mcp.
func prmURL(cfg plugpassConfig) string {
	u, _ := url.Parse(cfg.resourceURL)
	return u.Scheme + "://" + u.Host + "/.well-known/oauth-protected-resource" + mcpPath
}

// Lazy per-URL keyfunc singleton. keyfunc.NewDefault refreshes hourly in the
// background (inside the 4h ceiling) and refetches on an unknown kid
// (rate-limited) — the rotation semantics we need, built in.
var (
	jwksMu    sync.Mutex
	jwksByURL = map[string]jwt.Keyfunc{}
)

func jwksKeyfunc(jwksURL string) (jwt.Keyfunc, error) {
	jwksMu.Lock()
	defer jwksMu.Unlock()
	if kf, ok := jwksByURL[jwksURL]; ok {
		return kf, nil
	}
	kf, err := keyfunc.NewDefault([]string{jwksURL})
	if err != nil {
		return nil, err
	}
	jwksByURL[jwksURL] = kf.Keyfunc
	return kf.Keyfunc, nil
}

// The auth.TokenVerifier the bearer gate runs: EdDSA-pinned, issuer exact,
// audience = the baked RESOURCE_URL, exp required; extracts sub. Every token
// failure MUST wrap auth.ErrInvalidToken — anything else becomes a 500 with
// no WWW-Authenticate challenge, silently breaking client OAuth discovery.
func verifyBearer(cfg plugpassConfig) auth.TokenVerifier {
	return func(ctx context.Context, token string, _ *http.Request) (*auth.TokenInfo, error) {
		kf, err := jwksKeyfunc(cfg.jwksURL)
		if err != nil {
			return nil, err // infra failure → 500 (correct: not a token problem)
		}
		tok, err := jwt.Parse(token, kf,
			jwt.WithValidMethods([]string{"EdDSA"}),
			jwt.WithIssuer(cfg.issuer),
			jwt.WithAudience(cfg.resourceURL),
			jwt.WithExpirationRequired(),
		)
		if err != nil || !tok.Valid {
			return nil, fmt.Errorf("%w: %v", auth.ErrInvalidToken, err)
		}
		sub, err := tok.Claims.GetSubject()
		if err != nil || sub == "" {
			return nil, fmt.Errorf("%w: missing sub", auth.ErrInvalidToken)
		}
		exp, err := tok.Claims.GetExpirationTime()
		if err != nil || exp == nil {
			return nil, fmt.Errorf("%w: missing exp", auth.ErrInvalidToken)
		}
		return &auth.TokenInfo{
			Expiration: exp.Time, // must be non-zero or the middleware 401s independently
			UserID:     sub,
			Extra:      map[string]any{"raw_token": token}, // the bearer the wrappers forward
		}, nil
	}
}

// Per-request reauth signal, injected by reauthTo401 below; tool handlers set
// it via the request context (visible because the transport is stateless).
type reauthSignal struct{ v atomic.Bool }
type reauthKey struct{}

func setReauthRequired(ctx context.Context) {
	if s, ok := ctx.Value(reauthKey{}).(*reauthSignal); ok {
		s.v.Store(true)
	}
}

// toolHints builds the ToolAnnotations every tool declares. Take the helper,
// never hand-build the struct: DestructiveHint and
// OpenWorldHint are *bool precisely because their spec defaults are TRUE, so
// leaving either nil serializes as absent and the client reads the pessimistic
// default — a read-only tool then shows up in ChatGPT as destructive and
// open-world. (ReadOnlyHint / IdempotentHint are plain bools defaulting to false,
// so omitting those is harmless; set all four anyway — OpenAI's app submission
// requires readOnlyHint / destructiveHint / openWorldHint to be stated.)
func toolHints(readOnly, destructive, idempotent, openWorld bool) *mcp.ToolAnnotations {
	return &mcp.ToolAnnotations{
		ReadOnlyHint:    readOnly,
		DestructiveHint: &destructive,
		IdempotentHint:  idempotent,
		OpenWorldHint:   &openWorld,
	}
}

// challengeWriter fills in the auth gate's 401 WWW-Authenticate header with
// error="invalid_token" — the exact signal MCP clients key OAuth discovery off.
// auth.RequireBearerToken emits only `resource_metadata=…` (RFC 6750 permits
// omitting the error on a missing token), so add the full challenge when the
// header lacks an `error=`; a header that already carries one (reauthTo401's) is
// left as-is.
type challengeWriter struct {
	http.ResponseWriter
	cfg plugpassConfig
}

func (c *challengeWriter) WriteHeader(status int) {
	if status == http.StatusUnauthorized && !strings.Contains(c.Header().Get("WWW-Authenticate"), "error=") {
		c.Header().Set("WWW-Authenticate",
			fmt.Sprintf("Bearer realm=%q, error=%q, error_description=%q, resource_metadata=%q",
				c.cfg.resourceURL, "invalid_token", "Authentication required", prmURL(cfg)))
	}
	c.ResponseWriter.WriteHeader(status)
}

func withFullChallenge(cfg plugpassConfig, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		next.ServeHTTP(&challengeWriter{ResponseWriter: w, cfg: cfg}, r)
	})
}

// Buffers the MCP handler's (JSON-mode) response; when the tool flagged
// reauth, discards it and answers the transport-level 401 challenge instead —
// the same shape as an invalid bearer, so the client re-authorizes and retries.
func reauthTo401(cfg plugpassConfig, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		sig := &reauthSignal{}
		r = r.WithContext(context.WithValue(r.Context(), reauthKey{}, sig))
		bw := newBufferingResponseWriter() // captures Header/WriteHeader/Write; no-op Flush
		next.ServeHTTP(bw, r)              // returns only after the SDK delivered the full JSON body
		if sig.v.Load() {
			w.Header().Set("WWW-Authenticate",
				fmt.Sprintf("Bearer realm=%q, error=%q, error_description=%q, resource_metadata=%q",
					cfg.resourceURL, "invalid_token", "Access token no longer valid", prmURL(cfg)))
			w.Header().Set("Content-Type", "application/json")
			w.WriteHeader(http.StatusUnauthorized)
			_, _ = w.Write([]byte(`{"error": "invalid_token", "error_description": "Access token no longer valid"}`))
			return
		}
		bw.replay(w) // status, headers, body verbatim
	})
}
```

The entitlement client, the `EntitlementResult` union, the `non_authorized` rendering (the response's `result_text` string emitted verbatim as the tool's text — a single-field pipe, never parsed or re-serialized), and the unavailable-deny `CallToolResult` carry over from the wire contract in TOOLS.md — POST `{cfg.entitlementAPIOrigin}/entitlement/{op}` with the forwarded bearer and a 5-second `http.Client` timeout per attempt — ONE retry after a 2-second backoff on a transport error or a 5xx (4xx are terminal, never retried); HTTP 401 → reauth; any other non-200 or a transport failure after the retry → the deny (never fail open).

**`main.go` composition** — PRM route public, bearer gate outermost on `/mcp`, reauth buffer inside it:

```go
import (
	"github.com/modelcontextprotocol/go-sdk/auth"
	"github.com/modelcontextprotocol/go-sdk/mcp"
	"github.com/modelcontextprotocol/go-sdk/oauthex"
)

func main() {
	cfg := loadPlugpassConfig()
	server := mcp.NewServer(&mcp.Implementation{Name: "<server name>", Version: "<version>"}, nil)
	// ...mcp.AddTool registrations (below)...

	mcpHandler := mcp.NewStreamableHTTPHandler(
		func(*http.Request) *mcp.Server { return server },
		&mcp.StreamableHTTPOptions{Stateless: true, JSONResponse: true}, // required: lets a revoked-bearer tool swap in a 401
	)
	requireAuth := auth.RequireBearerToken(verifyBearer(cfg), &auth.RequireBearerTokenOptions{
		ResourceMetadataURL: prmURL(cfg),
	})

	mux := http.NewServeMux()
	mux.Handle("/.well-known/oauth-protected-resource"+mcpPath, auth.ProtectedResourceMetadataHandler(
		&oauthex.ProtectedResourceMetadata{
			Resource:               cfg.resourceURL,
			AuthorizationServers:   []string{cfg.issuer},
			BearerMethodsSupported: []string{"header"},
		}))
	mux.Handle(mcpPath, withFullChallenge(cfg, requireAuth(reauthTo401(cfg, mcpHandler))))
	log.Fatal(http.ListenAndServe(":"+cmp.Or(os.Getenv("PORT"), "8080"), mux))
}
```

**Per-request identity inside a tool**: the SDK plumbs the gate's `TokenInfo` into every request — read `req.Extra.TokenInfo.UserID` (the sub) and `req.Extra.TokenInfo.Extra["raw_token"].(string)` (the bearer to forward). The reauth flag rides the handler's `ctx` (`setReauthRequired(ctx)`).

**The check proxy tool (check host only).** A pure pipe — never parse or reformat `result_text`:

```go
const checkPremiumAccessDescription = "<verbatim from TOOLS.md>"

type checkPremiumAccessArgs struct {
	PluginID      string `json:"plugin_id" jsonschema:"the plugin's Plugpass id"`
	FeatureID     string `json:"feature_id" jsonschema:"the component's Plugpass id"`
	PluginVersion string `json:"plugin_version" jsonschema:"the installed plugin.json version"`
}

mcp.AddTool(server, &mcp.Tool{
	Name:        "{CheckToolName}",
	Title:       "Check premium access",
	Description: checkPremiumAccessDescription,
	// Take the toolHints helper: DestructiveHint and OpenWorldHint are *bool because their
	// spec defaults are TRUE, so leaving either nil publishes the pessimistic value.
	Annotations: toolHints(false, false, false, false),
}, func(ctx context.Context, req *mcp.CallToolRequest, args checkPremiumAccessArgs) (*mcp.CallToolResult, any, error) {
	bearer, _ := req.Extra.TokenInfo.Extra["raw_token"].(string)
	status, body, err := postEntitlement(ctx, cfg, bearer, "check_premium_access", args) // 5s/attempt, one 2s-backoff retry
	if err != nil || status >= 500 || status == 400 || status == 403 {
		return unavailableDeny(), nil, nil
	}
	if status == 401 || body.Status == "reauth_required" {
		setReauthRequired(ctx) // → transport-level 401 via reauthTo401
		return textResult("Re-authentication required."), nil, nil // discarded by the swap
	}
	return textResult(body.ResultText), nil, nil // verbatim — byte-identical to the native tool
})
```

**Solo paid tool wrapper** (consume-on-invocation; no `auth_token` parameter on this path): validate nothing in the handler — the gate already did — read `sub` + bearer from `req.Extra.TokenInfo`, call `track_usage` (`feature_id` = the tool's own plugpass-component-id, matching its `_meta`; the id's `tool_` prefix carries the feature type), and branch: `ok` → run the body scoped to `sub`; `non_authorized` → the `result_text` verbatim as the text result; `reauth_required` → `setReauthRequired(ctx)` + placeholder; error/timeout → the unavailable deny. Keep `_meta` via the tool's `Meta` field (`mcp.Tool{ …, Meta: mcp.Meta{"plugpass_component_id": "<plugpass_id>"} }`) and never declare an output schema on a wrapped tool (use `any` for the structured output type). Re-derive the tool's `Annotations`: metering makes it neither read-only nor idempotent, whatever it was before the wrap — `Annotations: toolHints(false, <its own destructive value>, false, <its own open-world value>)`, see TOOLS.md § Tool annotations.

**Paired-tool add side** (`operation: add`): same shape but `feature_id` = the add tool's `database_record.custom_entitlement_id` (its `custom_` prefix carries the feature type; **not** `database_record.plugpass_id`, and **not** the tool's own `_meta` id), always **`check_remaining`**, passing the user's current count from the developer's own store (scoped to `sub`) as `current_count` — see TOOLS.md → Reading the user's current count.

**Identity tool** (the paired remove side, or any per-user free tool): no Entitlement API call — read `req.Extra.TokenInfo.UserID` and scope the body to it.

**Placement guidance.** A vanilla `net/http` server follows the composition above; a server using chi/gin/echo mounts the same three pieces (public PRM route; `requireAuth(reauthTo401(mcpHandler))` on `/mcp`) through its router's `http.Handler` adapters. Invariants regardless of layout: `Stateless: true, JSONResponse: true` (both — the flag mechanism silently dies in stateful mode); verifier failures wrap `auth.ErrInvalidToken`; the gate covers every `/mcp` method. Local-dev note: the SDK 403s requests whose `Host` isn't loopback when listening on loopback (DNS-rebinding guard) — testing through a tunnel needs `DisableLocalhostProtection: true`, never in production.
