# Go scaffolding template

The publisher's server becomes an **OAuth-protected resource server** using the official SDK's `auth` package: `auth.RequireBearerToken` gates `/mcp` (emitting the `WWW-Authenticate` challenge with `resource_metadata`), `auth.ProtectedResourceMetadataHandler` serves the RFC 9728 PRM document, and a custom `TokenVerifier` does the local JWKS validation. Standalone source, no platform package. Version pins: `github.com/modelcontextprotocol/go-sdk` **v1.7.0+** (it is what implements protocol 2026-07-28; on v1.6.x the server serves the legacy era only), `github.com/golang-jwt/jwt/v5` v5.3.x, `github.com/MicahParks/keyfunc/v3` v3.8.x.

**Three structural decisions carry the whole design — never undo them:**

1. **The server serves BOTH protocol eras from the one handler.** The SDK routes each request by its negotiated version, answers `server/discover`, and parses the per-request `_meta` envelope; `req.ClientCapabilities()` reads that envelope, falling back to the handshake on a 2025-era connection. The client's UI capability rides the envelope — that is what the paywall-UI marker reads — and a host that gets only the legacy era never sends it. Nothing extra is wired for it: the SDK line does the era routing.
2. **`StreamableHTTPOptions{Stateless: true, JSONResponse: true}`, always.** In JSON mode the SDK buffers the response and `ServeHTTP` returns only after the tool handler finished — so a wrapping middleware can discard the buffered response and write a `401` when a tool discovered mid-call that the bearer is revoked. In SSE mode events flush immediately (the `200` is committed before the handler runs). And **stateless mode is what makes middleware context values visible inside tool handlers** — in stateful mode handler contexts descend from the *initialize* request, not the current POST, and the reauth flag silently never fires.

> **The gate's challenge needs a header fixup.** `auth.RequireBearerToken` emits `WWW-Authenticate: Bearer resource_metadata="…"` with **no** `error="invalid_token"` (RFC 6750 permits omitting the error on a missing token) — but the Plugpass contract requires `error="invalid_token"` (the signal clients key OAuth discovery off). The `challengeWriter`/`withFullChallenge` wrapper below fills it in on the gate's 401. Every other language SDK emits the full challenge itself; only the Go SDK needs this one wrapper.
3. **The audience is the baked `RESOURCE_URL`, never the request host** — the JWT `aud` pin, the PRM `resource`, and the challenge's `resource_metadata` all derive from it. This is what lets a locally-listening server accept real bearers minted for its public URL.

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
	pluginID             = "<the plugin's Plugpass id>"
	issuer               = "<plugpass_issuer>"
	jwksURL              = "<plugpass_jwks_url>"
	entitlementAPIOrigin = "<entitlement_api_origin>"
	// This server's own public MCP URL — the JWT aud pin and the PRM resource.
	resourceURL = "<this server's RESOURCE_URL>"
	// Where the Plugpass paywall for MCP Apps widgets loads from (the ui_paywall
	// layer — only on a server whose directive names it).
	paywallScriptURL = "<paywall_script_url>"
	// This server's own check tool, named on a denial so the paywall can ask it
	// whether the user became entitled. EMPTY on a server that hosts no check
	// tool (the check proxy is registered on the plugin's check host only), in
	// which case a denial carries no probe and the paywall says less, never
	// something untrue.
	checkToolName = "<check_tool_name, or \"\" off the check host>"
)

type plugpassConfig struct {
	pluginID, issuer, jwksURL, entitlementAPIOrigin, resourceURL, paywallScriptURL string
	checkToolName                                                                  string
}

