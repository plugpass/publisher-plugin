# JVM (Java/Kotlin) scaffolding template

The publisher's server becomes an **OAuth-protected resource server**: a servlet `Filter` gates `/mcp` (validating the bearer against Plugpass's JWKS with Nimbus + the JDK's native Ed25519), a tiny servlet serves the RFC 9728 PRM document, and the MCP servlet transport runs behind them under embedded Jetty. Standalone source, no platform package. Version pins: `io.modelcontextprotocol.sdk:mcp` **2.0.0** (the aggregate — `mcp-core` + Jackson 3; apps pinned to Jackson 2 use `mcp-core` + `mcp-json-jackson2`), `com.nimbusds:nimbus-jose-jwt` 10.9.x, `org.eclipse.jetty.ee10:jetty-ee10-servlet` 12.1.x. Java 17+ (the reference targets 21). Keep the shade plugin's `ServicesResourceTransformer` — the SDK discovers its JSON mapper via ServiceLoader.

**Three structural decisions carry the whole design — never undo them:**

1. **Use `HttpServletStatelessServerTransport`** (not the streamable provider). It answers `tools/call` with a plain `application/json` body, synchronously on the request thread, no `startAsync`, no sessions — so the auth filter's buffering wrapper holds the complete response after `chain.doFilter` returns and can swap in a `401` when a tool discovered mid-call that the bearer is revoked. (The streamable provider answers tools/call over SSE; a buffering filter still works there only because of deferred-`complete()` servlet semantics, with mid-stream-notification caveats — stay stateless unless the publisher's tools need sampling/elicitation.) GET returns 405 — spec-legal for streamable-HTTP servers.
2. **Verify Ed25519 with the JDK (`Signature.getInstance("Ed25519")`), not Nimbus's `Ed25519Verifier`** — the Nimbus verifier still requires the optional Google Tink dependency, and its stock `DefaultJWSVerifierFactory` has no EdDSA support at all. Nimbus handles JWKS fetching/caching/selection and claims verification; the signature check is ~10 lines of JDK crypto.
3. **The audience is the baked `RESOURCE_URL`, never the request host** — the JWT `aud` pin, the PRM `resource`, and the challenge's `resource_metadata` all derive from it. This is what lets a locally-listening server accept real bearers minted for its public URL.

