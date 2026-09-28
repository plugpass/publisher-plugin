# Go scaffolding template

The publisher's server becomes an **OAuth-protected resource server** using the official SDK's `auth` package: `auth.RequireBearerToken` gates the strict paths (emitting the `WWW-Authenticate` challenge with `resource_metadata`) — `/mcp/test` from the first deploy, `/mcp` once the plugin is published, through the publish-armed gate below — `auth.ProtectedResourceMetadataHandler` serves the RFC 9728 PRM documents for both, and a custom `TokenVerifier` does the local JWKS validation. Standalone source, no platform package. Version pins: `github.com/modelcontextprotocol/go-sdk` **v1.7.0+** (it is what implements protocol 2026-07-28; on v1.6.x the server serves the legacy era only), `github.com/golang-jwt/jwt/v5` v5.3.x, `github.com/MicahParks/keyfunc/v3` v3.8.x.

**Four structural decisions carry the whole design — never undo them:**

1. **The server serves BOTH protocol eras from the one handler.** The SDK routes each request by its negotiated version, answers `server/discover`, and parses the per-request `_meta` envelope; `req.ClientCapabilities()` reads that envelope, falling back to the handshake on a 2025-era connection. The client's UI capability rides the envelope — that is what the paywall-UI marker reads — and a host that gets only the legacy era never sends it. Nothing extra is wired for it: the SDK line does the era routing.
2. **`StreamableHTTPOptions{Stateless: true, JSONResponse: true}`, always.** In JSON mode the SDK buffers the response and `ServeHTTP` returns only after the tool handler finished — so a wrapping middleware can discard the buffered response and write a `401` when a tool discovered mid-call that the bearer is revoked. In SSE mode events flush immediately (the `200` is committed before the handler runs). And **stateless mode is what makes middleware context values visible inside tool handlers** — in stateful mode handler contexts descend from the *initialize* request, not the current POST, and the reauth flag silently never fires.

> **The gate's challenge needs a header fixup.** `auth.RequireBearerToken` emits `WWW-Authenticate: Bearer resource_metadata="…"` with **no** `error="invalid_token"` (RFC 6750 permits omitting the error on a missing token) — but the Plugpass contract requires `error="invalid_token"` (the signal clients key OAuth discovery off). The `challengeWriter`/`withFullChallenge` wrapper below fills it in on the gate's 401. Every other language SDK emits the full challenge itself; only the Go SDK needs this one wrapper.
3. **Every accepted audience comes from Plugpass, never from the request.** A bearer's `aud` must be the path's own resource — the baked `RESOURCE_URL` at `/mcp`, `{RESOURCE_URL}/test` at `/mcp/test` — or one of the retired URLs Plugpass reports for this server (the addresses it moved off, which old installs still call) in the same form. The two never cross. The PRM `resource` and the challenge's `resource_metadata` name the URL the request was addressed to (`X-Forwarded-Host`, else `Host`) only when that URL is an accepted audience, else `RESOURCE_URL` — with the `/test` suffix at the test path. This is what lets a locally-listening server accept real bearers minted for its public URL, and what keeps a moved server working for installs of its old address.
4. **The `/mcp` gate is armed by publish, and only by publish.** `enforced()` answers whether the plugin has a published version — fetched from Plugpass before the first request is handled, cached five minutes and refreshed in the background, keyed by `resourceURL` and nothing else, final once `true`, strict while there is no answer. Off, `/mcp` challenges nobody: a valid bearer takes the strict chain exactly as when on, a missing (or malformed, or expired) bearer takes the open chain with no `TokenInfo`, and every tool states what it does with none. The test path never consults it.

