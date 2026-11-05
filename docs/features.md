# Features

Everything Lygard does, with a link to the page that explains each feature in depth. There is no paid edition: every feature listed here is open source.

## Traffic

### Proxy

HTTP/1.1 and HTTP/2 (ALPN), TLS 1.2/1.3 via rustls (no OpenSSL), SNI with hot
certificate reload, WebSocket/upgrade, h2c/gRPC backends, HTTPS backends, PROXY protocol v1/v2,
graceful drain on SIGTERM, zero-downtime config reload (atomic snapshot swap; invalid config is
rejected and the previous one keeps serving).

### Routing

Host (exact / wildcard), path (exact / element-wise prefix / anchored regex), method,
header and query matches with Gateway API precedence; weighted backends (canary) with
per-backend header changes; header add/set/remove; URL rewrite (prefix, full, regex); redirects
(301/302/303/307/308); fixed responses; custom error pages; gRPC-Web to gRPC translation;
Gateway API retries and timeouts; HTTP 421 for misdirected (coalesced) requests; request
mirroring (several mirrors per rule, percentage sampling, bodies up to 1 MiB mirrored intact);
backend TLS verified against a private CA (`BackendTLSPolicy`); TCP and UDP relays
(`TCPRoute` / `UDPRoute`, one UDP session per client, 60 s idle timeout). Paths are
normalised (dot-segments, duplicate slashes) before routing, so `/public/../admin` cannot slip
past a `/public` rule.

### Resilience

Least-request (P2C), round-robin, random and consistent-hash load balancing;
active health checks; passive outlier ejection capped at a share of the backend; circuit breaker;
retries with per-try timeout and retry-safety rules (bodies replayed only when buffered, status
retries only for idempotent methods).

### Zero-downtime operations

Draining pods answer HTTP/1 with `Connection: close` and keep
serving until load balancers notice; refused or reset backend connections are retried once on
another endpoint; per-cloud Helm presets size the drain timing to each load balancer. Measured:
zero failed requests across rollouts and node drains on Kubernetes 1.31–1.37, and through a
real cloud load balancer.

### Progressive rollouts with automatic rollback

Annotate an HTTPRoute whose rules list the
stable Service and a new one with `lygard.io/rollout-canary: <service>` and the controller moves
traffic to it in steps (`lygard.io/rollout-steps`, default `5,25,50,100`), each held for
`lygard.io/rollout-interval` (default 1m) and until the canary has served
`lygard.io/rollout-min-requests` (default 50). Every step is judged on the gateway's own numbers
from all replicas: a canary 5xx rate above `lygard.io/rollout-max-error-rate` (default 1%), or a
p95 more than `lygard.io/rollout-max-latency-increase` (default 50%) above the stable backends',
rolls it back to 0% at once, mid-step if need be. After the last step it is Promoted at 100%.
Without enough traffic it holds rather than promote blind. `lygard.io/rollout-paused: "true"`
pauses, `lygard.io/rollout-abort: "true"` rolls back, and changing `lygard.io/rollout-revision`
(e.g. to the image tag, from CI) or the canary Service starts a new rollout. Progress is reported
as Events on the HTTPRoute, at the controller's `/rollouts` endpoint and as
`lygard_controller_rollout_*` metrics; state survives controller restarts and leader changes.
See `examples/rollout-demo.yaml`.

## Security

### WAF and access control

Native WAF (48 rules: SQLi, XSS, RCE, LFI/RFI, SSRF, Log4Shell, SSTI, prototype
pollution, XXE, scanners, protocol abuse) with input normalisation, anomaly scoring, paranoia
levels 1–4, detect/block modes, per-route rule and argument exclusions, and custom regex rules;
**rule packs** (WordPress, PHP, sensitive files, bad bots, header tricks, admin paths), condition
rules (method, path, header, cookie, IP, country → block / allow / log / score), attack categories
switchable per route, **IP reputation lists** (Spamhaus DROP, FireHOL, Tor exits, your own from a
ConfigMap or URL), all editable from the dashboard as GitOps-ready YAML ([waf rules](waf-rules.md));
token-bucket rate limiting; bcrypt basic auth; JWT (HS/RS/PS/ES/EdDSA, static keys or JWKS);
IP allow/deny lists; CORS; security headers incl. HSTS; request body limits; strict PROXY
protocol; slowloris and HTTP/2 rapid-reset limits; **cluster-wide rate limits without Redis**
(replicas exchange usage every 200 ms; measured on a 2-replica cluster: 60–69 requests admitted
of a 300-request burst vs 108–110 with per-replica limits); **TLS passthrough** by SNI on the
same port as terminated HTTPS.

