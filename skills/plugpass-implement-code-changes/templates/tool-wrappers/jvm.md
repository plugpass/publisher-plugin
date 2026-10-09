# JVM (Java/Kotlin) scaffolding template

The publisher's server becomes an **OAuth-protected resource server**: a servlet `Filter` gates `/mcp` and `/mcp/test` (validating the bearer against Plugpass's JWKS with Nimbus + the JDK's native Ed25519 — `/mcp/test` strict from the first deploy, `/mcp` once the plugin is published), a tiny servlet serves the RFC 9728 PRM documents for both, and the MCP servlet transport runs behind them under embedded Jetty. Standalone source, no platform package. Version pins: `io.modelcontextprotocol.sdk:mcp` **2.0.1** (the aggregate — `mcp-core` + Jackson 3; apps pinned to Jackson 2 use `mcp-core` + `mcp-json-jackson2`), `com.nimbusds:nimbus-jose-jwt` 10.9.x, `org.eclipse.jetty.ee10:jetty-ee10-servlet` 12.1.x. Java 17+ (the reference targets 21). Keep the shade plugin's `ServicesResourceTransformer` — the SDK discovers its JSON mapper via ServiceLoader.

> **This SDK line serves the 2025 protocol era only.** `io.modelcontextprotocol.sdk:mcp` implements nothing past 2025-11-25: there is no `server/discover` and no per-request `_meta` envelope, so a 2026-07-28 host opens with discover, takes the 404, and falls back. Everything else in this template holds unchanged — including the status probe, which is an ordinary tool parameter and needs no protocol support. **The one behavioral difference: a widget denial on a modern host carries no paywall-UI marker, so the chat asks beside the modal there.**

**Four structural decisions carry the whole design — never undo them:**

1. **Use `HttpServletStatelessServerTransport`** (not the streamable provider). It answers `tools/call` with a plain `application/json` body, synchronously on the request thread, no `startAsync`, no sessions — so the auth filter's buffering wrapper holds the complete response after `chain.doFilter` returns and can swap in a `401` when a tool discovered mid-call that the bearer is revoked. (The streamable provider answers tools/call over SSE; a buffering filter still works there only because of deferred-`complete()` servlet semantics, with mid-stream-notification caveats — stay stateless unless the publisher's tools need sampling/elicitation.) GET returns 405 — spec-legal for streamable-HTTP servers.
2. **Verify Ed25519 with the JDK (`Signature.getInstance("Ed25519")`), not Nimbus's `Ed25519Verifier`** — the Nimbus verifier still requires the optional Google Tink dependency, and its stock `DefaultJWSVerifierFactory` has no EdDSA support at all. Nimbus handles JWKS fetching/caching/selection and claims verification; the signature check is ~10 lines of JDK crypto.
3. **Every accepted audience comes from Plugpass, never from the request.** A bearer's `aud` must be the path's own resource — the baked `RESOURCE_URL` at `/mcp`, `{RESOURCE_URL}/test` at `/mcp/test` — or one of the retired URLs Plugpass reports for this server (the addresses it moved off, which old installs still call) in the same form. The two never cross. The PRM `resource` and the challenge's `resource_metadata` name the URL the request was addressed to (`X-Forwarded-Host`, else `Host`) only when that URL is an accepted audience, else `RESOURCE_URL` — with the `/test` suffix at the test path. This is what lets a locally-listening server accept real bearers minted for its public URL, and what keeps a moved server working for installs of its old address.
4. **The `/mcp` gate is armed by publish, and only by publish.** `enforced()` answers whether the plugin has a published version — fetched from Plugpass before the first request is handled, cached five minutes and refreshed in the background, keyed by `RESOURCE_URL` and nothing else, final once `true`, strict while there is no answer. Off, `/mcp` challenges nobody: a valid bearer is handled as when on, a missing (or malformed, or expired) bearer means no `PlugpassAuth` on the request, and every tool states what it does with none. The test path never consults it.