```go
// premium_feature_access_check.go — written once per server. Standalone.
package main

import (
	"cmp"
	"context"
	"encoding/json"
	"fmt"
	"net/http"
	"net/url"
	"os"
	"slices"
	"strings"
	"sync"
	"sync/atomic"
	"time"

	"github.com/MicahParks/keyfunc/v3"
	"github.com/golang-jwt/jwt/v5"
	"github.com/modelcontextprotocol/go-sdk/auth"
)

const mcpPath = "/mcp"

// The test path: the same handler, strictly gated from the first deploy, for a
// resource of its own — resourceURL + "/test" — that Plugpass mints only to the
// plugin's test users.
const testPathSuffix = "/test"

// Plugpass endpoints for this server.
const (
	pluginID             = "<the plugin's Plugpass id>"
	issuer               = "<plugpass_issuer>"
	jwksURL              = "<plugpass_jwks_url>"
	entitlementAPIOrigin = "<entitlement_api_origin>"
	// This server's own public MCP URL — its current bearer audience and the
	// PRM's default resource.
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

// RFC 9728 path-aware PRM URL: origin + /.well-known/oauth-protected-resource +
// the resource's path — the test document for a test resource.
func prmURL(resource string) string {
	u, _ := url.Parse(resource)
	path := mcpPath
	if strings.HasSuffix(u.Path, testPathSuffix) {
		path += testPathSuffix
	}
	return u.Scheme + "://" + u.Host + "/.well-known/oauth-protected-resource" + path
}

// The URLs this server moved off, which Plugpass reports so old installs keep
// working. Fetched only when a bearer or a request names another address;
// cached 5 minutes (1 minute after a failed fetch, which accepts nothing extra).
var (
	retiredMu      sync.Mutex
	retiredURLs    = map[string]bool{}
	retiredExpires time.Time
	retiredClient  = &http.Client{Timeout: 5 * time.Second}
)

func retiredAudiences(ctx context.Context, cfg plugpassConfig) map[string]bool {
	retiredMu.Lock()
	defer retiredMu.Unlock()
	if time.Now().Before(retiredExpires) {
		return retiredURLs
	}
	urls, ttl := map[string]bool{}, time.Minute
	req, err := http.NewRequestWithContext(ctx, http.MethodGet,
		cfg.entitlementAPIOrigin+"/entitlement/retired-audiences?resource="+url.QueryEscape(cfg.resourceURL), nil)
	if err == nil {
		if res, err := retiredClient.Do(req); err == nil {
			var body struct {
				Retired []string `json:"retired"`
			}
			if res.StatusCode == http.StatusOK && json.NewDecoder(res.Body).Decode(&body) == nil {
				for _, u := range body.Retired {
					urls[u] = true
				}
				ttl = 5 * time.Minute
			}
			res.Body.Close()
		}
	}
	retiredURLs, retiredExpires = urls, time.Now().Add(ttl)
	return retiredURLs
}

// The enforcement state — whether the plugin is published, the one input that
// turns the /mcp gate on. Fetched before the first request a process serves is
// handled (one attempt at a time under the mutex; a concurrent caller waits and
// takes its answer), cached 5 minutes and refreshed in the background after
// that; keyed by resourceURL and by nothing in any request. true is final for
// the process. A failed fetch keeps the last answer; with no answer yet the
// gate is strict, and the fetch is retried after 1 minute.
var (
	enforcementMu        sync.Mutex
	enforcementAnswer    *bool // the last answer; nil until one arrives
	enforcementExpires   time.Time
	enforcementAttemptAt time.Time
	enforcementClient    = &http.Client{Timeout: 5 * time.Second}
)

func refreshEnforcement(ctx context.Context, cfg plugpassConfig) {
	enforcementMu.Lock()
	defer enforcementMu.Unlock()
	if time.Now().Before(enforcementAttemptAt) {
		return
	}
	enforcementAttemptAt = time.Now().Add(time.Minute)
	req, err := http.NewRequestWithContext(ctx, http.MethodGet,
		cfg.entitlementAPIOrigin+"/entitlement/enforcement?resource="+url.QueryEscape(cfg.resourceURL), nil)
	if err != nil {
		return
	}
	res, err := enforcementClient.Do(req)
	if err != nil {
		return // unreachable: the last answer stands (strict while there is none) until the retry
	}
	defer res.Body.Close()
	var body struct {
		Enforced *bool `json:"enforced"`
	}
	if res.StatusCode == http.StatusOK && json.NewDecoder(res.Body).Decode(&body) == nil && body.Enforced != nil {
		enforcementAnswer, enforcementExpires = body.Enforced, time.Now().Add(5*time.Minute)
	}
}

func enforced(ctx context.Context, cfg plugpassConfig) bool {
	enforcementMu.Lock()
	answer, expires := enforcementAnswer, enforcementExpires
	enforcementMu.Unlock()
	if answer != nil && *answer {
		return true // final
	}
	if answer == nil {
		// No answer yet: learn it before handling the request; strict until it arrives.
		refreshEnforcement(ctx, cfg)
		enforcementMu.Lock()
		defer enforcementMu.Unlock()
		return enforcementAnswer == nil || *enforcementAnswer
	}
	if time.Now().After(expires) {
		// Off and stale: refresh in the background, the cached answer serving meanwhile.
		go refreshEnforcement(context.Background(), cfg)
	}
	return false
}

// The URL a request was addressed to — X-Forwarded-Host (a proxy on an old
// address sets it), else its own host, on RESOURCE_URL's scheme and path — when
// that URL is an accepted audience; otherwise RESOURCE_URL. At the test path,
// the same with the /test suffix.
func addressedResource(ctx context.Context, cfg plugpassConfig, r *http.Request, test bool) string {
	suffix := ""
	if test {
		suffix = testPathSuffix
	}
	host := r.Host
	if fwd := r.Header.Get("X-Forwarded-Host"); fwd != "" {
		host, _, _ = strings.Cut(fwd, ",")
	}
	host = strings.ToLower(strings.TrimSpace(host))
	current, _ := url.Parse(cfg.resourceURL)
	if host == "" || host == current.Host {
		return cfg.resourceURL + suffix
	}
	candidate := current.Scheme + "://" + host + current.Path
	if retiredAudiences(ctx, cfg)[candidate] {
		return candidate + suffix
	}
	return cfg.resourceURL + suffix
}

// The full 401 challenge for a resource.
func challenge(resource, description string) string {
	return fmt.Sprintf("Bearer realm=%q, error=%q, error_description=%q, resource_metadata=%q",
		resource, "invalid_token", description, prmURL(resource))
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

// The auth.TokenVerifier the bearer gate runs, one per path: EdDSA-pinned,
// issuer exact, exp required, audience = the path's own resource (the baked
// RESOURCE_URL, or its /test form at the test path) or one of this server's
// retired URLs in the same form — a canonical token is refused at the test path
// and a test token at the canonical path; extracts sub. Every token failure MUST
// wrap auth.ErrInvalidToken — anything else becomes a 500 with no
// WWW-Authenticate challenge, silently breaking client OAuth discovery.
func verifyBearer(cfg plugpassConfig, test bool) auth.TokenVerifier {
	suffix := ""
	if test {
		suffix = testPathSuffix
	}
	return func(ctx context.Context, token string, _ *http.Request) (*auth.TokenInfo, error) {
		kf, err := jwksKeyfunc(cfg.jwksURL)
		if err != nil {
			return nil, err // infra failure → 500 (correct: not a token problem)
		}
		tok, err := jwt.Parse(token, kf,
			jwt.WithValidMethods([]string{"EdDSA"}),
			jwt.WithIssuer(cfg.issuer),
			jwt.WithExpirationRequired(),
		)
		if err != nil || !tok.Valid {
			return nil, fmt.Errorf("%w: %v", auth.ErrInvalidToken, err)
		}
		auds, err := tok.Claims.GetAudience()
		if err != nil || len(auds) == 0 {
			return nil, fmt.Errorf("%w: missing aud", auth.ErrInvalidToken)
		}
		if !slices.Contains(auds, cfg.resourceURL+suffix) {
			retired := retiredAudiences(ctx, cfg)
			retiredForm := func(a string) bool {
				canonical, ok := strings.CutSuffix(a, suffix)
				return ok && retired[canonical]
			}
			if !slices.ContainsFunc(auds, retiredForm) {
				return nil, fmt.Errorf("%w: audience not accepted", auth.ErrInvalidToken)
			}
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

// The verified identity a tool call carries, or nil on a request with none (the
// /mcp gate off, before the plugin is published), which every tool handles
// explicitly: UserID is the sub, bearerOf the raw token the wrappers forward.
func identityOf(req *mcp.CallToolRequest) *auth.TokenInfo {
	if req == nil || req.Extra == nil {
		return nil
	}
	return req.Extra.TokenInfo
}

func bearerOf(info *auth.TokenInfo) string {
	if info == nil {
		return ""
	}
	bearer, _ := info.Extra["raw_token"].(string)
	return bearer
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
// error="invalid_token" — the exact signal MCP clients key OAuth discovery off —
// for the addressed resource. auth.RequireBearerToken emits only
// `resource_metadata=…` (RFC 6750 permits omitting the error on a missing
// token), so write the full challenge when the header lacks an `error=`; a
// header that already carries one (reauthTo401's) is left as-is.
type challengeWriter struct {
	http.ResponseWriter
	resource string
}

func (c *challengeWriter) WriteHeader(status int) {
	if status == http.StatusUnauthorized && !strings.Contains(c.Header().Get("WWW-Authenticate"), "error=") {
		c.Header().Set("WWW-Authenticate", challenge(c.resource, "Authentication required"))
	}
	c.ResponseWriter.WriteHeader(status)
}

type resourceKey struct{}

// Outermost: resolves the addressed resource once, for every challenge the
// request can draw.
func withFullChallenge(cfg plugpassConfig, test bool, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		resource := addressedResource(r.Context(), cfg, r, test)
		r = r.WithContext(context.WithValue(r.Context(), resourceKey{}, resource))
		next.ServeHTTP(&challengeWriter{ResponseWriter: w, resource: resource}, r)
	})
}

// The bearer a request carries, if any (the scheme compared case-insensitively).
func bearerToken(r *http.Request) (string, bool) {
	header := r.Header.Get("Authorization")
	if len(header) < 7 || !strings.EqualFold(header[:7], "bearer ") {
		return "", false
	}
	return strings.TrimSpace(header[7:]), true
}

// The gate, armed by publish: the test path is strict from the first deploy;
// /mcp challenges only while the plugin is published. Strict, the request runs
// the strict chain (auth.RequireBearerToken, which challenges or injects the
// TokenInfo); off, a valid bearer takes the same chain, and a missing, malformed,
// or expired one takes the open chain — no TokenInfo, never challenged.
func publishArmedGate(cfg plugpassConfig, test bool, strict, open http.Handler) http.Handler {
	verify := verifyBearer(cfg, test)
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if test || enforced(r.Context(), cfg) {
			strict.ServeHTTP(w, r)
			return
		}
		if token, ok := bearerToken(r); ok {
			if _, err := verify(r.Context(), token, r); err == nil {
				strict.ServeHTTP(w, r)
				return
			}
		}
		open.ServeHTTP(w, r)
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
			resource, _ := r.Context().Value(resourceKey{}).(string)
			w.Header().Set("WWW-Authenticate", challenge(cmp.Or(resource, cfg.resourceURL), "Access token no longer valid"))
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

**`main.go` composition** — both PRM routes public; per path, the challenge fixup outermost, the publish-armed gate choosing between the bearer-gated chain and the open one, the reauth buffer inside both:

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
	reauthed := reauthTo401(cfg, mcpHandler)

	mux := http.NewServeMux()
	for _, m := range []struct {
		path string
		test bool
	}{{mcpPath, false}, {mcpPath + testPathSuffix, true}} {
		suffix := ""
		if m.test {
			suffix = testPathSuffix
		}
		mux.HandleFunc("/.well-known/oauth-protected-resource"+m.path, func(w http.ResponseWriter, r *http.Request) {
			auth.ProtectedResourceMetadataHandler(&oauthex.ProtectedResourceMetadata{
				Resource:               addressedResource(r.Context(), cfg, r, m.test),
				AuthorizationServers:   []string{cfg.issuer},
				BearerMethodsSupported: []string{"header"},
			}).ServeHTTP(w, r)
		})
		requireAuth := auth.RequireBearerToken(verifyBearer(cfg, m.test), &auth.RequireBearerTokenOptions{
			ResourceMetadataURL: prmURL(cfg.resourceURL + suffix), // withFullChallenge rewrites it per request
		})
		mux.Handle(m.path, withFullChallenge(cfg, m.test, publishArmedGate(cfg, m.test, requireAuth(reauthed), reauthed)))
	}
	log.Fatal(http.ListenAndServe(":"+cmp.Or(os.Getenv("PORT"), "8080"), mux))
}
```

