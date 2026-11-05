# Annotations

Route policy can be set three ways, which combine: annotations on an `Ingress`, `HTTPRoute` or
`GRPCRoute`; a [`LygardPolicy`](policy.md) that targets one; and (for Ingresses migrating from
ingress-nginx) the [`nginx.ingress.kubernetes.io/*` annotations](migrating-from-ingress-nginx.md).
When the same setting comes from more than one, `LygardPolicy` wins over `lygard.io/*`
annotations, which win over translated nginx ones.

A malformed annotation is reported (controller log and warning) and ignored; it never breaks the
other routes. With the chart's admission policy enabled (`validation.enabled`, default on), an
unknown `lygard.io/*` key is flagged by `kubectl apply` itself. A missing basic-auth Secret or an
unreachable external auth service makes the route refuse requests (fail closed).

## Routing and timeouts

| Annotation | Example | Meaning |
|---|---|---|
| `lygard.io/ssl-redirect`, `force-ssl-redirect` | `"false"` | Redirect HTTP to HTTPS (default: on when the host has TLS) |
| `lygard.io/rewrite-target` | `/` | Replace the matched path prefix |
| `lygard.io/request-timeout` | `30s` | Time to response headers, all attempts included |
| `lygard.io/retries` | `2` | Extra attempts (max 10) |
| `lygard.io/retry-on` | `connect-failure,reset,503` | Conditions: `connect-failure`, `reset`, `timeout`, `5xx`, or status codes. Status retries only for idempotent methods. Without any retry setting, a refused connection is retried once on another endpoint (any method) and a reset one for idempotent methods |
| `lygard.io/retry-per-try-timeout` | `2s` | Timeout per attempt |
| `lygard.io/request-headers-set`, `request-headers-remove` | `X-Env: prod; X-Team: a` | Request header changes |
| `lygard.io/response-headers-set`, `response-headers-remove` | `Cache-Control: no-store` | Response header changes |
| `lygard.io/error-page-<code>` | `<h1>Back soon</h1>` | Replace the body of every 4xx/5xx response with that code (upstream or Lygard's own). Status and headers such as `Retry-After` are kept; a body starting with `{` or `[` is served as JSON |

## Backends

| Annotation | Example | Meaning |
|---|---|---|
| `lygard.io/backend-protocol` | `https`, `h2c`, `grpc`, `h2`, `grpcs` | `https` = HTTP/1.1 over TLS, `h2` = HTTP/2 over TLS, `grpc` = h2c. Requests with a gRPC-Web content type are translated to gRPC automatically for `h2`/`h2c` backends |
| `lygard.io/backend-tls-insecure` | `"true"` | Skip backend certificate verification |
| `lygard.io/backend-sni` | `api.internal` | SNI / name verified on the backend certificate |
| `lygard.io/lb` | `least_request`, `round_robin`, `random`, `consistent_hash` | Load balancing (default `least_request`, power of two choices) |
| `lygard.io/hash-on` | `ip`, `header:x-user`, `cookie:session` | Key for `consistent_hash` |
| `lygard.io/session-cookie` | `SID` | Sticky sessions: Lygard issues this cookie naming the endpoint that served the first request and keeps the client there while it is available (also `nginx.ingress.kubernetes.io/affinity: cookie`) |
| `lygard.io/session-cookie-ttl` | `1h` | Session cookie lifetime; unset is a browser-session cookie |
| `lygard.io/connect-timeout` | `2s` | TCP/TLS connect timeout |
| `lygard.io/max-requests` | `1000` | Circuit breaker: concurrent requests per backend |
| `lygard.io/health-check-path`, `health-check-interval` | `/healthz`, `5s` | Active health checks |

## Security

| Annotation | Example | Meaning |
|---|---|---|
| `lygard.io/waf`, `waf-mode` | `detect`, `block`, `off` | WAF mode |
| `lygard.io/waf-engine` | `native`, `crs` | Native rules (fast default) or the OWASP Core Rule Set v4 |
| `lygard.io/waf-paranoia` | `1`-`4` | Paranoia level |
| `lygard.io/waf-threshold` | `5` | Anomaly score that blocks |
| `lygard.io/waf-exclusions` | `942100 args:comment /blog POST; * header:referer` | Targeted false-positive exclusions: `<ids or *> [args:x\|header:x\|cookie:x\|body\|path] [/prefix ...] [METHOD ...]`; the dashboard's tuning panel suggests them |
| `lygard.io/waf-exclude-rules`, `waf-exclude-args` | `942100`, `html_body` | Route-wide exclusions |
| `lygard.io/waf-inspect-body`, `waf-inspect-response` | `"true"` | Inspect request bodies / response bodies (CRS response phases) |
| `lygard.io/waf-skip-extensions` | `css, js, png, woff2` | Leave plain static-file fetches uninspected: GET/HEAD, no query string, no body ([cost](performance.md)) |
| `lygard.io/waf-rulesets` | `wordpress, php, sensitive-files` | Built-in rule packs or `waf.ruleGroups` to add ([waf-rules.md](waf-rules.md)) |
| `lygard.io/waf-disabled-categories` | `java, xss` | Attack categories switched off for this route, in either engine |
| `lygard.io/ip-deny-lists` | `spamhaus-drop, tor-exits` | Refuse addresses in named IP lists (presets or `waf.ipLists`) |
| `lygard.io/allow-cidrs`, `deny-cidrs` | `10.0.0.0/8, 192.168.0.0/16` | IP filtering |
| `lygard.io/geo-allow`, `geo-deny` | `DE, FR` | Country filtering (ISO 3166 alpha-2). Countries come from CDN headers set by `trustedProxies` (`CF-IPCountry`, `CloudFront-Viewer-Country`, ...) or a GeoLite2 database (`anomaly.geoipDatabase`). With `geo-allow`, an unknown country is refused |
| `lygard.io/ja4-deny` | `t13d1516h2_*` | Refuse TLS client fingerprints |
| `lygard.io/max-body-size` | `10m` | Request body limit |
| `lygard.io/security-headers` | `"false"` | Turn off the default security headers |
| `lygard.io/hsts-max-age`, `hsts-include-subdomains`, `frame-options`, `content-security-policy`, `referrer-policy` | | Response hardening |

## Authentication

See [authentication.md](authentication.md) for examples.

| Annotation | Example | Meaning |
|---|---|---|
| `lygard.io/auth-basic-secret`, `auth-realm` | `htpasswd` | Basic auth from a Secret labelled `lygard.io/secret=true` with bcrypt lines under `auth` |
| `lygard.io/jwt-issuer`, `jwt-audiences`, `jwt-jwks-url` | | JWT validation (HS/RS/PS/ES/EdDSA) |
| `lygard.io/jwt-from-cookie` | `session` | Also read the token from a cookie |
| `lygard.io/jwt-claims-to-headers` | `sub:x-user, email:x-email` | Forward claims to the backend |
| `lygard.io/jwt-required-claims` | `scope=orders:write; roles=admin\|ops` | A valid token must also carry these claims (403 otherwise). Arrays and space-separated `scope` strings match if they contain a value; dotted paths reach nested claims |
| `lygard.io/jwt-strip-token` | `"true"` | Remove `Authorization` before proxying |
| `lygard.io/auth-url` | `http://oauth2-proxy.auth.svc/oauth2/auth` | External (forward) auth: 2xx lets the request through, anything else is returned to the client |
| `lygard.io/auth-signin` | `https://app.example.com/oauth2/start` | On 401, redirect the browser here with `rd=<original URL>` |
| `lygard.io/auth-response-headers` | `X-Auth-Request-User, X-Auth-Request-Email` | Headers copied from the auth service's 2xx to the backend request |
| `lygard.io/auth-request-headers` | `Authorization, Cookie, X-Api-Key` | Client headers sent to the auth service (default `Authorization, Cookie`) |
| `lygard.io/auth-method` | `GET`, `original` | Method of the auth sub-request |
| `lygard.io/auth-timeout` | `2s` | Auth sub-request timeout |
| `lygard.io/auth-fail-open` | `"true"` | Let requests through when the auth service is unreachable (default: 503) |
| `lygard.io/cors-allow-origins` | `https://*.example.com` | CORS (the allowed origin is always echoed, never `*`) |
| `lygard.io/cors-allow-methods`, `cors-allow-headers`, `cors-expose-headers`, `cors-allow-credentials`, `cors-max-age` | | CORS details; `*` methods/headers echo what the browser asked for |

## Rate limiting

| Annotation | Example | Meaning |
|---|---|---|
| `lygard.io/rate-limit-rps`, `rate-limit-burst` | `50`, `100` | Token bucket |
| `lygard.io/rate-limit-key` | `ip`, `header:x-api-key`, `route` | Bucket key |
| `lygard.io/rate-limit-scope` | `cluster` (default), `local` | `cluster`: replicas exchange usage every 200 ms so the limit holds cluster-wide (approximately) |

## Observability and plugins

| Annotation | Example | Meaning |
|---|---|---|
| `lygard.io/access-log` | `"false"` | Per-route access log switch |
| `lygard.io/plugins` | `auth, tagger` | proxy-wasm plugins (defined in `global.plugins`) in order |

## Progressive rollouts (HTTPRoute, GRPCRoute)

| Annotation | Default | Meaning |
|---|---|---|
| `lygard.io/rollout-canary` | | The canary Service; the route's rules list it next to the stable one |
| `lygard.io/rollout-steps` | `5,25,50,100` | Traffic percentages |
| `lygard.io/rollout-interval` | `1m` | Minimum time per step |
| `lygard.io/rollout-min-requests` | `50` | Canary requests needed per step |
| `lygard.io/rollout-max-error-rate` | `1%` | Canary 5xx rate that rolls back |
| `lygard.io/rollout-max-latency-increase` | `50%` | Canary p95 above stable that rolls back |
| `lygard.io/rollout-paused`, `rollout-abort` | | Pause / roll back |
| `lygard.io/rollout-revision` | | Changing it starts a new rollout (e.g. set to the image tag in CI) |

## Certificates

| Annotation | Meaning |
|---|---|
| `lygard.io/acme: "true"` | Issue the Ingress's TLS Secrets through ACME (needs `acme.enabled`); HTTP-01, or DNS-01 for hosts a DNS solver covers and for wildcards ([certificates](certificates.md)) |
| `lygard.io/acme-solver` | Use this DNS-01 solver (by name) for every host of the Ingress, or `http01` to force HTTP-01 |