### OWASP CRS / ModSecurity compatibility

A SecLang engine that loads the **unmodified OWASP
Core Rule Set v4** (shipped in the image) with anomaly scoring, paranoia levels, `ctl` runtime
exclusions, chains, transformations and libinjection. Measured with CRS's own regression suite
(`lygard waf-test --crs <dir>`): **5,074 / 5,164 tests (98.3%) pass** against CRS v4.30.0.
Routes opt in per route (`lygard.io/waf-engine: crs`); the faster native engine stays the default.

### WAF false-positive tuning

Every WAF match is grouped by route, rule and target (the
argument, header or cookie it fired on; native and CRS naming are unified). The dashboard's WAF
tuning panel ranks the groups: many different clients, a free-text field (`comment`, `bio`, `q`,
…) and no other rule firing alongside point to a false positive; one client tripping several
rules at once points to an attack. Each suggestion is a *targeted* exclusion — that rule, on that
field, under the common path prefix — with how many of the route's recent blocked requests it
would have let through, and a copy button for the exact `kubectl annotate` command (existing
exclusions on the route are kept) or config snippet. Exclusions are written as
`lygard.io/waf-exclusions: "942100,942110 args:comment /blog POST; 941100 header:referer"`
(`<rule ids|*> [args:<name>|header:<name>|cookie:<name>|body|path] [/path-prefix …] [METHOD …]`)
or as `waf.exclusions` in a route policy, and work with both the native engine and the CRS. JSON
fields are named by their path (`args:user.bio`).

### Edge authentication

Basic auth, JWT with required-claim rules (403 on missing scope/role),
external "forward" auth compatible with ingress-nginx `auth-url` and Traefik ForwardAuth (so
oauth2-proxy, Authelia or Pomerium add OIDC login to any route), and client-certificate (mTLS)
validation on Gateway listeners (Gateway API frontend validation, both modes). See
[authentication](authentication.md).

### Automatic certificates (ACME)