```java
// PremiumFeatureAccessCheck.java — written once per server. Standalone.
public final class PremiumFeatureAccessCheck {
  public static final String MCP_PATH = "/mcp";

  // Plugpass endpoints for this server.
  private static final String ISSUER = "<plugpass_issuer>";
  private static final String JWKS_URL = "<plugpass_jwks_url>";
  private static final String ENTITLEMENT_API_ORIGIN = "<entitlement_api_origin>";
  // This server's own public MCP URL — the JWT aud pin and the PRM resource.
  private static final String RESOURCE_URL = "<this server's RESOURCE_URL>";
  // Where the Plugpass paywall for MCP Apps widgets loads from (the `ui_paywall`
  // layer — only on a server whose directive names it).
  private static final String PAYWALL_SCRIPT_URL = "<paywall_script_url>";

  public static String issuer() { return ISSUER; }
  public static String jwksUrl() { return JWKS_URL; }
  public static String entitlementApiOrigin() { return ENTITLEMENT_API_ORIGIN; }
  public static String resourceUrl() { return RESOURCE_URL; }
  public static String paywallScriptUrl() { return PAYWALL_SCRIPT_URL; }

  // RFC 9728 path-aware PRM URL: origin + /.well-known/oauth-protected-resource + /mcp.
  public static String prmUrl() {
    String origin = resourceUrl().substring(0, resourceUrl().length() - MCP_PATH.length());
    return origin + "/.well-known/oauth-protected-resource" + MCP_PATH;
  }

  // Remote JWKS: 4h cache, rate-limited refetch (an unknown kid re-fetches
  // structurally — selection misses trigger a rate-limited refresh), no
  // refresh-ahead executors to manage.
  private static final JWKSource<SecurityContext> JWKS = JWKSourceBuilder
      .create(url(JWKS_URL))
      .cache(Duration.ofHours(4).toMillis(), Duration.ofSeconds(15).toMillis())
      .refreshAheadCache(false)
      .rateLimited(30_000L)
      .outageTolerant(Duration.ofHours(24).toMillis())
      .build();

  /**
   * EdDSA-pinned, issuer exact, audience = the baked RESOURCE_URL, exp + sub
   * required. Nimbus selects the OKP key by kid; the JDK verifies Ed25519
   * (raw x wrapped in the fixed 12-byte SPKI prefix). Returns sub, or null.
   */
  public static String verifyBearer(String token) {
    try {
      SignedJWT jwt = SignedJWT.parse(token);
      if (!JWSAlgorithm.EdDSA.equals(jwt.getHeader().getAlgorithm())) return null; // alg pinned
      JWKSelector selector = new JWKSelector(new JWKMatcher.Builder()
          .keyType(KeyType.OKP).curves(Curve.Ed25519)
          .keyID(jwt.getHeader().getKeyID()).build());
      List<JWK> keys = JWKS.get(selector, null);
      if (keys.isEmpty()) return null;
      OctetKeyPair okp = keys.get(0).toOctetKeyPair();

      byte[] spki = concat(ED25519_SPKI_PREFIX, okp.getDecodedX()); // {0x30,0x2a,0x30,0x05,0x06,0x03,0x2b,0x65,0x70,0x03,0x21,0x00}
      PublicKey publicKey = KeyFactory.getInstance("Ed25519").generatePublic(new X509EncodedKeySpec(spki));
      Signature sig = Signature.getInstance("Ed25519");
      sig.initVerify(publicKey);
      sig.update(jwt.getSigningInput());
      if (!sig.verify(jwt.getSignature().decode())) return null;

      // Claims: iss exact, aud contains RESOURCE_URL, exp + sub required
      // (60s default clock skew).
      new DefaultJWTClaimsVerifier<SecurityContext>(
          Set.of(resourceUrl()),
          new JWTClaimsSet.Builder().issuer(issuer()).build(),
          Set.of("sub", "exp"),
          null
      ).verify(jwt.getJWTClaimsSet(), null);
      return jwt.getJWTClaimsSet().getSubject();
    } catch (Exception e) {
      return null;
    }
  }

  /** Per-request holder the filter creates, the context extractor forwards,
   *  and tool handlers read. The AtomicBoolean is the reauth back-channel. */
  public record PlugpassAuth(String sub, String bearer, AtomicBoolean reauthRequired) {}
  public static final String AUTH_ATTRIBUTE = "plugpass.auth";

  // Entitlement API client: java.net.http.HttpClient, 5s connect + request
  // timeouts PER ATTEMPT with ONE retry after a 2s backoff on a transport
  // error or a 5xx (4xx terminal, never retried), Authorization: Bearer
  // <token>. HTTP 401 → reauth_required; every other failure → status
  // "unavailable", which GRANTS: the call is server-to-server from this host,
  // so the end user cannot have caused it.
  /* …entitlement(bearer, bodyJson, op), UNAVAILABLE_CHECK_GRANT… */

  /** The call a wrapper denied: the tool's name and the arguments it was called with. */
  public record DeniedCall(String name, Map<String, Object> arguments) {}

  /** The marker a UI-backed tool's denial carries when the client renders widgets. */
  public static final String PAYWALL_UI_MARKER = "PLUGPASS_PAYWALL_UI=true";

  // non_authorized → result_text verbatim — a single-field pipe (the trigger keys
  // inside it auto-fire the plugin's access-handler skill). Never parse, reformat,
  // or re-serialize it.
  //
  // Two things ride beside it for the Plugpass paywall an MCP Apps widget shows
  // (both inert everywhere else):
  //   - The denial NAMES the call it denied, on the result's `_meta` — which hosts
  //     pass to a widget and never show the model — so the paywall can replay the
  //     same call once the user has upgraded, and says whether a widget may call
  //     the tool at all (widgetCallable below): a host refuses a widget's call to
  //     a model-only tool, so the paywall then hands the retry to the conversation.
  //   - The marker: when the denied tool is UI-backed (it renders a widget) AND the
  //     client renders widgets (paywallUi below), a second text block carries
  //     PLUGPASS_PAYWALL_UI=true. The widget's paywall is then the one asking the
  //     user, and the plugin's access-handler skill posts nothing beside it. The
  //     composed text stays untouched in its own block.
  // Both read the tool's own registered meta and the request, at runtime.
  public static CallToolResult nonAuthorizedToolResponse(Map<String, Object> result, DeniedCall call, Map<String, Object> toolMeta, CallToolRequest request) {
    var response = CallToolResult.builder().addTextContent(String.valueOf(result.get("result_text")));
    if (paywallUi(toolMeta, request)) response.addTextContent(PAYWALL_UI_MARKER);
    Map<String, Object> deniedCall = new LinkedHashMap<>();
    deniedCall.put("name", call.name());
    deniedCall.put("arguments", call.arguments() == null ? Map.of() : call.arguments());
    deniedCall.put("widget_callable", widgetCallable(toolMeta));
    return response.meta(Map.of("plugpass_denied_call", deniedCall)).build();
  }

  // Whether the calling client renders MCP Apps widgets: it declared the UI
  // extension among the client capabilities every request carries in its `_meta`
  // (`io.modelcontextprotocol/clientCapabilities` — the stateless protocol's
  // per-request declaration; this server keeps no session, so the initialize
  // handshake is not a source). Read off the tool call request's meta(). A client
  // that declares nothing renders nothing, and gets no marker.
  private static final String CLIENT_CAPABILITIES_META_KEY = "io.modelcontextprotocol/clientCapabilities";
  private static final String UI_EXTENSION_ID = "io.modelcontextprotocol/ui";

  public static boolean clientRendersWidgets(CallToolRequest request) {
    Map<String, Object> meta = request.meta();
    if (meta == null) return false;
    return meta.get(CLIENT_CAPABILITIES_META_KEY) instanceof Map<?, ?> capabilities
        && capabilities.get("extensions") instanceof Map<?, ?> extensions
        && extensions.get(UI_EXTENSION_ID) instanceof Map<?, ?>;
  }

  // Whether the widget's paywall is the one asking on this denial: the tool renders
  // a widget (its own registration's meta declares ui.resourceUri) AND the client
  // renders widgets. Read off the registration at runtime, so a tool that gains or
  // loses its widget changes nothing here.
  public static boolean paywallUi(Map<String, Object> toolMeta, CallToolRequest request) {
    return toolMeta.get("ui") instanceof Map<?, ?> ui
        && ui.get("resourceUri") instanceof String
        && clientRendersWidgets(request);
  }

  // Who may call the tool, off the same registration: an undeclared visibility
  // means the model and a widget both may; a declared list means exactly its
  // members. A widget's paywall replays a denied call itself only when it may.
  public static boolean widgetCallable(Map<String, Object> toolMeta) {
    return !(toolMeta.get("ui") instanceof Map<?, ?> ui)
        || !(ui.get("visibility") instanceof List<?> visibility)
        || visibility.contains("app");
  }
}
```