```java
// PremiumFeatureAccessCheck.java — written once per server. Standalone.
public final class PremiumFeatureAccessCheck {
  public static final String MCP_PATH = "/mcp";
  // The test path: the same transport, strictly gated from the first deploy, for
  // a resource of its own — RESOURCE_URL + "/test" — that Plugpass mints only to
  // the plugin's test users.
  public static final String TEST_PATH_SUFFIX = "/test";
  public static final String TEST_MCP_PATH = MCP_PATH + TEST_PATH_SUFFIX;

  // Plugpass endpoints for this server.
  private static final String PLUGIN_ID = "<the plugin's Plugpass id>";
  private static final String ISSUER = "<plugpass_issuer>";
  private static final String JWKS_URL = "<plugpass_jwks_url>";
  private static final String ENTITLEMENT_API_ORIGIN = "<entitlement_api_origin>";
  // This server's own public MCP URL — its current bearer audience and the PRM's
  // default resource.
  private static final String RESOURCE_URL = "<this server's RESOURCE_URL>";
  // Where the Plugpass paywall for MCP Apps widgets loads from (the `ui_paywall`
  // layer — only on a server whose directive names it).
  private static final String PAYWALL_SCRIPT_URL = "<paywall_script_url>";
  // This server's own check tool, named on a denial so the paywall can ask it
  // whether the user became entitled. null on a server that hosts no check tool
  // (the check proxy is registered on the plugin's check host only), in which case
  // a denial carries no probe and the paywall says less, never something untrue.
  private static final String CHECK_TOOL_NAME = "<check_tool_name, or null off the check host>";

  public static String pluginId() { return PLUGIN_ID; }
  public static String issuer() { return ISSUER; }
  public static String jwksUrl() { return JWKS_URL; }
  public static String entitlementApiOrigin() { return ENTITLEMENT_API_ORIGIN; }
  public static String resourceUrl() { return RESOURCE_URL; }
  public static String paywallScriptUrl() { return PAYWALL_SCRIPT_URL; }
  public static String checkToolName() { return CHECK_TOOL_NAME; }

  // RFC 9728 path-aware PRM URL: origin + /.well-known/oauth-protected-resource +
  // the resource's path — the test document for a test resource.
  public static String prmUrl(String resource) {
    String path = resource.endsWith(TEST_MCP_PATH) ? TEST_MCP_PATH : MCP_PATH;
    String origin = resource.substring(0, resource.length() - path.length());
    return origin + "/.well-known/oauth-protected-resource" + path;
  }

  // The URLs this server moved off, which Plugpass reports so old installs keep
  // working. Fetched only when a bearer or a request names another address;
  // cached 5 minutes (1 minute after a failed fetch, which accepts nothing extra).
  private static final HttpClient RETIRED_HTTP =
      HttpClient.newBuilder().connectTimeout(Duration.ofSeconds(5)).build();
  private static Set<String> retired = Set.of();
  private static long retiredExpiresAt = System.nanoTime();

  public static synchronized Set<String> retiredAudiences() {
    long now = System.nanoTime();
    if (now - retiredExpiresAt < 0) return retired;
    Set<String> urls = Set.of();
    Duration ttl = Duration.ofMinutes(1);
    try {
      HttpRequest request = HttpRequest.newBuilder(URI.create(entitlementApiOrigin()
              + "/entitlement/retired-audiences?resource="
              + URLEncoder.encode(resourceUrl(), StandardCharsets.UTF_8)))
          .timeout(Duration.ofSeconds(5)).GET().build();
      HttpResponse<String> response = RETIRED_HTTP.send(request, HttpResponse.BodyHandlers.ofString());
      if (response.statusCode() == 200
          && JSONObjectUtils.parse(response.body()).get("retired") instanceof List<?> list) {
        urls = list.stream().filter(String.class::isInstance).map(String.class::cast)
            .collect(Collectors.toUnmodifiableSet());
        ttl = Duration.ofMinutes(5);
      }
    } catch (Exception e) {
      // unreachable: accept only RESOURCE_URL until the retry
    }
    retired = urls;
    retiredExpiresAt = now + ttl.toNanos();
    return retired;
  }

  // The enforcement state — whether the plugin is published, the one input that
  // turns the /mcp gate on. Fetched before the first request a process serves is
  // handled (one attempt at a time under the monitor; a concurrent caller waits
  // and takes its answer), cached 5 minutes and refreshed in the background after
  // that; keyed by RESOURCE_URL and by nothing in any request. true is final for
  // the process. A failed fetch keeps the last answer; with no answer yet the
  // gate is strict, and the fetch is retried after 1 minute.
  private static final HttpClient ENFORCEMENT_HTTP =
      HttpClient.newBuilder().connectTimeout(Duration.ofSeconds(5)).build();
  private static final ExecutorService ENFORCEMENT_REFRESHER =
      Executors.newSingleThreadExecutor(r -> { Thread t = new Thread(r, "plugpass-enforcement"); t.setDaemon(true); return t; });
  private static Boolean enforcementAnswer = null; // the last answer; null until one arrives
  private static long enforcementExpiresAt = System.nanoTime();
  private static long enforcementAttemptAt = System.nanoTime();

  private static synchronized void refreshEnforcement() {
    long now = System.nanoTime();
    if (now - enforcementAttemptAt < 0) return;
    enforcementAttemptAt = now + Duration.ofMinutes(1).toNanos();
    try {
      HttpRequest request = HttpRequest.newBuilder(URI.create(entitlementApiOrigin()
              + "/entitlement/enforcement?resource="
              + URLEncoder.encode(resourceUrl(), StandardCharsets.UTF_8)))
          .timeout(Duration.ofSeconds(5)).GET().build();
      HttpResponse<String> response = ENFORCEMENT_HTTP.send(request, HttpResponse.BodyHandlers.ofString());
      if (response.statusCode() == 200
          && JSONObjectUtils.parse(response.body()).get("enforced") instanceof Boolean answer) {
        enforcementAnswer = answer;
        enforcementExpiresAt = System.nanoTime() + Duration.ofMinutes(5).toNanos();
      }
    } catch (Exception e) {
      // unreachable: the last answer stands (strict while there is none) until the retry
    }
  }

  public static boolean enforced() {
    Boolean answer;
    long expiresAt;
    synchronized (PremiumFeatureAccessCheck.class) {
      answer = enforcementAnswer;
      expiresAt = enforcementExpiresAt;
    }
    if (Boolean.TRUE.equals(answer)) return true; // final
    if (answer == null) {
      // No answer yet: learn it before handling the request; strict until it arrives.
      refreshEnforcement();
      synchronized (PremiumFeatureAccessCheck.class) {
        return enforcementAnswer == null || enforcementAnswer;
      }
    }
    if (System.nanoTime() - expiresAt >= 0) {
      // Off and stale: refresh in the background, the cached answer serving meanwhile.
      ENFORCEMENT_REFRESHER.execute(PremiumFeatureAccessCheck::refreshEnforcement);
    }
    return false;
  }

  // The URL a request was addressed to — X-Forwarded-Host (a proxy on an old
  // address sets it), else its Host, on RESOURCE_URL's scheme and path — when
  // that URL is an accepted audience; otherwise RESOURCE_URL. At the test path,
  // the same with the /test suffix.
  public static String addressedResource(HttpServletRequest req, boolean test) {
    String suffix = test ? TEST_PATH_SUFFIX : "";
    String forwarded = req.getHeader("X-Forwarded-Host");
    String raw = forwarded != null ? forwarded.split(",")[0] : req.getHeader("Host");
    String host = raw == null ? "" : raw.trim().toLowerCase(Locale.ROOT);
    URI current = URI.create(resourceUrl());
    String currentHost = current.getPort() == -1 ? current.getHost() : current.getHost() + ":" + current.getPort();
    if (host.isEmpty() || host.equals(currentHost)) return resourceUrl() + suffix;
    String candidate = current.getScheme() + "://" + host + current.getPath();
    return (retiredAudiences().contains(candidate) ? candidate : resourceUrl()) + suffix;
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
   * EdDSA-pinned, issuer exact, exp + sub required, audience = the path's own
   * resource (the baked RESOURCE_URL, or its /test form at the test path) or one
   * of this server's retired URLs in the same form — a canonical token is refused
   * at the test path and a test token at the canonical path. Nimbus selects the
   * OKP key by kid; the JDK verifies Ed25519 (raw x wrapped in the fixed 12-byte
   * SPKI prefix). Returns sub, or null on any token failure; throws
   * KeySetUnavailable when the key set couldn't be fetched.
   */
  public static String verifyBearer(String token, boolean test) {
    String suffix = test ? TEST_PATH_SUFFIX : "";
    try {
      SignedJWT jwt = SignedJWT.parse(token);
      if (!JWSAlgorithm.EdDSA.equals(jwt.getHeader().getAlgorithm())) return null; // alg pinned
      JWKSelector selector = new JWKSelector(new JWKMatcher.Builder()
          .keyType(KeyType.OKP).curves(Curve.Ed25519)
          .keyID(jwt.getHeader().getKeyID()).build());
      List<JWK> keys;
      try {
        keys = JWKS.get(selector, null);
      } catch (RateLimitReachedException e) {
        return null; // a rate-limited refetch for a key the set lacks: the token's fault
      } catch (KeySourceException e) {
        throw new KeySetUnavailable(e);
      }
      if (keys.isEmpty()) return null;
      OctetKeyPair okp = keys.get(0).toOctetKeyPair();

      byte[] spki = concat(ED25519_SPKI_PREFIX, okp.getDecodedX()); // {0x30,0x2a,0x30,0x05,0x06,0x03,0x2b,0x65,0x70,0x03,0x21,0x00}
      PublicKey publicKey = KeyFactory.getInstance("Ed25519").generatePublic(new X509EncodedKeySpec(spki));
      Signature sig = Signature.getInstance("Ed25519");
      sig.initVerify(publicKey);
      sig.update(jwt.getSigningInput());
      if (!sig.verify(jwt.getSignature().decode())) return null;

      // Claims: iss exact, aud contains the path's own resource (or, only when it
      // does not, a retired URL's same form), exp + sub required (60s default
      // clock skew).
      JWTClaimsSet claims = jwt.getJWTClaimsSet();
      String own = resourceUrl() + suffix;
      Set<String> accepted = claims.getAudience().contains(own)
          ? Set.of(own)
          : Stream.concat(Stream.of(own), retiredAudiences().stream().map(url -> url + suffix))
              .collect(Collectors.toUnmodifiableSet());
      new DefaultJWTClaimsVerifier<SecurityContext>(
          accepted,
          new JWTClaimsSet.Builder().issuer(issuer()).build(),
          Set.of("sub", "exp"),
          null
      ).verify(claims, null);
      return claims.getSubject();
    } catch (KeySetUnavailable e) {
      throw e;
    } catch (Exception e) {
      return null;
    }
  }

  /** The signing key set couldn't be fetched: no verdict on the bearer, so the
   *  filter answers 503, never the 401 a client reads as signed out. */
  public static final class KeySetUnavailable extends RuntimeException {
    KeySetUnavailable(Throwable cause) {
      super("The signing key set could not be resolved", cause);
    }
  }

  /** Per-request holder the filter creates, the context extractor forwards,
   *  and tool handlers read — absent on a request with no identity (the /mcp
   *  gate off, before the plugin is published), which every tool handles
   *  explicitly. The AtomicBoolean is the reauth back-channel; the filter creates
   *  it for every request, identity or none. */
  public record PlugpassAuth(String sub, String bearer, AtomicBoolean reauthRequired) {}
  public static final String AUTH_ATTRIBUTE = "plugpass.auth";
  public static final String REAUTH_ATTRIBUTE = "plugpass.reauth";

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
  // All read the tool's own registered meta and the request, at runtime.
  public static CallToolResult nonAuthorizedToolResponse(Map<String, Object> result, DeniedCall call, Map<String, Object> toolMeta, CallToolRequest request, String featureId) {
    var response = CallToolResult.builder().addTextContent(String.valueOf(result.get("result_text")));
    if (paywallUi(toolMeta, request)) response.addTextContent(PAYWALL_UI_MARKER);
    Map<String, Object> deniedCall = new LinkedHashMap<>();
    deniedCall.put("name", call.name());
    deniedCall.put("arguments", call.arguments() == null ? Map.of() : call.arguments());
    deniedCall.put("widget_callable", widgetCallable(toolMeta));
    Map<String, Object> meta = new LinkedHashMap<>();
    meta.put("plugpass_denied_call", deniedCall);
    Map<String, Object> probe = statusProbe(toolMeta, featureId);
    if (probe != null) meta.put("plugpass_status_probe", probe);
    return response.meta(meta).build();
  }

  // The read-only entitlement probe the paywall may call when it cannot replay a
  // denied call: this server's check tool, with the arguments already composed so
  // the widget's script supplies nothing of its own. Named ONLY when the call is
  // unreplayable (a model-only UI-backed tool) and this server hosts a check tool;
  // every other denial carries none, since a replay answers the same question by
  // actually running the call.
  private static Map<String, Object> statusProbe(Map<String, Object> toolMeta, String featureId) {
    if (checkToolName() == null || !(toolMeta.get("ui") instanceof Map<?, ?>) || widgetCallable(toolMeta)) {
      return null;
    }
    Map<String, Object> arguments = new LinkedHashMap<>();
    arguments.put("plugin_id", pluginId());
    arguments.put("feature_id", featureId);
    arguments.put("status_code", true);
    Map<String, Object> probe = new LinkedHashMap<>();
    probe.put("name", checkToolName());
    probe.put("arguments", arguments);
    return probe;
  }

  // Whether the calling client renders MCP Apps widgets: it declared the UI
  // extension among the client capabilities the request carries in its `_meta`
  // (`io.modelcontextprotocol/clientCapabilities`). Read off the tool call
  // request's meta(). This SDK line serves the 2025 era only, so a modern host
  // never reaches here with that key — see the note at the top. A client that
  // declares nothing renders nothing, and gets no marker.
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

**`BearerAuthFilter`** — mapped on `/mcp/*` (which covers `/mcp` and `/mcp/test`): the gate plus the reauth swap (buffer POST bodies only). The test path is the same servlet: the filter rewrites the request URI to `/mcp` on the way in, after the gate has read it, since the stateless transport 404s any other URI:

```java
public final class BearerAuthFilter extends HttpFilter {
  @Override
  protected void doFilter(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
      throws IOException, ServletException {
    String uri = req.getRequestURI();
    boolean test = uri.equals(PremiumFeatureAccessCheck.TEST_MCP_PATH);
    if (!test && !uri.equals(PremiumFeatureAccessCheck.MCP_PATH)) { res.sendError(404); return; }
    String resource = PremiumFeatureAccessCheck.addressedResource(req, test);
    // The gate: the test path is strict from the first deploy; /mcp challenges
    // only while the plugin is published. Off, a request with no valid bearer is
    // handled with no identity — never challenged.
    boolean strict = test || PremiumFeatureAccessCheck.enforced();
    String header = req.getHeader("Authorization");
    String token = header != null && header.regionMatches(true, 0, "Bearer ", 0, 7) ? header.substring(7).trim() : null;
    String sub;
    try {
      sub = token == null ? null : PremiumFeatureAccessCheck.verifyBearer(token, test);
    } catch (PremiumFeatureAccessCheck.KeySetUnavailable e) {
      keySetUnavailable(res);
      return;
    }
    if (sub == null && strict) {
      challenge(res, resource, token == null ? "Missing bearer token" : "Token invalid or expired");
      return;
    }
    // Off, a malformed or expired bearer is no bearer: no identity is set.
    var reauth = new AtomicBoolean(false);
    req.setAttribute(PremiumFeatureAccessCheck.REAUTH_ATTRIBUTE, reauth);
    if (sub != null) {
      req.setAttribute(PremiumFeatureAccessCheck.AUTH_ATTRIBUTE,
          new PremiumFeatureAccessCheck.PlugpassAuth(sub, token, reauth));
    }

    HttpServletRequest inner = test ? new HttpServletRequestWrapper(req) {
      @Override public String getRequestURI() { return PremiumFeatureAccessCheck.MCP_PATH; }
      @Override public String getServletPath() { return PremiumFeatureAccessCheck.MCP_PATH; }
      @Override public String getPathInfo() { return null; }
    } : req;
    BufferingResponseWrapper buffer = new BufferingResponseWrapper(res); // overrides getWriter/getOutputStream only
    chain.doFilter(inner, buffer);                                      // stateless transport: fully synchronous
    if (reauth.get() && !res.isCommitted()) {
      challenge(res, resource, "Access token no longer valid");         // → the client re-authorizes and retries
      return;
    }
    buffer.replayTo(res); // status/headers passed through; body copied verbatim
  }

  // The signing key set couldn't be fetched: no verdict on the bearer.
  private static void keySetUnavailable(HttpServletResponse res) throws IOException {
    res.setStatus(503);
    res.setHeader("Retry-After", "30");
    res.setHeader("Cache-Control", "no-store");
    res.setContentType("application/json");
    res.getWriter().write("{\"error\": \"temporarily_unavailable\", \"error_description\": \"The signing key set could not be resolved\"}");
  }

  // The challenge for the addressed resource. error="invalid_token" exactly — the
  // signal MCP clients treat as "needs OAuth".
  private static void challenge(HttpServletResponse res, String resource, String description) throws IOException {
    res.setStatus(401);
    res.setHeader("WWW-Authenticate", "Bearer realm=\"" + resource
        + "\", error=\"invalid_token\", error_description=\"" + description
        + "\", resource_metadata=\"" + PremiumFeatureAccessCheck.prmUrl(resource) + "\"");
    res.setContentType("application/json");
    res.getWriter().write("{\"error\": \"invalid_token\", \"error_description\": \"" + description + "\"}");
  }
}
```

**`Main`** — stateless transport + context extractor + Jetty composition:

```java
var transport = HttpServletStatelessServerTransport.builder()
    .mcpEndpoint(PremiumFeatureAccessCheck.MCP_PATH)
    // The holder rides the context when the request carries an identity; a
    // request with none (the /mcp gate off) carries an empty context.
    .contextExtractor(request -> {
      Object auth = request.getAttribute(PremiumFeatureAccessCheck.AUTH_ATTRIBUTE);
      return McpTransportContext.create(
          auth == null ? Map.of() : Map.of(PremiumFeatureAccessCheck.AUTH_ATTRIBUTE, auth));
    })
    .build();

McpServer.sync(transport)
    .serverInfo("<server name>", "<version>")
    .capabilities(ServerCapabilities.builder().tools(true).build())
    .tools(/* …McpStatelessServerFeatures.SyncToolSpecification list, incl. the check proxy… */)
    .build();

