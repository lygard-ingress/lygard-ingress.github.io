# Roadmap

What is planned, in rough order. It comes from Lygard's own test results and from a feature and
issue review of nine other gateways (Envoy Gateway, Istio, Contour, ingress-nginx, F5 NGINX
Ingress Controller, HAProxy, Traefik, Cilium, Apache APISIX). Want something moved up, or want to
build it? Open an issue.

## Now

- **Gateway API, remaining extended features**: `ListenerSet`, `TLSRoute` Terminate mode,
  Gateway infrastructure propagation (generated resources in the Gateway's namespace), static
  addresses, Gateway client certificates to backends, session persistence (GEP-1619).
- **Upstream conformance report**: submit the v1.6 report to kubernetes-sigs/gateway-api.
- **Cloud validation**: run the rollout and node-drain tests through real EKS, AKS, GKE, DOKS and
  OKE load balancers with the shipped presets (Linode is validated today).
- **Soak and chaos**: 72-hour runs with pod, node and controller failures under load.
- **Incremental configuration**: send changes instead of full documents to the proxies (full
  documents are already gzip-compressed).

## Next

- **ACME External Account Binding** (CAs such as ZeroSSL and Google Trust Services), and more
  DNS-01 providers natively (OVH, Hetzner, Gandi, ...; any provider works today through the
  webhook solver).
- **Dashboard SSO** (OIDC) and roles for the admin API (a read-only token exists today).
- **Authorization policies**: CEL/OPA rules on routes beyond JWT claims, IP and country lists.
- **Built-in OIDC login** (today: forward auth to oauth2-proxy and similar).
- **Per-client connection limits** and response compression (gzip/brotli/zstd).
- **OWASP CRS**: faster evaluation of request bodies (a literal prefilter across rules), managed
  rule updates, per-rule metrics in the dashboard.
- **Public, reproducible competitor benchmark** with one harness for every gateway.

## Done recently

- **HTTP/3 (QUIC)** on HTTPS listeners, with Alt-Svc, graceful drain and UDP Service ports from
  the chart and the controller ([docs/http3.md](http3.md)).
- **Memory per connection**: an idle HTTP/1 keep-alive connection holds no I/O buffers; 35 → 11 KB
  per proxied connection ([docs/benchmarks.md](benchmarks.md)).
- **OpenTelemetry**: protobuf (default) and gzip export, shared log records, log sampling and
  batching settings; 100% sampling with logs costs ~5 µs per request ([docs/performance.md](performance.md)).
- **WAF**: matches recorded off the request path, cheaper CRS evaluation, `skipExtensions` for
  static files, and the cost of every setting measured ([docs/performance.md](performance.md)).
- **proxy-wasm**: unary gRPC calls, shared queues, `compress`/`uncompress` foreign functions.
- **Service mesh routing (GAMMA)**: HTTPRoutes attached to Services: matches, weights, header
  filters, timeouts, producer and consumer routes.
- **ACME DNS-01 and wildcard certificates**: Cloudflare, Route 53, Google Cloud DNS, Azure DNS,
  DigitalOcean, Linode, RFC 2136 and a webhook solver; CNAME delegation of `_acme-challenge`
  ([docs/certificates.md](certificates.md)).

## Later

- **Exact global rate limits** with an optional Redis/Valkey store.
- **proxy-wasm**: gRPC streams, signed plugin artifacts, ext_proc.
- **Service mesh**: the GAMMA conformance profile (MESH-HTTP), ReferenceGrant for cross-namespace
  mesh backends, retries and redirects in mesh routes; a node-level (ambient-style) proxy.
- **Persistence** of API discovery, anomaly baselines and WAF tuning across restarts.
- **Operational extras**: response caching, fault injection, zone-aware routing, PROXY protocol
  to backends, an OperatorHub bundle, multi-cluster.
- **FIPS 140** build and a third-party security audit.