**`BearerAuthFilter`** — the gate plus the reauth swap (buffer POST bodies only):

```java
public final class BearerAuthFilter extends HttpFilter {
  @Override
  protected void doFilter(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
      throws IOException, ServletException {
    String header = req.getHeader("Authorization");
    String token = header != null && header.regionMatches(true, 0, "Bearer ", 0, 7) ? header.substring(7).trim() : null;
    if (token == null) { challenge(res, "Missing bearer token"); return; }
    String sub = PremiumFeatureAccessCheck.verifyBearer(token);
    if (sub == null) { challenge(res, "Token invalid or expired"); return; }

    var auth = new PremiumFeatureAccessCheck.PlugpassAuth(sub, token, new AtomicBoolean(false));
    req.setAttribute(PremiumFeatureAccessCheck.AUTH_ATTRIBUTE, auth);

    BufferingResponseWrapper buffer = new BufferingResponseWrapper(res); // overrides getWriter/getOutputStream only
    chain.doFilter(req, buffer);                                        // stateless transport: fully synchronous
    if (auth.reauthRequired().get() && !res.isCommitted()) {
      challenge(res, "Access token no longer valid");                   // → the client re-authorizes and retries
      return;
    }
    buffer.replayTo(res); // status/headers passed through; body copied verbatim
  }

  // error="invalid_token" exactly — the signal MCP clients treat as "needs OAuth".
  private static void challenge(HttpServletResponse res, String description) throws IOException {
    res.setStatus(401);
    res.setHeader("WWW-Authenticate", "Bearer realm=\"" + PremiumFeatureAccessCheck.resourceUrl()
        + "\", error=\"invalid_token\", error_description=\"" + description
        + "\", resource_metadata=\"" + PremiumFeatureAccessCheck.prmUrl() + "\"");
    res.setContentType("application/json");
    res.getWriter().write("{\"error\": \"invalid_token\", \"error_description\": \"" + description + "\"}");
  }
}
```