With `acme.enabled=true` (Let's Encrypt by default, any
RFC 8555 CA via `acme.server`), an Ingress annotated `lygard.io/acme: "true"` gets the Secrets
named in its `tls` entries issued for their hosts, and a Gateway HTTPS listener with
`tls.options: {lygard.io/acme: "true"}` gets its certificateRef issued for its hostname. HTTP-01
answers are served by the gateway itself (the controller adds them as exact-path routes, which
outrank the site's own routes and redirects), and the controller fetches each one through the
public hostname before asking the CA to validate. Hosts in a zone covered by an
`acme.dns01.solvers` entry, and wildcard hosts (`*.example.com`), use DNS-01 instead: the TXT
record is written through Cloudflare, Route 53, Cloud DNS, Azure DNS, DigitalOcean, Linode,
RFC 2136 or a webhook (following a CNAME on `_acme-challenge` into a separate validation zone),
and the controller waits until the zone's nameservers serve it. Certificates renew 30 days before expiry,
failures back off from 1 minute to 6 hours, and progress is reported as Events on the Ingress or
Gateway, at the controller's `/acme` endpoint and as `lygard_controller_acme_*` metrics. A
Secret that already exists without the `lygard.io/acme-managed` mark is never overwritten.
Enabling grants the controller write access to Secrets. See
[certificates](certificates.md).

## Observability

### Request IDs, metrics and the dashboard

Every request has one ID, equal to its trace ID, shown in the `x-request-id` header, on error pages, in logs and in traces. The request inspector finds any recent request by that ID and explains every decision taken for it. 40+ Prometheus metrics (requests, latency histograms, bytes, connections, TLS, upstreams, health checks, WAF verdicts per rule and action, challenges, IP lists, rate limits, auth outcomes); JSON access logs; an embedded dashboard; Grafana dashboard and Prometheus alert rules in the Helm chart. See [observability](observability.md) and [dashboard](dashboard.md).

### Status pages

Lygard's own errors (401, 403, 404, 429, 501, 502, 503, 504, ...) answer browsers with a clean HTML page, API clients with JSON and everyone else with text, always with the request ID and never with internal details; titles and pages are configurable. See [observability](observability.md#error-pages).

### OpenTelemetry

Traces, metrics and logs over OTLP/HTTP to any collector or vendor
(`global.tracing`): a SERVER span per request plus a CLIENT span per upstream attempt (retries
and backend latency visible, W3C `traceparent` continued), every Prometheus metric pushed as OTLP
metrics, access logs as OTLP log records carrying their trace and span ids, Kubernetes resource
attributes, custom resource attributes and auth headers for hosted backends. The endpoint and
switches change on reload, no restart.

### API discovery

Every routed request is folded into an endpoint (host, method, path
template): numeric ids, UUIDs, hashes, tokens and emails in paths become placeholders, and a
position that keeps taking new values (`/users/alice`, `/users/bob`, …) collapses into
`{param}` after 50 of them. Per endpoint: calls by status, latency, query parameter names (never
values), content types, the share of callers that sent credentials, and which kinds of sensitive
data (email, card numbers passing Luhn, JWTs, AWS keys, private keys, US SSNs) appeared in its
response bodies. Bodies are scanned only when uncompressed and text-like, for an endpoint's
first 20 responses and then about once a minute, up to 64 KiB each. Give it your OpenAPI or
Swagger specs (`global.discovery.specs`) and each endpoint is marked documented or **shadow**,
and spec operations nobody calls are listed. The dashboard has an API discovery panel;
`/api/discovery` is merged across replicas, and `/api/discovery/openapi?host=…` exports an
OpenAPI 3 document of what was observed. 404s are not recorded, so scanners probing random
paths do not create endpoints.

### Traffic anomaly detection

No external service and no AI calls: every route's requests
are counted per minute (rate, 5xx, upstream failure kinds, latency histogram, client country),
every replica merges all replicas' minutes, and per-route baselines (an hour-long moving mean
and variance) flag a **traffic spike** or **drop**, an **error spike**, a **latency spike**, a
**new error** (a 5xx code or upstream failure kind never seen on the route) and **traffic from a
new country**. Routes are judged after a warm-up (`global.anomaly.warmup`, 30m); a minute that a
replica did not report is skipped rather than learnt. Anomalies appear on the dashboard and at
`/api/anomalies`, as `lygard_anomalies_total{route,kind}`, and — from one replica only, with a
per-route/kind cooldown — at `global.anomaly.webhook` (Slack-compatible `text` plus the events).
Countries come from CDN headers (`CF-IPCountry`, `CloudFront-Viewer-Country`, …) on requests
from `trustedProxies`, or from a MaxMind GeoLite2 database (`global.anomaly.geoipDatabase`).

### Traffic capture and replay

Start a capture from the dashboard or `POST /api/captures`
(`{"filter": {"route": "api", "pathPrefix": "/orders", "minStatus": 500}, "maxRequests": 200,
"duration": "10m"}`) and every replica records the matching requests and responses, bodies up to
a limit. Credentials are masked as they are recorded: Authorization and API-key headers, cookie
values, sensitive query parameters and form/JSON fields (`password`, `token`, `secret`, …), and
JWTs, bearer tokens, AWS keys and private keys wherever they appear. Download it as HAR 1.2
(`/api/captures/<id>/har`), then replay it against staging or a new build — from the gateway
(`POST /api/captures/<id>/replay {"target": "http://api.staging.svc"}`, handy for in-cluster
targets) or anywhere with `lygard replay capture.har --target https://staging.example.com -H
'Authorization: Bearer <staging token>'`. Each response is compared with the recorded one
(status, and body where both are complete); the CLI exits non-zero on differences, so a capture
of last night's failures becomes a regression check. Masked credentials are never sent; pass the
target's own with `-H`. Nothing is buffered unless a capture matches the request.

## Kubernetes

### Policy as annotations or CRDs

Every route setting is an `lygard.io/*` annotation or a field
of a [`LygardPolicy`](policy.md) (typed, strictly validated, with status); a
ValidatingAdmissionPolicy flags unknown annotations at `kubectl apply` time.

### Drop-in for ingress-nginx