**Per-request identity inside a tool**: the SDK plumbs the gate's `TokenInfo` into every request — read `info := identityOf(req)`: `info.UserID` is the sub, `bearerOf(info)` the bearer to forward, and `nil` a request with no identity (the /mcp gate off, before the plugin is published), which every tool handles explicitly. The reauth flag rides the handler's `ctx` (`setReauthRequired(ctx)`).

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
	info := identityOf(req)
	// No identity — the /mcp gate off, before the plugin is published: the one
	// answer the proxy composes itself, without reaching Plugpass.
	if info == nil {
		return unavailableCheckGrant(), nil, nil
	}
	bearer := bearerOf(info)
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

**Solo paid tool wrapper** (consume-on-invocation; no `auth_token` parameter on this path): validate nothing in the handler — the gate already did — read `info := identityOf(req)`; when it is `nil` (no bearer — the /mcp gate off, the plugin unpublished) run the body with no entitlement call, as the tool ran before Plugpass; otherwise take `sub` (`info.UserID`) and the bearer (`bearerOf(info)`), call `track_usage` (`feature_id` = the tool's own plugpass-component-id, matching its `_meta`; the id's `tool_` prefix carries the feature type), and branch: `ok` → run the body scoped to `sub`; `non_authorized` → `nonAuthorizedToolResponse(result, deniedCall{Name: "<tool name>", Arguments: args}, paidToolMeta, req, cfg, paidToolFeatureID)` (the renderer reads the tool's own registered `Meta` — its widget, who may call it — and the request, never a baked per-tool constant); `reauth_required` → `setReauthRequired(ctx)` + placeholder; error/timeout, or any other non-200 → run the body (the unavailable grant consumed nothing). Keep `_meta` via the tool's `Meta` field, hoisted to a package-level `const paidToolFeatureID = "<plugpass_id>"` + `var paidToolMeta = mcp.Meta{"plugpass_component_id": paidToolFeatureID}` (a UI-backed tool keeps its `"ui": map[string]any{"resourceUri": …, "visibility": …}` beside it) that both the registration (`Meta: paidToolMeta`) and the marker rule read, and never declare an output schema on a wrapped tool (use `any` for the structured output type). Re-derive the tool's `Annotations`: metering makes it neither read-only nor idempotent, whatever it was before the wrap — `Annotations: toolHints(false, <its own destructive value>, false, <its own open-world value>)`, see TOOLS.md § Tool annotations.

**Paired-tool add side** (`operation: add`): same shape, and `feature_id` is the tool's OWN `plugpass_id` (its `tool_` prefix carries the feature type) exactly as for a solo tool — every gated artifact bakes its own component's id, and the `entitlement` subfield is identity, never a `feature_id`. Always **`check_remaining`**, passing the user's current count from the publisher's own store (scoped to `sub`) as `current_count` — see TOOLS.md → Reading the user's current count.

**Identity tool** (the paired remove side, or any per-user free tool): no Entitlement API call — read `identityOf(req)` and scope the body to its `UserID`; when it is `nil` (the /mcp gate off), answer that nobody is signed in and touch no record:

```go
info := identityOf(req)
if info == nil {
	return textResult("No user is signed in."), nil, nil
}
// …existing tool body, scoped to info.UserID…
```

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

**Placement guidance.** A vanilla `net/http` server follows the composition above; a server using chi/gin/echo mounts the same pieces (the public PRM routes; per path, `withFullChallenge(cfg, test, publishArmedGate(cfg, test, requireAuth(reauthed), reauthed))`) through its router's `http.Handler` adapters. Invariants regardless of layout: `Stateless: true, JSONResponse: true` (both — the flag mechanism silently dies in stateful mode); verifier failures wrap `auth.ErrInvalidToken`; the handler is mounted at `/mcp` and `/mcp/test`, each behind `withFullChallenge` and `publishArmedGate` covering every method — strict at the test path, armed by publish at `/mcp`, never challenging while off; both PRM routes public; on a server whose directive names `ui_paywall`, every `text/html;profile=mcp-app` resource read passes through `withPaywall`. Local-dev note: the SDK 403s requests whose `Host` isn't loopback when listening on loopback (DNS-rebinding guard) — testing through a tunnel needs `DisableLocalhostProtection: true`, never in production.