**`Main`** — stateless transport + context extractor + Jetty composition:

```java
var transport = HttpServletStatelessServerTransport.builder()
    .mcpEndpoint(PremiumFeatureAccessCheck.MCP_PATH)
    .contextExtractor(request -> McpTransportContext.create(
        Map.of(PremiumFeatureAccessCheck.AUTH_ATTRIBUTE,
               request.getAttribute(PremiumFeatureAccessCheck.AUTH_ATTRIBUTE))))
    .build();

McpServer.sync(transport)
    .serverInfo("<server name>", "<version>")
    .capabilities(ServerCapabilities.builder().tools(true).build())
    .tools(/* …McpStatelessServerFeatures.SyncToolSpecification list, incl. the check proxy… */)
    .build();

Server jetty = new Server(new InetSocketAddress("0.0.0.0", port));
ServletContextHandler handler = new ServletContextHandler();
handler.setContextPath("/");
handler.addServlet(new ServletHolder(transport), PremiumFeatureAccessCheck.MCP_PATH);
handler.addServlet(new ServletHolder(new PrmServlet()), "/.well-known/oauth-protected-resource/mcp");
FilterHolder auth = new FilterHolder(new BearerAuthFilter());
auth.setAsyncSupported(true);
handler.addFilter(auth, PremiumFeatureAccessCheck.MCP_PATH, EnumSet.of(DispatcherType.REQUEST)); // /mcp ONLY — never the PRM path
jetty.setHandler(handler);
jetty.start();
jetty.join();
```

`PrmServlet` is a trivial `doGet` writing `{"resource": resourceUrl(), "authorization_servers": [issuer()], "bearer_methods_supported": ["header"]}` as `application/json` — unauthenticated.

**Per-request identity inside a tool**: stateless handlers receive the transport context directly — `BiFunction<McpTransportContext, CallToolRequest, CallToolResult>`; read `var auth = (PlugpassAuth) ctx.get(PremiumFeatureAccessCheck.AUTH_ATTRIBUTE)`. On reauth: `auth.reauthRequired().set(true)`.

**The check proxy tool (check host only).** A pure pipe — never parse or reformat `result_text`. SDK 2.0 builds tool input schemas as **`Map<String, Object>` that must be valid JSON Schema 2020-12** (meta-validated at `build()`), and input validation runs by default:

```java
static final String CHECK_PREMIUM_ACCESS_DESCRIPTION = "<the VERBATIM description from TOOLS.md § The check proxy tool>";

// The ToolAnnotations every tool declares. Set all
// four: an omitted hint falls back to the spec's pessimistic default, so a
// read-only tool that leaves them out is shown as destructive and open-world.
private static McpSchema.ToolAnnotations hints(
    boolean readOnly, boolean destructive, boolean idempotent, boolean openWorld) {
  return McpSchema.ToolAnnotations.builder()
      .readOnlyHint(readOnly)
      .destructiveHint(destructive)
      .idempotentHint(idempotent)
      .openWorldHint(openWorld)
      .build();
}

var checkPremiumAccess = McpStatelessServerFeatures.SyncToolSpecification.builder()
    .tool(Tool.builder("{CheckToolName}", Map.of(
            "type", "object",
            "properties", Map.of(
                "plugin_id", Map.of("type", "string"),
                "feature_id", Map.of("type", "string"),
                "plugin_version", Map.of("type", "string")),
            "required", List.of("plugin_id", "feature_id", "plugin_version")))
        .title("Check premium access")
        .description(CHECK_PREMIUM_ACCESS_DESCRIPTION)
        .annotations(hints(false, false, false, false))
        .build()) // no outputSchema
    .callHandler((ctx, request) -> {
      var auth = (PremiumFeatureAccessCheck.PlugpassAuth) ctx.get(PremiumFeatureAccessCheck.AUTH_ATTRIBUTE);
      if (auth == null) return unavailableCheckGrant(); // unreachable behind the gate
      var result = PremiumFeatureAccessCheck.entitlement(auth.bearer(),
          toJson(request.arguments()), "check_premium_access");
      if ("reauth_required".equals(result.status())) {
        auth.reauthRequired().set(true); // → transport-level 401 via the filter
        return CallToolResult.builder().addTextContent("Re-authentication required.").build(); // discarded by the swap
      }
      if ("unavailable".equals(result.status())) {
        return unavailableCheckGrant(); // Plugpass could not answer → the check grants
      }
      return CallToolResult.builder().addTextContent(result.resultText()).build(); // verbatim — byte-identical to the native tool
    })
    .build();
```

**Solo paid tool wrapper** (consume-on-invocation; no `auth_token` parameter on this path): read the holder from the context (`auth.sub()` scopes the body, `auth.bearer()` is the forwarded bearer), call `track_usage` (`feature_id` = the tool's own plugpass-component-id, matching its `_meta`; the id's `tool_` prefix carries the feature type), and branch: `ok` → run the body; `non_authorized` → `nonAuthorizedToolResponse(result, new DeniedCall("<tool name>", request.arguments()), PAID_TOOL_META, request)` (the renderer reads the tool's own registered meta — its widget, who may call it — and the request, never a baked per-tool constant); `reauth_required` → set the flag + placeholder; `unavailable` → run the body (it consumed nothing and grants). Stamp `_meta` via `Tool.builder(...).meta(PAID_TOOL_META)`, the map hoisted to a `static final Map<String, Object> PAID_TOOL_META = Map.of("plugpass_component_id", "<plugpass_id>")` (a UI-backed tool's also carries its `"ui", Map.of("resourceUri", …, "visibility", …)`) that both the builder and the marker rule read, and never declare an `outputSchema` on a wrapped tool (paywall/reauth responses are text-only). Re-derive the tool's `.annotations(…)`: metering makes it neither read-only nor idempotent, whatever it was before the wrap — `.annotations(hints(false, <its own destructive value>, false, <its own open-world value>))`, see TOOLS.md § Tool annotations.

**Paired-tool add side** (`operation: add`): same shape but `feature_id` = the add tool's `database_record.custom_entitlement_id` (its `custom_` prefix carries the feature type; **not** `database_record.plugpass_id`, and **not** the tool's own `_meta` id), always **`check_remaining`**, passing the user's current count from the publisher's own store (scoped to `sub`) as `current_count` — see TOOLS.md → Reading the user's current count.