The common `nginx.ingress.kubernetes.io/*` annotations are
translated (rewrites, body size, timeouts, allow-lists, rate limits, CORS, basic and external
auth, backend protocol, affinity, ModSecurity/CRS); the rest are named in a warning with their
replacement. See [migrating from ingress nginx](migrating-from-ingress-nginx.md).

### Service mesh (preview)

`mesh.enabled=true` turns on Linkerd-style sidecars for namespaces
labelled `lygard.io/mesh=enabled`: an admission webhook injects an iptables init container and a
native sidecar; each pod exchanges its ServiceAccount token (TokenReview) for a 24 h certificate
from the mesh CA (`<sa>.<ns>.serviceaccount.identity.lygard.local`, SPIFFE URI included),
renewed automatically; connections between meshed pods are upgraded to TLS 1.3 mTLS with the
peer's identity verified (never downgraded), ClusterIP traffic is balanced in the sidecar, and
per-identity connection/byte metrics give a service graph. Plaintext from unmeshed clients is
accepted (permissive) or refused (`lygard.io/mesh-mode=strict`).

Outbound HTTP/1 and HTTP/2 (gRPC included) is detected and handled per request: every request
is balanced on its own (one long-lived gRPC connection no longer pins a single pod), GET/HEAD/
OPTIONS without a body are retried once on another endpoint after a connect failure or a 503,
and `lygard_mesh_requests_total` / `lygard_mesh_request_duration_seconds` count each request per
peer identity. WebSocket upgrades pass through. Other protocols are relayed as bytes; ports of
server-first protocols (MySQL, SMTP, …) are listed as opaque by default and can be extended per
pod with `lygard.io/opaque-ports`. A server-first protocol on an unlisted port still works but
waits one second per connection for detection.

Routing (Gateway API GAMMA): an HTTPRoute whose `parentRefs` names a Service (`kind: Service`,
`group: ""`) controls how meshed clients reach it: path, method, header and query matches,
weighted backends (canaries between Service versions), request/response header modifiers and
request timeouts, applied by the client's sidecar. A route in the Service's namespace applies to
every client; a route in another namespace (consumer route) applies to that namespace's clients
only and takes precedence for them. Route status reports `Accepted`/`ResolvedRefs` per Service
parent. Routing needs the HTTP-aware path above (not opaque ports or TLS from the app).

Authorization: annotate a Service with `lygard.io/mesh-allow: "web.shop, *.ops"` and only those
identities (`<sa>.<ns>`, `*.<ns>`, `*` for any meshed client, `unauthenticated` for plaintext
clients such as an ingress outside the mesh) may connect to its meshed endpoints. Everyone else
is refused after the handshake, an empty list admits no one, and tightening a policy also cuts
connections that are already open.

## Extending

### Proxy-wasm plugins

Envoy-compatible WebAssembly filters (ABI 0.2.1, any SDK: Rust, C++,
Go/TinyGo, AssemblyScript) run unmodified. Define them once in `global.plugins` (`file:` a
`.wasm` path or `module:` base64, `configuration:` handed to the plugin as-is or as JSON) and list
them on a route (`policy.plugins: [auth, tagger]`, or `lygard.io/plugins: "auth, tagger"` on an
Ingress/HTTPRoute). They run after authentication and the WAF, see and change request and
response headers (including `:path`, `:method`, `:authority`), can answer the request themselves,
read Envoy-style properties (`request.path`, `source.address`, `route_name`, `response.code`, …),
make HTTP calls and unary gRPC calls to a backend by name or to `host:port` (external auth
works), define metrics (`lygard_plugin_counter{plugin,name}` …), share data between requests,
pass messages through shared queues (`proxy_on_queue_ready` wakes the instance that registered
the queue), call Envoy's `compress`/`uncompress` foreign functions, and use ticks. Bodies are
buffered for plugins that set `requestBody` / `responseBody` (up to `maxBodyBytes`). Each plugin
runs as a pool of instances (`instances`, default 4) with a CPU deadline per callback
(`callTimeout`, 100ms — an infinite loop is stopped, not a hung request) and a memory cap
(`memoryLimitMb`); a trapped instance is replaced, and the request fails with 500 or, with
`failOpen`, continues without the plugin. gRPC streams are not supported (they answer
`Unimplemented`). `tests/wasm-filter` is a filter written against the standard Rust SDK that the
test suite runs.