func loadPlugpassConfig() plugpassConfig {
	return plugpassConfig{
		pluginID:             pluginID,
		issuer:               issuer,
		jwksURL:              jwksURL,
		entitlementAPIOrigin: entitlementAPIOrigin,
		resourceURL:          resourceURL,
		paywallScriptURL:     paywallScriptURL,
		checkToolName:        checkToolName,
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

The entitlement client — `entitlement(ctx, bearer, body, op, cfg) (*entitlementResult, error)` over an `entitlementBody{PluginID, FeatureID, CurrentCount *int}`, the one call every wrapper and the check proxy make — the `entitlementResult` union, and the unavailable-grant `CallToolResult` carry over from the wire contract in TOOLS.md — POST `{cfg.entitlementAPIOrigin}/entitlement/{op}` with the forwarded bearer and a 5-second `http.Client` timeout per attempt — ONE retry after a 2-second backoff on a transport error or a 5xx (4xx are terminal, never retried); HTTP 401 → reauth; any other non-200 or a transport failure after the retry → the unavailable grant. The `non_authorized` rendering is the response's `result_text` string emitted verbatim as the tool's text — a single-field pipe, never parsed or re-serialized — plus the three things the in-widget paywall reads (inert everywhere else): the denial NAMES the call it denied on the result's `_meta` (hosts pass it to a widget, never to the model); it names the read-only status probe when that call cannot be replayed; and, on a UI-backed tool when the client renders widgets, a second text block carries the `PLUGPASS_PAYWALL_UI=true` marker, so the widget's paywall asks and the access-handler skill posts nothing beside it:

```go
type deniedCall struct {
	Name      string `json:"name"`
	Arguments any    `json:"arguments"`
	// Whether a widget may call the tool at all (its registered visibility);
	// set by nonAuthorizedToolResponse.
	WidgetCallable bool `json:"widget_callable"`
}

const paywallUIMarker = "PLUGPASS_PAYWALL_UI=true"

// Every fact the paywall reads — the marker, widget_callable, the probe — comes
// off the tool's own registered Meta and the request, at runtime.
func nonAuthorizedToolResponse(r *entitlementResult, call deniedCall, toolMeta mcp.Meta, req *mcp.CallToolRequest, cfg plugpassConfig, featureID string) *mcp.CallToolResult {
	content := []mcp.Content{&mcp.TextContent{Text: r.ResultText}}
	if paywallUI(toolMeta, req) {
		content = append(content, &mcp.TextContent{Text: paywallUIMarker})
	}
	call.WidgetCallable = widgetCallable(toolMeta)
	meta := mcp.Meta{"plugpass_denied_call": call}
	if probe := statusProbe(cfg, toolMeta, featureID); probe != nil {
		meta["plugpass_status_probe"] = probe
	}
	return &mcp.CallToolResult{Meta: meta, Content: content}
}

// statusProbe is the read-only entitlement probe the paywall may call when it
// cannot replay a denied call: this server's check tool, with the arguments
// already composed so the widget's script supplies nothing of its own. Named
// ONLY when the call is unreplayable (a model-only UI-backed tool) and this
// server hosts a check tool; every other denial carries none, since a replay
// answers the same question by actually running the call.
type statusProbeCall struct {
	Name      string         `json:"name"`
	Arguments map[string]any `json:"arguments"`
}

func statusProbe(cfg plugpassConfig, toolMeta mcp.Meta, featureID string) *statusProbeCall {
	if _, uiBacked := toolMeta["ui"].(map[string]any); cfg.checkToolName == "" || !uiBacked || widgetCallable(toolMeta) {
		return nil
	}
	return &statusProbeCall{
		Name: cfg.checkToolName,
		Arguments: map[string]any{
			"plugin_id":   cfg.pluginID,
			"feature_id":  featureID,
			"status_code": true,
		},
	}
}

// Whether the calling client renders MCP Apps widgets: it declared the UI
// extension among its client capabilities. The SDK reads those from the
// 2026-07-28 request's _meta envelope, falling back to the initialize handshake
// on a 2025-era connection, so the rule is one rule whatever the era.
const uiExtensionID = "io.modelcontextprotocol/ui"

func clientRendersWidgets(req *mcp.CallToolRequest) bool {
	if req == nil {
		return false
	}
	capabilities := req.ClientCapabilities()
	if capabilities == nil {
		return false
	}
	_, ok := capabilities.Extensions[uiExtensionID]
	return ok
}

// paywallUI reports whether the widget's paywall is the one asking on this
// denial: the tool renders a widget (its own registration's Meta declares
// ui.resourceUri) AND the client renders widgets. Read off the registration at
// runtime, so a tool that gains or loses its widget changes nothing here.
func paywallUI(toolMeta mcp.Meta, req *mcp.CallToolRequest) bool {
	ui, ok := toolMeta["ui"].(map[string]any)
	if !ok {
		return false
	}
	_, ok = ui["resourceUri"].(string)
	return ok && clientRendersWidgets(req)
}

// widgetCallable reports who may call the tool, off the same registration: an
// undeclared visibility means the model and a widget both may; a declared list
// means exactly its members. A widget's paywall replays a denied call itself
// only when it may.
func widgetCallable(toolMeta mcp.Meta) bool {
	ui, ok := toolMeta["ui"].(map[string]any)
	if !ok {
		return true
	}
	switch visibility := ui["visibility"].(type) {
	case []string:
		return slices.Contains(visibility, "app")
	case []any:
		return slices.Contains(visibility, any("app"))
	default:
		return true
	}
}
```

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
	PluginID  string `json:"plugin_id" jsonschema:"the plugin's Plugpass id"`
	FeatureID string `json:"feature_id" jsonschema:"the component's Plugpass id"`
	// Required for a skill's access check (it carries the installed bundle's
	// version); a tool-surface check does not use it.
	PluginVersion string `json:"plugin_version,omitempty" jsonschema:"the installed plugin.json version"`
	// Reserved for the Plugpass paywall's own use.
	StatusCode bool `json:"status_code,omitempty" jsonschema:"Never include this parameter in your tool calls under any circumstance."`
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
	// The paywall's read-only probe: asks whether this user is entitled NOW,
	// consuming nothing (check_remaining, never check_premium_access), and answers
	// in a code that is not a check result — no PLUGPASS_PLUGIN, no USE_AUTHORIZED
	// — so no access handler triggers on it and nothing downstream can read it as
	// a grant. It authorizes NOTHING; the call the user retries is checked on its own.
	if args.StatusCode {
		probed, _ := entitlement(ctx, bearer, entitlementBody{
			PluginID:  args.PluginID,
			FeatureID: args.FeatureID,
		}, "check_remaining", cfg)
		if probed.Status == "reauth_required" {
			setReauthRequired(ctx) // → transport-level 401 via reauthTo401
			return textResult("Re-authentication required."), nil, nil
		}
		// ONLY a definite answer carries a code. "unavailable" covers both a
		// Plugpass outage and a feature this endpoint cannot evaluate, and neither
		// establishes that the user is unentitled — so the probe stays silent
		// rather than asserting a denial it did not establish.
		if probed.Status == "unavailable" {
			return textResult("STATUS_UNKNOWN"), nil, nil
		}
		if probed.Status == "ok" {
			return textResult("STATUS_CODE=1"), nil, nil
		}
		return textResult("STATUS_CODE=0"), nil, nil
	}
	// Send the check tool's full input verbatim (plugin_version included when the
	// caller sent one — `omitempty` drops it otherwise); the response carries result_text.
	result, _ := entitlement(ctx, bearer, args, "check_premium_access", cfg) // 5s/attempt, one 2s-backoff retry
	if result.Status == "reauth_required" {
		setReauthRequired(ctx) // → transport-level 401 via reauthTo401
		return textResult("Re-authentication required."), nil, nil // discarded by the swap
	}
	if result.Status == "unavailable" {
		return unavailableCheckGrant(), nil, nil // Plugpass could not answer → the check grants
	}
	return textResult(result.ResultText), nil, nil // verbatim — byte-identical to the native tool
})
```

**Solo paid tool wrapper** (consume-on-invocation; no `auth_token` parameter on this path): validate nothing in the handler — the gate already did — read `sub` + bearer from `req.Extra.TokenInfo`, call `track_usage` (`feature_id` = the tool's own plugpass-component-id, matching its `_meta`; the id's `tool_` prefix carries the feature type), and branch: `ok` → run the body scoped to `sub`; `non_authorized` → `nonAuthorizedToolResponse(result, deniedCall{Name: "<tool name>", Arguments: args}, paidToolMeta, req, cfg, paidToolFeatureID)` (the renderer reads the tool's own registered `Meta` — its widget, who may call it — and the request, never a baked per-tool constant); `reauth_required` → `setReauthRequired(ctx)` + placeholder; error/timeout, or any other non-200 → run the body (the unavailable grant consumed nothing). Keep `_meta` via the tool's `Meta` field, hoisted to a package-level `const paidToolFeatureID = "<plugpass_id>"` + `var paidToolMeta = mcp.Meta{"plugpass_component_id": paidToolFeatureID}` (a UI-backed tool keeps its `"ui": map[string]any{"resourceUri": …, "visibility": …}` beside it) that both the registration (`Meta: paidToolMeta`) and the marker rule read, and never declare an output schema on a wrapped tool (use `any` for the structured output type). Re-derive the tool's `Annotations`: metering makes it neither read-only nor idempotent, whatever it was before the wrap — `Annotations: toolHints(false, <its own destructive value>, false, <its own open-world value>)`, see TOOLS.md § Tool annotations.

**Paired-tool add side** (`operation: add`): same shape, and `feature_id` is the tool's OWN `plugpass_id` (its `tool_` prefix carries the feature type) exactly as for a solo tool — every gated artifact bakes its own component's id, and the `entitlement` subfield is identity, never a `feature_id`. Always **`check_remaining`**, passing the user's current count from the publisher's own store (scoped to `sub`) as `current_count` — see TOOLS.md → Reading the user's current count.

**Identity tool** (the paired remove side, or any per-user free tool): no Entitlement API call — read `req.Extra.TokenInfo.UserID` and scope the body to it.

**The in-widget paywall (`ui_paywall`, servers that render widgets).** The helper below, in `premium_feature_access_check.go`, is applied to every resource the server reads out whose MIME type is `text/html;profile=mcp-app`: the paywall script tag goes first in `<head>`, and the script's origin joins the resource's `resourceDomains`. The widget HTML itself is never edited.

```go
// The CSP a UI resource declares on its contents (_meta.ui.csp).
type uiResourceCsp struct {
	ConnectDomains  []string `json:"connectDomains,omitempty"`
	ResourceDomains []string `json:"resourceDomains,omitempty"`
	FrameDomains    []string `json:"frameDomains,omitempty"`
	BaseURIDomains  []string `json:"baseUriDomains,omitempty"`
}