**Identity tool** (the paired remove side, or any per-user free tool): no Entitlement API call — read `auth.sub()` from the context and scope the body to it.

**The in-widget paywall (`ui_paywall`, servers that render widgets).** The helper below, in `PremiumFeatureAccessCheck`, is applied to every resource the server reads out whose MIME type is `text/html;profile=mcp-app`: the paywall script tag goes first in `<head>`, and the script's origin joins the resource's `resourceDomains`. The widget HTML itself is never edited.

```java
/** The CSP a UI resource declares on its contents (_meta.ui.csp); toMap() is the wire shape. */
public record UiResourceCsp(List<String> connectDomains, List<String> resourceDomains, List<String> frameDomains, List<String> baseUriDomains) {
  public Map<String, Object> toMap() { /* camelCase keys; empty lists omitted */ }
}

public record PaywalledWidget(String html, UiResourceCsp csp) {}

private static final Pattern HEAD_OPEN = Pattern.compile("<head(\\s[^>]*)?>", Pattern.CASE_INSENSITIVE);

// Loads the Plugpass paywall into a widget's HTML on its way out: the script
// tag first in <head> (ahead of the widget's own code; prepended to the
// document when it has no <head>), its origin added to resourceDomains so the
// sandbox lets it load.
public static PaywalledWidget withPaywall(String html, UiResourceCsp csp) {
  String tag = "<script src=\"" + paywallScriptUrl() + "\"></script>";
  Matcher head = HEAD_OPEN.matcher(html);
  String injected = head.find() ? html.substring(0, head.end()) + tag + html.substring(head.end()) : tag + html;
  String origin = originOf(paywallScriptUrl()); // scheme + host (+ port), no path
  List<String> resourceDomains = new ArrayList<>(csp.resourceDomains() == null ? List.of() : csp.resourceDomains());
  if (!resourceDomains.contains(origin)) resourceDomains.add(origin);
  return new PaywalledWidget(injected, new UiResourceCsp(csp.connectDomains(), resourceDomains, csp.frameDomains(), csp.baseUriDomains()));
}
```

Every UI resource read passes through it — the resource is registered with `.resources(...)` on the server builder, whose capabilities add `.resources(false, false)`:

```java
new McpStatelessServerFeatures.SyncResourceSpecification(
    McpSchema.Resource.builder().uri("ui://<plugin>/<widget>").name("<widget>").title("…").description("…").mimeType("text/html;profile=mcp-app").build(),
    (ctx, request) -> {
      var widget = PremiumFeatureAccessCheck.withPaywall(WIDGET_HTML, WIDGET_CSP);
      return new McpSchema.ReadResourceResult(List.of(
          McpSchema.TextResourceContents.builder(request.uri(), widget.html())
              .mimeType("text/html;profile=mcp-app")
              .meta(Map.of("ui", Map.of("csp", widget.csp().toMap())))
              .build()));
    })
```

**Placement guidance.** An embedded-Jetty server follows the composition above; a servlet-container deployment registers the same three pieces in its `web.xml`/programmatic config (filter on `/mcp` only, `asyncSupported` true, PRM servlet public). On a server whose directive names `ui_paywall`, every `text/html;profile=mcp-app` resource read passes through `withPaywall`. If the publisher's server must stay on the stateful `HttpServletStreamableServerTransportProvider` (tools that use sampling/elicitation), the identity path changes to `exchange.transportContext()` on `McpSyncServerExchange` and the same buffering filter works via the servlet's deferred-`complete()` semantics — but never wrap the GET listening stream, leave `keepAliveInterval` unset, and know that mid-call notifications are delayed until completion. Migration notes for a publisher's older SDK: 1.x `McpSchema.JsonSchema` is deprecated (bridge maps in), and 2.0 validates tool inputs by default.