Server jetty = new Server(new InetSocketAddress("0.0.0.0", port));
ServletContextHandler handler = new ServletContextHandler();
handler.setContextPath("/");
// `/mcp/*` reaches /mcp and /mcp/test alike; the filter 404s every other URI
// under it and rewrites the test path to /mcp for the transport.
handler.addServlet(new ServletHolder(transport), PremiumFeatureAccessCheck.MCP_PATH + "/*");
handler.addServlet(new ServletHolder(new PrmServlet()), "/.well-known/oauth-protected-resource/mcp/*");
FilterHolder auth = new FilterHolder(new BearerAuthFilter());
auth.setAsyncSupported(true);
handler.addFilter(auth, PremiumFeatureAccessCheck.MCP_PATH + "/*", EnumSet.of(DispatcherType.REQUEST)); // the MCP paths ONLY — never the PRM paths
jetty.setHandler(handler);
jetty.start();
jetty.join();
```

`PrmServlet` is a trivial `doGet` serving both documents, unauthenticated: for a request URI of `/.well-known/oauth-protected-resource/mcp` or `…/mcp/test` (404 otherwise) it writes `{"resource": addressedResource(req, test), "authorization_servers": [issuer()], "bearer_methods_supported": ["header"]}` as `application/json`, `test` being whether the URI ends in `/test`.

**Per-request identity inside a tool**: stateless handlers receive the transport context directly — `BiFunction<McpTransportContext, CallToolRequest, CallToolResult>`; read `var auth = (PlugpassAuth) ctx.get(PremiumFeatureAccessCheck.AUTH_ATTRIBUTE)`, `null` on a request with no identity (the /mcp gate off, before the plugin is published), which every tool handles explicitly. On reauth: `auth.reauthRequired().set(true)`.

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
                // plugin_version: required for a skill's access check (it carries the
                // installed bundle's version); a tool-surface check does not use it.
                "plugin_version", Map.of("type", "string"),
                // Reserved for the Plugpass paywall's own use.
                "status_code", Map.of("type", "boolean", "description",
                    "Never include this parameter in your tool calls under any circumstance.")),
            "required", List.of("plugin_id", "feature_id")))
        .title("Check premium access")
        .description(CHECK_PREMIUM_ACCESS_DESCRIPTION)
        .annotations(hints(false, false, false, false))
        .build()) // no outputSchema
    .callHandler((ctx, request) -> {
      var auth = (PremiumFeatureAccessCheck.PlugpassAuth) ctx.get(PremiumFeatureAccessCheck.AUTH_ATTRIBUTE);
      // No identity — the /mcp gate off, before the plugin is published: the one
      // answer the proxy composes itself, without reaching Plugpass.
      if (auth == null) return unavailableCheckGrant();
      // The paywall's read-only probe: asks whether this user is entitled NOW,
      // consuming nothing (check_remaining, never check_premium_access), and answers
      // in a code that is not a check result — no PLUGPASS_PLUGIN, no USE_AUTHORIZED
      // — so no access handler triggers on it and nothing downstream can read it as a
      // grant. It authorizes NOTHING; the call the user retries is checked on its own.
      if (Boolean.TRUE.equals(request.arguments().get("status_code"))) {
        var probed = PremiumFeatureAccessCheck.entitlement(auth.bearer(),
            Map.of("plugin_id", request.arguments().get("plugin_id"),
                   "feature_id", request.arguments().get("feature_id")), "check_remaining");
        if ("reauth_required".equals(probed.status())) {
          auth.reauthRequired().set(true); // → transport-level 401 via the filter
          return CallToolResult.builder().addTextContent("Re-authentication required.").build();
        }
        // ONLY a definite answer carries a code. "unavailable" covers both a Plugpass
        // outage and a feature this endpoint cannot evaluate, and neither establishes
        // that the user is unentitled — so the probe stays silent rather than
        // asserting a denial it did not establish.
        String code = "unavailable".equals(probed.status()) ? "STATUS_UNKNOWN"
            : "ok".equals(probed.status()) ? "STATUS_CODE=1" : "STATUS_CODE=0";
        return CallToolResult.builder().addTextContent(code).build();
      }
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

**Solo paid tool wrapper** (entry, body, settle; no `auth_token` parameter on this path): read the holder from the context — `null` (no bearer: the /mcp gate off, the plugin unpublished) runs the body with no entitlement call, as the tool ran before Plugpass; with one, `auth.sub()` scopes the body and `auth.bearer()` is the forwarded bearer — call `track_usage` (`feature_id` = the tool's own plugpass-component-id, matching its `_meta`; the id's `tool_` prefix carries the feature type), and branch: `ok` → run the body; `non_authorized` → `nonAuthorizedToolResponse(result, new DeniedCall("<tool name>", request.arguments()), PAID_TOOL_META, request, PAID_TOOL_FEATURE_ID)` (the renderer reads the tool's own registered meta — its widget, who may call it — and the request, never a baked per-tool constant); `reauth_required` → set the flag + placeholder; `unavailable` → run the body (it consumed nothing and grants). Stamp `_meta` via `Tool.builder(...).meta(PAID_TOOL_META)`, the map hoisted to a `static final String PAID_TOOL_FEATURE_ID = "<plugpass_id>"` + `static final Map<String, Object> PAID_TOOL_META = Map.of("plugpass_component_id", PAID_TOOL_FEATURE_ID)` (a UI-backed tool's also carries its `"ui", Map.of("resourceUri", …, "visibility", …)`) that both the builder and the marker rule read, and never declare an `outputSchema` on a wrapped tool (paywall/reauth responses are text-only). Re-derive the tool's `.annotations(…)`: metering makes it neither read-only nor idempotent, whatever it was before the wrap — `.annotations(hints(false, <its own destructive value>, false, <its own open-world value>))`, see TOOLS.md § Tool annotations. **The call id, the quantities, and the settle** (TOOLS.md → The settle, Reading a tool's quantities): mint the call id with `UUID.randomUUID().toString()` before the entry and send it, with `quantities` — each recorded quantity's key mapped to an input's count from the arguments, or, when the tool takes a size argument, an output's requested size (its default when the caller omitted it) — on `track_usage`. On `ok`, cap the tool's size argument at `output.max` (null leaves it as asked) and keep `settle`; the unavailable grant is `settle: none`. Run the body; when it throws or returns an error result and `settle` isn't `none`, call `settle(bearer, Map.of(…))` with `outcome: "failure"` before failing as the tool would. On a success with `settle: always`, call it with `outcome: "success"` and the result's item count under `output.key`, then: `non_authorized` → the same denial as at entry; `ok` → `deliverSettled(result, settled, trim, call, PAID_TOOL_META)`, which trims every representation of the result to `deliver` (the tool's own `trim`, its text and its structured content alike) and adds `note` as a text block of its own and, when the result carries structured content, under its `limit_note` key, and `partial` on `_meta.plugpass_partial` beside the call echo; anything else (reauth included) → the whole result, with no note. The settle client POSTs `{origin}/entitlement/settle` with the entry's timeout and retry.

**Paired-tool add side** (`operation: add`): same shape, and `feature_id` is the tool's OWN `plugpass_id` (its `tool_` prefix carries the feature type) exactly as for a solo tool — every gated artifact bakes its own component's id, and the `entitlement` subfield is identity, never a `feature_id`. Always **`check_remaining`**, passing the user's current count from the publisher's own store (scoped to `sub`) as `current_count` — see TOOLS.md → Reading the user's current count — and `adding`, how many records the call adds (`1`, or the list's length for a batch). No call id and no settle: a record add never consumes.

**Identity tool** (the paired remove side, or any per-user free tool): no Entitlement API call — read the holder from the context and scope the body to `auth.sub()`; with `null` (the /mcp gate off), answer that nobody is signed in and touch no record:

```java
var auth = (PremiumFeatureAccessCheck.PlugpassAuth) ctx.get(PremiumFeatureAccessCheck.AUTH_ATTRIBUTE);
if (auth == null) return CallToolResult.builder().addTextContent("No user is signed in.").build();
// …existing tool body, scoped to auth.sub()…
```

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

**Placement guidance.** An embedded-Jetty server follows the composition above; a servlet-container deployment registers the same three pieces in its `web.xml`/programmatic config (filter on `/mcp/*` only — strict at the test path, armed by publish at `/mcp` through `enforced()`, never challenging while off — `asyncSupported` true, PRM servlet public at both paths). On a server whose directive names `ui_paywall`, every `text/html;profile=mcp-app` resource read passes through `withPaywall`. If the publisher's server must stay on the stateful `HttpServletStreamableServerTransportProvider` (tools that use sampling/elicitation), the identity path changes to `exchange.transportContext()` on `McpSyncServerExchange` and the same buffering filter works via the servlet's deferred-`complete()` semantics — but never wrap the GET listening stream, leave `keepAliveInterval` unset, and know that mid-call notifications are delayed until completion. Migration notes for a publisher's older SDK: 1.x `McpSchema.JsonSchema` is deprecated (bridge maps in), and 2.0 validates tool inputs by default.