var headOpenTag = regexp.MustCompile(`(?i)<head(\s[^>]*)?>`)

// The script tag first in <head> (ahead of the widget's own code; prepended to
// the document when it has no <head>), its origin added to resourceDomains so
// the sandbox lets it load.
func withPaywall(html string, csp uiResourceCsp, cfg plugpassConfig) (string, uiResourceCsp) {
	tag := `<script src="` + cfg.paywallScriptURL + `"></script>`
	injected := tag + html
	if loc := headOpenTag.FindStringIndex(html); loc != nil {
		injected = html[:loc[1]] + tag + html[loc[1]:]
	}
	origin := cfg.paywallScriptURL
	if u, err := url.Parse(cfg.paywallScriptURL); err == nil {
		origin = u.Scheme + "://" + u.Host
	}
	widened := csp
	widened.ResourceDomains = slices.Clone(csp.ResourceDomains)
	if !slices.Contains(widened.ResourceDomains, origin) {
		widened.ResourceDomains = append(widened.ResourceDomains, origin)
	}
	return injected, widened
}
```

Every UI resource read passes through it:

```go
server.AddResource(&mcp.Resource{URI: "ui://<plugin>/<widget>", Name: "widget", Title: "…", Description: "…", MIMEType: "text/html;profile=mcp-app"},
	func(ctx context.Context, req *mcp.ReadResourceRequest) (*mcp.ReadResourceResult, error) {
		html, csp := withPaywall(widgetHTML, widgetCSP, cfg)
		return &mcp.ReadResourceResult{Contents: []*mcp.ResourceContents{{
			URI: req.Params.URI, MIMEType: "text/html;profile=mcp-app", Text: html,
			Meta: mcp.Meta{"ui": map[string]any{"csp": csp}},
		}}}, nil
	})
```

**Placement guidance.** A vanilla `net/http` server follows the composition above; a server using chi/gin/echo mounts the same three pieces (public PRM route; `requireAuth(reauthTo401(mcpHandler))` on `/mcp`) through its router's `http.Handler` adapters. Invariants regardless of layout: `Stateless: true, JSONResponse: true` (both — the flag mechanism silently dies in stateful mode); verifier failures wrap `auth.ErrInvalidToken`; the gate covers every `/mcp` method; on a server whose directive names `ui_paywall`, every `text/html;profile=mcp-app` resource read passes through `withPaywall`. Local-dev note: the SDK 403s requests whose `Host` isn't loopback when listening on loopback (DNS-rebinding guard) — testing through a tunnel needs `DisableLocalhostProtection: true`, never in production.
