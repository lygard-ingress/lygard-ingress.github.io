# Changelog

All notable changes. Versions follow [Semantic Versioning](https://semver.org/); before 1.0,
minor versions may change behaviour (called out under **Changed**).

## Unreleased

### Added

- **ListenerSet** (Gateway API v1.6): teams add listeners (hostnames, certificates, ports) to a
  shared Gateway without editing it. The Gateway opts in with `spec.allowedListeners`; set
  listeners follow the Gateway's own, oldest set first, and one that collides with an earlier
  listener on its port is `Conflicted` (hostname or protocol) instead of taking traffic. Routes
  attach to a ListenerSet with a `parentRef` of kind `ListenerSet`. Certificates referenced from a
  set default to its namespace and need a ReferenceGrant from `ListenerSet` across namespaces.
- **TLSRoute Terminate mode**: a Gateway `TLS` listener with `tls.mode: Terminate` terminates TLS
  with its certificates and relays the decrypted stream to the TLSRoute's backend (databases and
  other TCP services that should not handle TLS themselves). Passthrough and Terminate listeners
  can share a port (Mixed). Standalone: `terminate: true` on a TLS route.
- **Gateway `spec.infrastructure` and `spec.addresses`** (per-gateway mode): the Gateway's
  infrastructure labels and annotations are added to its load balancer Service (its annotations
  override the inherited ones), and an IP address becomes the Service's `loadBalancerIP`. The
  Gateway is `AddressNotUsable` until the load balancer has the requested IP; hostname addresses
  are rejected with `UnsupportedAddress`
  ([docs/cloud-providers.md](cloud-providers.md#one-load-balancer-per-gateway)).
- **Chaos tests** (`tests/chaos/run.sh`): failure injection under load in kind. Graceful and forced
  data plane pod deletion, the controller leader killed (and a route created after), a data plane
  restart with no controller running, and a worker node stopped. On Kubernetes 1.37: no failed
  requests in any scenario except the node stop (1 of 12,000, a connection in flight on the lost
  node).
- **Gateway client certificate to backends** (GEP-3155): a Gateway's
  `spec.tls.backend.clientCertificateRef` (a `kubernetes.io/tls` Secret, cross-namespace with a
  ReferenceGrant) is presented to backends Lygard reaches over TLS, for example through a
  BackendTLSPolicy, so backends that require mutual TLS work. Plaintext backends are unaffected; a
  route attached to Gateways with different client certificates uses none and logs a warning.
  Standalone: backend `tlsClientCertPem` / `tlsClientKeyPem`.
- **Session persistence (sticky sessions)**: Lygard issues a cookie naming the endpoint that served a
  client and routes the client's later requests there while it is available; when it is gone the
  client moves and gets a new cookie. Backend `sessionPersistence` (standalone), HTTPRoute rule
  `sessionPersistence` (GEP-1619, cookie type), and `lygard.io/session-cookie` /
  `session-cookie-ttl`. ingress-nginx `affinity: cookie` now issues the cookie too (it used to hash
  an existing one), including `session-cookie-max-age`/`expires`. The cookie value is a hash, not
  the endpoint address.
- **Akamai Cloud Quick Deploy App** (`deploy/marketplace/linode-marketplace-lygard`): a hardened
  Ubuntu 24.04 Linode (sudo user, SSH lockdown, firewall, fail2ban, automatic security updates)
  running Lygard from its image under systemd, with automatic HTTPS, the OWASP CRS WAF (detect or
  block) and the dashboard over HTTPS on port 9443. Deployed end to end on Linode.
- **Automatic HTTPS for standalone installs** (`global.acme`): the data plane gets and renews
  Let's Encrypt (or any ACME CA) certificates over HTTP-01, caches them on disk and serves them
  without a restart ([docs/certificates.md](certificates.md#standalone-automatic-https)).
  Tried against Let's Encrypt on a Linode VM: issued in ~5 s, served from cache after a restart.
- **Behind a cloud layer 7 load balancer**: presets `aws-alb.yaml` (Application Load Balancer),
  `azure-appgw.yaml` (Application Gateway v2 with AGIC), `azure-agc.yaml` (Application Gateway for
  Containers), `gcp-l7.yaml` (GKE Gateway, global external Application Load Balancer) and
  `oke-l7.yaml` (OCI Native Ingress Controller). The chart's new `frontend` values create the
  cloud's Ingress or Gateway pointing at Lygard, plus its health-check and backend policies
  ([docs/cloud-providers.md](cloud-providers.md#behind-a-layer-7-load-balancer)).
- `linode-l7.yaml`: the NodeBalancer in HTTP/HTTPS mode, ending TLS with a certificate from a
  Secret; tried on LKE. New `service.httpsTargetPort` sends a load balancer's decrypted 443 traffic
  to Lygard's HTTP listener.
- `global.trustedHops`: X-Forwarded-For entries a trusted load balancer appends after the
  client's (1 for Google Cloud).

### Changed

- **Dashboard**: a sign-in screen when the admin token is missing or wrong (where to find it,
  "locked" status) instead of empty panels; a section bar (Overview, Traffic, Security, API, Tools,
  Configuration); WAF events grouped by request and rules with a count and an "attacks only"
  filter; the page no longer overflows the window when tables are wide.
- The WAF event buffer evicts minor events (score under 5, not blocked) first, so a noisy rule
  cannot push real attacks out of `/api/waf/events`.
- The OWASP CRS engine uses 22-24% less CPU per request (GET, all features on); libinjectionrs is
  vendored with a faster keyword lookup. `multiMatch` rules no longer run their operator again on
  transformation steps that leave the value unchanged: the engine's time on a 1 KB JSON body drops
  a further 22-26% (942100, libinjection on every argument, was the most expensive rule).
  Detection is unchanged.

### Security

- **The admin API now fails closed without a token.** With no `adminToken`, the dashboard and
  `/api/*` (including replacing the configuration) were open on whatever address the admin
  server listened on. They are now open only on a loopback admin address; elsewhere they answer
  401 and Lygard logs a warning at startup. `/metrics`, `/healthz` and `/readyz` are unchanged.
  The Helm chart always sets a token, so Kubernetes installs are not affected.
- Dependency advisories cleared (wasmtime, webpki-roots; unmaintained rustls-pemfile and
  serde_yaml replaced); CI now runs `cargo audit` and `govulncheck`; the controller builds with
  Go 1.27.3.

- **ExternalName Services are off by default** as backends (`controller.allowExternalNameServices`):
  anyone able to create a Service could point the proxy at cloud metadata (169.254.169.254), the
  node's loopback (including the admin port) or another namespace's Services. When enabled,
  loopback, link-local and other namespaces' in-cluster names are still refused.
- **Forged identity headers**: headers filled from JWT claims or the external auth service are
  removed from the client's request first. Before, a token without the claim (or a session the
  auth service answered without the header) let a client-supplied value reach the backend.
- **External auth judged the raw path**: it now sees the normalized path the backend receives,
  so `/public/../admin` is no longer approved as `/public/...`.
- **Path normalization** decodes percent-encoded unreserved characters (`/%61dmin` is `/admin`)
  and treats `..;` / `.;` as dot segments, so path-scoped policies cannot be bypassed that way.
- **Slow bodies**: `global.bodyReadTimeout` (30 s) bounds the part of a request body Lygard
  buffers before proxying; new optional `global.maxConnectionsPerIp`.
- **CORS**: the `null` origin matches only when listed; `allowOrigins: ["*"]` with
  `allowCredentials` is warned about.
- `waf.oversizeAction: reject` refuses bodies larger than the WAF inspects (default `pass`, as
  before; truncated inspections are now flagged `WAFT` in the access log).
- `/api/config` also redacts credential-like header values, plugin configurations and URL
  passwords.
- New [docs/hardening.md](hardening.md).

### Fixed

- Behind a load balancer that ends TLS, `httpsRedirect` redirected every request (a loop) and
  HSTS was never sent: Lygard now takes the scheme from `X-Forwarded-Proto` when the peer is in
  `trustedProxies`, for redirects, HSTS, request captures, external auth and plugins.
- X-Forwarded-For entries with a port (`203.0.113.7:51234`, sent by Azure Application Gateway)
  were ignored, so the client appeared as the load balancer.

## 0.13.1 — 2026-11-02

### Fixed

- **`idleTimeout` was not applied to HTTP/1.1 keep-alive connections.** They were closed after
  `headerReadTimeout` (15 s) without a request, not after `idleTimeout` (120 s), because hyper
  starts the header timer while waiting for the next request. Clients that reuse connections less
  often than every 15 s (and load balancers pooling to Lygard) paid a new TCP and TLS handshake.
  Now the wait between requests is bounded by `idleTimeout` and `headerReadTimeout` starts with a
  request's first byte; slow-header protection is unchanged. HTTP/2 connections are still kept
  open by pings until the client closes them or `maxConnections` is reached.

## 0.13.0 — 2026-11-02

### Added

- **HTTP/3 (QUIC)**: `http3: true` on an HTTPS listener serves HTTP/3 on UDP at the same address
  and port, with the same certificates, routes and policies, and advertises it with `Alt-Svc` on
  HTTP/1.1 and HTTP/2 responses (`http3AdvertisePort` when the load balancer maps ports). 0-RTT
  is off. Graceful drain sends GOAWAY; listener changes keep the UDP socket. New metrics
  `lygard_quic_connections_total`, `lygard_quic_active_connections`,
  `lygard_quic_handshake_errors_total`. Helm: `http3.enabled` adds UDP 443 to the data plane
  Service; the controller adds UDP ports for Gateway HTTPS listeners on other ports and on
  per-Gateway Services. Enabled in the Azure, GCP, OKE and MetalLB presets
  ([docs/http3.md](http3.md)).
- **Gateway API GAMMA (service mesh routing)**: HTTPRoutes whose parent is a Service steer meshed
  clients: path/method/header/query matches, weighted backends, request/response header
  modifiers, request timeouts; producer routes (all clients) and consumer routes (one
  namespace's clients, preferred by them); `Accepted`/`ResolvedRefs` status per Service parent.
- **proxy-wasm**: unary gRPC calls (`proxy_grpc_call`, cancel/close), shared queues (register,
  resolve, enqueue, dequeue, `proxy_on_queue_ready`), and the `compress`/`uncompress` foreign
  functions.
- **OpenTelemetry**: protobuf export (default; `protocol: json` keeps JSON), `compression: gzip`,
  `logSampling` (`all`/`sampled`/`errors`), `batchSize`, `exportInterval`, `queueSize`.
- **WAF**: `skipExtensions` / `lygard.io/waf-skip-extensions` leaves plain static-file fetches
  uninspected; `lygard_waf_events_dropped_total`.
- [docs/performance.md](performance.md): the measured cost of every feature and setting;
  `bench/run.py` scenarios for each; `tests/soak/` for long-running soak tests.

### Changed

- An idle HTTP/1 keep-alive connection holds no I/O buffers (hyper carried with a small patch in
  `third_party/hyper`): 35 → 11 KB per proxied connection on Linux. `presets/high-scale.yaml`
  allows 200,000 connections per pod.
- OpenTelemetry: access log records are shared between stdout and OTLP instead of being
  converted per request; 100% sampling with logs costs ~5 µs per request (was ~21 µs).
  `lygard_trace_spans_exported_total` counts spans (it counted batches).
- WAF matches are recorded by a background thread; matching requests no longer wait on shared
  locks. CRS: allocation-free transformations when nothing changes, captures only on a match.

## 0.12.1 — 2026-11-02

### Fixed

- The native WAF engine panicked on a request whose text had an `&` followed within 12 bytes by
  a multi-byte UTF-8 character, which dropped that connection (found by the soak test). Any
  client could trigger it on routes with `waf` enabled (native engine).

## 0.12.0 — 2026-11-01

First public release.

### Added

- **Gateway API v1.6.3**: types, CRDs and conformance; HTTPRoute `retry` (GEP-1731); TCPRoute
  and UDPRoute at v1 (v1alpha2 still read on older CRD bundles).
- **Gateway API features**: CORS filter; request and backend-request timeouts; 303/307/308
  redirects; per-backend request header filters; HTTP 421 for misdirected (coalesced) requests;
  BackendTLSPolicy `subjectAltNames` (DNS and URI); frontend client-certificate validation
  (`AllowValidOnly`, `AllowInsecureFallback`, per-port overrides, status conditions).
- **External (forward) auth**: `lygard.io/auth-url`, `auth-signin`, `auth-response-headers`, ...;
  compatible with ingress-nginx and Traefik ForwardAuth, so oauth2-proxy/Authelia/Pomerium work.
- **JWT required claims** (`lygard.io/jwt-required-claims`): 403 when a valid token lacks them.
- **Country filtering** (`lygard.io/geo-allow`, `geo-deny`).
- **Custom error pages** (`lygard.io/error-page-<code>`).
- **gRPC-Web** translation to gRPC for HTTP/2 backends (binary and text).
- **LygardPolicy** CRD (`lygard.io/v1alpha1`): typed route policy with strict validation and status.
- **ingress-nginx compatibility**: `nginx.ingress.kubernetes.io/*` annotations translated, with a
  warning naming the replacement for each unsupported one.
- **ValidatingAdmissionPolicy** flagging unknown `lygard.io/*` annotations (Warn or Deny).
- **Read-only viewer token** for the dashboard and admin API.
- **Progressive rollouts for GRPCRoute**.
- **Helm**: presets for AWS (incl. EKS Auto Mode), Azure, GCP, DigitalOcean, Linode (+ Premium
  NodeBalancer), Oracle OKE, Hetzner, MetalLB and OpenShift, plus `light` and `high-scale`;
  `extraVolumes`/`extraVolumeMounts`; `service.loadBalancerClass`, `service.nodePorts`, `sysctls`.
- **Release engineering**: CI (fmt, clippy, tests, chart lint), Kubernetes 1.31–1.37 matrix and
  conformance in kind, an A/B performance gate, and signed multi-arch releases with SBOM and
  SLSA provenance.
- **ACME DNS-01 and wildcard certificates**: `acme.dns01.solvers` configures DNS providers by
  zone (longest match wins): Cloudflare, AWS Route 53 (static keys or IRSA web identity),
  Google Cloud DNS (service account key or Workload Identity), Azure DNS (service principal or
  Workload Identity), DigitalOcean, Linode, RFC 2136 (TSIG) and a generic webhook. Hosts in a
  covered zone and every `*.` host are validated with DNS-01; the controller follows a CNAME on
  `_acme-challenge` and waits until all of the zone's authoritative nameservers serve the record
  (`acme.dns01.propagationTimeout`, `acme.dns01.nameservers`). `lygard.io/acme-solver` on an
  Ingress or Gateway listener picks a solver by name (or `http01`). See
  [docs/certificates.md](certificates.md).
- **Helm**: `controller.serviceAccountAnnotations`, `controller.podLabels` and `controller.env`
  (cloud workload identity for DNS providers).

### Changed

- Draining data planes answer HTTP/1 with `Connection: close`; `shutdownDelay` defaults to 15 s.
- Without a retry policy, a refused connection is retried once on another endpoint, and a reset
  one for idempotent methods.
- CORS responses always echo the request's origin (never `*`), and `*` methods/headers echo what
  the browser asked for.
- The controller requires Go 1.26 to build; images move to `ghcr.io/lygard-ingress/*`.
- Configuration is sent to the data plane gzip-compressed.
- Wildcard TLS hosts with `lygard.io/acme: "true"` are no longer skipped with a warning: they are
  issued through DNS-01, or reported as `CertificateFailed` on the Ingress/Gateway when no
  DNS-01 solver covers them. A Gateway HTTPS listener asking for ACME no longer needs an HTTP
  listener when its host is validated with DNS-01.

### Fixed

- Ingress routes and static routes were also served on per-Gateway load balancers.
- A UDP listener made the Linode cloud controller reject the whole Service (PROXY protocol on UDP).
- WAF tuning suggestions only reflected one replica.
- `minReadySeconds` / `terminationGracePeriodSeconds` could not be set to 0.
- Changing a listener's settings on Linux could reset a few connections: the listening sockets
  are now handed to the new accept loops instead of being closed and rebound.
- JWT validation used jsonwebtoken 9, affected by GHSA-h395-gr6q-cpjc (a claim of the wrong JSON
  type skipped `exp`/`nbf` checks); now jsonwebtoken 11.
- `lygard.io/acme` no longer produces an "unknown annotation" warning on Ingresses.
