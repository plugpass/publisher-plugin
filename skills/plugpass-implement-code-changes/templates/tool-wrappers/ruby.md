# Ruby scaffolding template

The publisher's server becomes an **OAuth-protected resource server**: a Rack layer in front of the official `mcp` gem's `StreamableHTTPTransport` gates `/mcp` (validating the bearer against Plugpass's JWKS), serves the RFC 9728 PRM document, and carries per-request identity on a thread-local. Standalone source, no platform package. Version pins: `gem "mcp", "~> 0.24"` (**pin it — the gem ships breaking security defaults across minor versions**), `gem "jwt", "~> 3.2"`, `puma`, `rack`, `rackup`. Ruby ≥ 3.1 (stdlib OpenSSL verifies Ed25519 — **no `ed25519`/`rbnacl` native gems**).

**Three structural decisions carry the whole design — never undo them:**

1. **`enable_json_response: true` on the transport, always.** In JSON mode the whole JSON-RPC dispatch — including your tool blocks — runs synchronously on the request thread inside `transport.call(env)`, so the wrapping Rack layer regains control with the response uncommitted: thread-local identity works, and the layer can swap in a `401` when a tool discovered mid-call that the bearer is revoked. In the default SSE mode dispatch happens inside the streaming body *after* the middleware returned — both mechanisms silently die.
2. **`dns_rebinding_protection: false`.** The gem's ≥ 0.23 default validates `Host` against loopback-only allowlists — dev on localhost works, then **every request to the deployed public host 403s**. Rebinding protection defends unauthenticated localhost servers; here every request requires a validated bearer, so disable it (or set `allowed_hosts` from env).
3. **The audience is the baked `RESOURCE_URL`, never the request host** — the JWT `aud` pin, the PRM `resource`, and the challenge's `resource_metadata` all derive from it. This is what lets a locally-listening server accept real bearers minted for its public URL.

```ruby
# premium_feature_access_check.rb — written once per server. Standalone.
require "base64"
require "json"
require "jwt"
require "net/http"
require "openssl"

module PremiumFeatureAccessCheck
  MCP_PATH = "/mcp"

  # Plugpass endpoints for this server.
  ISSUER = "<plugpass_issuer>"
  JWKS_URL = "<plugpass_jwks_url>"
  ENTITLEMENT_API_ORIGIN = "<entitlement_api_origin>"
  # This server's own public MCP URL — the JWT `aud` pin and the PRM `resource`.
  RESOURCE_URL = "<this server's RESOURCE_URL>"

  def self.issuer = ISSUER
  def self.jwks_url = JWKS_URL
  def self.entitlement_api_origin = ENTITLEMENT_API_ORIGIN
  def self.resource_url = RESOURCE_URL
  # RFC 9728 path-aware form: origin + /.well-known/oauth-protected-resource + /mcp.
  def self.prm_url = "#{resource_url.delete_suffix(MCP_PATH)}/.well-known/oauth-protected-resource#{MCP_PATH}"

  # Core jwt 3.x has no EdDSA and rejects OKP JWKs — this custom algorithm
  # verifies Ed25519 through stdlib OpenSSL (Ruby >= 3.1), zero native deps.
  class Ed25519Algorithm
    include JWT::JWA::SigningAlgorithm
    def initialize = @alg = "EdDSA"
    attr_reader :alg
    def verify(data:, signature:, verification_key:)
      verification_key.verify(nil, signature, data) # Ed25519: digest must be nil
    rescue OpenSSL::OpenSSLError
      false
    end
  end
  ED25519 = Ed25519Algorithm.new

  # JWKS cache: 4h TTL, refetch on an unknown kid rate-limited to one per 10s
  # (key rotation). Mutex-guarded — puma is multi-threaded.
  JWKS_TTL_SECONDS = 14_400
  @jwks_mutex = Mutex.new
  @jwks_cache = {}

  def self.key_for(kid)
    # …cache lookup keyed by jwks_url: fresh hit → find kid; miss/stale →
    # rate-limited refetch (Net::HTTP, 5s timeouts) → find kid; convert the
    # OKP/Ed25519 JWK via:
    #   OpenSSL::PKey.new_raw_public_key("ED25519", Base64.urlsafe_decode64(jwk["x"]))
    # Monotonic clock for timestamps; nil on any failure…
  end

  # EdDSA-pinned, issuer exact, audience = the baked RESOURCE_URL, exp+sub
  # required. Returns the user id (sub), or nil on any failure.
  def self.verify_bearer(token)
    kid = JWT::EncodedToken.new(token).header["kid"]
    key = key_for(kid)
    return nil unless key
    payload, = JWT.decode(token, key, true,
      algorithm: ED25519,                 # the INSTANCE — pins the alg, no downgrade
      verify_aud: true, aud: resource_url,
      verify_iss: true, iss: issuer,
      required_claims: %w[exp aud iss sub])
    payload["sub"]
  rescue JWT::DecodeError
    nil
  end

  # Per-request identity + the reauth flag, thread-local (one puma thread per
  # request; the gate resets both in ensure).
  def self.identity = Thread.current[:plugpass_identity]
  def self.reauth_required! = Thread.current[:plugpass_reauth_required] = true

  class RetryableEntitlementError < StandardError; end

  # Entitlement API client. 5s timeouts PER ATTEMPT, with ONE retry after a 2s
  # backoff on a transport error or a 5xx (4xx are terminal, never retried).
  # HTTP 401 → reauth; every other failure → "unavailable", which GRANTS: the
  # call is server-to-server from this host, so the end user cannot have caused
  # it.
  def self.entitlement(bearer, body, op)
    attempt = 0
    begin
      uri = URI("#{entitlement_api_origin}/entitlement/#{op}")
      http = Net::HTTP.new(uri.host, uri.port)
      http.use_ssl = uri.scheme == "https"
      http.open_timeout = http.read_timeout = http.write_timeout = 5
      req = Net::HTTP::Post.new(uri, "Authorization" => "Bearer #{bearer}", "Content-Type" => "application/json")
      req.body = JSON.generate(body)
      res = http.request(req)
      return { "status" => "reauth_required" } if res.code == "401"
      raise RetryableEntitlementError, "entitlement #{op}: #{res.code}" if res.code.to_i >= 500
      return { "status" => "unavailable" } unless res.code.start_with?("2")

      JSON.parse(res.body)
    rescue RetryableEntitlementError, SystemCallError, Timeout::Error, Net::OpenTimeout, Net::ReadTimeout, OpenSSL::SSL::SSLError, JSON::ParserError
      return { "status" => "unavailable" } if attempt.positive?

      attempt += 1
      sleep(2)
      retry
    end
  end

  # non_authorized → result_text verbatim — a single-field pipe (the trigger
  # keys inside it auto-fire the plugin's access-handler skill). Never
  # parse, reformat, or re-serialize it.
  def self.non_authorized_text(result)
    result.fetch("result_text")
  end

  # The check proxy's unavailable grant — Plugpass could not answer, so the
  # check grants and the paid skill runs. A wrapped tool needs no equivalent: it
  # just runs its body.
  UNAVAILABLE_CHECK_GRANT = "<the unavailable grant text from TOOLS.md>"
end
```

**Server entry** (`server.rb`) — the gate wraps the gem's Rack transport; PRM public, everything on `/mcp` bearer-gated:

```ruby
require "mcp"
require "rackup"

class PlugpassGate
  P = PremiumFeatureAccessCheck

  def initialize(app) = @app = app

  def call(env)
    return prm_response if env["PATH_INFO"] == prm_path && env["REQUEST_METHOD"] == "GET"
    return [404, { "content-type" => "application/json" }, ['{"error": "not_found"}']] unless env["PATH_INFO"] == P::MCP_PATH

    token = env["HTTP_AUTHORIZATION"] && env["HTTP_AUTHORIZATION"][/\ABearer (.+)\z/i, 1]
    return challenge("Missing bearer token") unless token
    sub = P.verify_bearer(token)
    return challenge("Token invalid or expired") unless sub

    Thread.current[:plugpass_identity] = { sub: sub, token: token }
    status, headers, body = @app.call(env)  # JSON mode: dispatch is synchronous, in-thread
    return challenge("Access token no longer valid") if Thread.current[:plugpass_reauth_required]
    [status, headers, body]
  ensure
    Thread.current[:plugpass_identity] = nil
    Thread.current[:plugpass_reauth_required] = nil
  end

  private

  def prm_path = "/.well-known/oauth-protected-resource#{P::MCP_PATH}"

  def prm_response
    [200, { "content-type" => "application/json" }, [JSON.generate(
      resource: P.resource_url,
      authorization_servers: [P.issuer],
      bearer_methods_supported: ["header"]
    )]]
  end

  # error="invalid_token" exactly — the signal MCP clients treat as "needs OAuth".
  def challenge(description)
    header = "Bearer realm=\"#{P.resource_url}\", error=\"invalid_token\", " \
             "error_description=\"#{description}\", resource_metadata=\"#{P.prm_url}\""
    [401, { "content-type" => "application/json", "www-authenticate" => header },
     [JSON.generate(error: "invalid_token", error_description: description)]]
  end
end

server = MCP::Server.new(name: "<server name>", version: "<version>", tools: [
  # ...tool classes / Tool.define results, including the check proxy...
])
transport = MCP::Server::Transports::StreamableHTTPTransport.new(
  server,
  enable_json_response: true,     # required: lets a revoked-bearer tool swap in a 401
  dns_rebinding_protection: false # required: the audience is the baked RESOURCE_URL, not the host
)
server.transport = transport
Rackup::Handler.get("puma").run(PlugpassGate.new(transport),
  Host: "0.0.0.0", Port: Integer(ENV.fetch("PORT", "8080")), Silent: true)
```

**The check proxy tool (check host only).** A pure pipe — never parse or reformat `result_text`:

```ruby
CHECK_PREMIUM_ACCESS_DESCRIPTION = "<the VERBATIM description from TOOLS.md § The check proxy tool>"

CheckPremiumAccess = MCP::Tool.define(
  name: "{CheckToolName}",
  title: "Check premium access",
  description: CHECK_PREMIUM_ACCESS_DESCRIPTION,
  input_schema: {
    type: "object",
    properties: {
      plugin_id: { type: "string" },
      feature_id: { type: "string" },
      plugin_version: { type: "string" },
    },
    required: %w[plugin_id feature_id plugin_version],
  },
  annotations: {
    read_only_hint: false,
    destructive_hint: false,
    idempotent_hint: false,
    open_world_hint: false,
  }
  # No output_schema.
) do |plugin_id:, feature_id:, plugin_version:, **|
  identity = PremiumFeatureAccessCheck.identity
  result = PremiumFeatureAccessCheck.entitlement(
    identity[:token],
    { plugin_id:, feature_id:, plugin_version: },
    "check_premium_access"
  )
  if result["status"] == "unavailable"
    next MCP::Tool::Response.new([{ type: "text", text: PremiumFeatureAccessCheck::UNAVAILABLE_CHECK_GRANT }])
  end
  if result["status"] == "reauth_required"
    PremiumFeatureAccessCheck.reauth_required!   # → transport-level 401 via the gate
    next MCP::Tool::Response.new([{ type: "text", text: "Re-authentication required." }]) # discarded by the swap
  end
  MCP::Tool::Response.new([{ type: "text", text: result["result_text"] }]) # verbatim — byte-identical to the native tool
end
```

**Solo paid tool wrapper** (consume-on-invocation; no `auth_token` parameter on this path): read `PremiumFeatureAccessCheck.identity` (`[:sub]` scopes the body, `[:token]` is the bearer to forward), call `entitlement(token, { plugin_id:, feature_id: "<plugpass_id>" }, "track_usage")`, and branch: `ok` → run the body; `non_authorized` → `MCP::Tool::Response.new([{ type: "text", text: non_authorized_text(result) }])`; `reauth_required` → `reauth_required!` + placeholder; `unavailable` → run the body (it consumed nothing and grants). Keep the tool's `meta: { plugpass_component_id: "<plugpass_id>" }` on `Tool.define`, and never declare an `output_schema` on a wrapped tool (the gem validates `structuredContent` against it, which paywall/reauth text responses would fail). Re-derive the tool's `annotations:`: metering makes it neither read-only nor idempotent, whatever it was before the wrap — `read_only_hint: false, idempotent_hint: false`, the other two keeping the tool's own values; see TOOLS.md § Tool annotations.

**Paired-tool add side** (`operation: add`): same shape but `feature_id` = the add tool's `database_record.custom_entitlement_id` (its `custom_` prefix carries the feature type; **not** `database_record.plugpass_id`, and **not** the tool's own `_meta` id), always **`check_remaining`**, passing the user's current count from the publisher's own store (scoped to `sub`) as `current_count` — see TOOLS.md → Reading the user's current count.

**Identity tool** (the paired remove side, or any per-user free tool): no Entitlement API call — read `PremiumFeatureAccessCheck.identity[:sub]` and scope the body to it.

**Placement guidance.** A standalone puma server follows the composition above. In a Rails app, mount the gate-wrapped transport (`mount PlugpassGate.new(transport) => "/mcp"` plus a route for the PRM path) — the same invariants hold. Tool blocks receive **symbolized keyword args**; `required` entries in `input_schema` are strings. Thread-locals are per-request under puma (one thread per request) — the pattern doesn't target fiber-scheduling servers like falcon. In-memory state means single-process puma (`workers 0`) or an external store.
