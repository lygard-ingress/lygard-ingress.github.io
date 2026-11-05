# Configuration reference

Every setting of the data plane configuration: the YAML file of a standalone install, and in Kubernetes the chart's `config:` values (`config.global`, `config.waf`, `config.listeners`, ...). Route policy (the [Policy](#policy) section) is what an Ingress's `lygard.io/*` annotations and a LygardPolicy's `spec.policy` set; in a standalone file it is each route's `policy` and `global.defaultPolicy`.

Durations are written like `30s`, `500ms`, `5m`, `1h`. Unknown fields are rejected, so a typo is an error rather than a silently ignored setting.

This page is generated from the source (`tools/docs-site/config_reference.py`); CI fails when it is out of date.

## Config


| Field | Type | Default | Description |
|---|---|---|---|
| `global` | [Global](#global) | `{"acceptors": 0, "accessLog": true, "accessLo…` |  |
| `listeners` | list of [Listener](#listener) | – |  |
| `routes` | list of [Route](#route) | – |  |
| `backends` | list of [Backend](#backend) | – |  |
| `certificates` | list of [Certificate](#certificate) | – |  |
| `tlsRoutes` | list of [TlsRoute](#tlsroute) | – | SNI-routed TLS passthrough: the encrypted stream is forwarded untouched. |
| `l4Routes` | list of [L4Route](#l4route) | – | Layer-4 routes for `tcp` / `udp` listeners (Gateway API TCPRoute / UDPRoute). |
| `waf` | [WafGlobal](#wafglobal) | `{"crsPath": "/usr/share/lygard/crs", "customR…` |  |

## Global

At `global`.

| Field | Type | Default | Description |
|---|---|---|---|
| `adminAddress` | string | `127.0.0.1:9901` | Admin server: /metrics, /healthz, /readyz, the dashboard and the JSON API. |
| `adminToken` | string | – | When set, the dashboard and /api/* require `Authorization: Bearer <token>`. /metrics, /healthz and /readyz stay open so probes and scrapers need no secret. |
| `viewerToken` | string | – | Read-only dashboard/API access (GET only; never /api/config or captures). Needs admin_token. |
| `accessLog` | bool | `true` |  |
| `accessLogQuery` | bool | `false` | Query strings often carry tokens; they are left out of logs unless asked for. |
| `serverHeader` | string | `lygard` | `Server` response header. Empty removes it. |
| `requestHistory` | integer | `4096` | Requests kept per replica for the request inspector (`/api/requests/<request or trace id>`): every decision taken for them. About 0.5 KB each; 0 turns the inspector off. |
| `trustedProxies` | list of CIDR | – | Peers whose X-Forwarded-For and X-Forwarded-Proto are believed. Anyone else's X-Forwarded-For is replaced, never appended to. |
| `trustedHops` | integer | `0` | X-Forwarded-For entries a trusted load balancer appends after the client's address (its own, e.g. 1 for Google Cloud's Application Load Balancers): skipped before looking for the client. Only applies when the peer is in `trusted_proxies`. |
| `maxConnections` | integer | `50000` |  |
| `maxConnectionsPerIp` | integer | `0` | Open connections one client address may hold across all listeners; 0 is unlimited. Counted on the TCP peer, so leave it at 0 behind a load balancer that does not keep client addresses (peers in `trusted_proxies` are never limited). |
| `acceptors` | integer | `0` | Accept sockets per listener (SO_REUSEPORT). 0: one per core up to 8 on Linux, 1 elsewhere. |
| `headerReadTimeout` | duration | `15s` |  |
| `idleTimeout` | duration | `2m` |  |
| `bodyReadTimeout` | duration | `30s` | Time allowed to receive the part of a request body Lygard holds before proxying (WAF inspection, plugins, mirrors); slower clients get 408. |
| `tlsHandshakeTimeout` | duration | `10s` |  |
| `maxHeaderBytes` | integer | `65536` |  |
| `http2MaxConcurrentStreams` | integer | `256` |  |
| `shutdownDelay` | duration | `5s` | Readiness turns false this long before listeners close, so load balancers stop sending first. |
| `drainTimeout` | duration | `30s` |  |
| `tracing` | [TracingConfig](#tracingconfig) | `{"batchSize": 512, "compression": "none", "ex…` |  |
| `errorPages` | [ErrorPagesConfig](#errorpagesconfig) | `{"format": "auto", "pages": {}, "title": ""}` | How Lygard's own error responses (403 from the WAF, 429, 502, 503, ...) look. |
| `challenge` | [ChallengeConfig](#challengeconfig) | `{"difficulty": 18, "passTtl": "30m", "provide…` | The challenge served by WAF rules with `action: captcha`. |
| `discovery` | [DiscoveryConfig](#discoveryconfig) | `{"enabled": true, "maxEndpoints": 5000, "scan…` |  |
| `anomaly` | [AnomalyConfig](#anomalyconfig) | `{"alertCooldown": "15m", "countryHeaders": ["…` |  |
| `plugins` | list of [PluginSpec](#pluginspec) | – | proxy-wasm plugins routes can use by name (policy `plugins`). |
| `peers` | list of string | – | Admin addresses (`ip:port`) of the other replicas, for cluster-wide rate limits. Filled in by the controller; this replica's own address is ignored. |
| `defaultPolicy` | [Policy](#policy) | – | Applied to every route; a route's own policy overrides field by field. |
| `acme` | [Acme](#acme) | – | Automatic HTTPS for standalone installs: certificates from an ACME CA (Let's Encrypt by default) for `hosts`, through HTTP-01 challenges answered on the HTTP listeners. In Kubernetes the controller issues certificates instead. |

## TracingConfig

At `global.tracing`.

| Field | Type | Default | Description |
|---|---|---|---|
| `otlpEndpoint` | string | – | OTLP/HTTP collector base URL, e.g. `http://otel-collector:4318`. Unset disables export; trace context is still propagated and logged. Changes apply without a restart. |
| `serviceName` | string | `lygard` |  |
| `sampleRatio` | number | `0.1` | Head sampling for requests that arrive without a sampled parent. |
| `headers` | map of string | – | Sent with every OTLP request, e.g. `authorization` for a hosted collector. |
| `resourceAttributes` | map of string | – | Added to the OTLP resource, e.g. `deployment.environment: prod`. |
| `exportMetrics` | bool | `true` | Push every Prometheus metric as OTLP metrics (cumulative) every `metricsInterval`. |
| `metricsInterval` | duration | `30s` |  |
| `exportLogs` | bool | `false` | Also send access log records as OTLP logs, carrying their trace and span ids. |
| `upstreamSpans` | bool | `true` | One CLIENT span per upstream attempt, so retries and backend latency show in traces. |
| `protocol` | `protobuf` \| `json` | `protobuf` | Wire format: `protobuf` (default, several times cheaper to encode) or `json`. |
| `compression` | `none` \| `gzip` | `none` | `gzip` trades CPU for bandwidth (worth it for a collector across a WAN); default `none`. |
| `logSampling` | `all` \| `sampled` \| `errors` | `all` | Which access log records become OTLP logs: `all`, `sampled` (only requests whose trace is sampled, so logs and traces match) or `errors` (status >= 500 or a response flag). |
| `traceLink` | string | – | Link from the request inspector to your tracing UI; `{traceId}` is replaced, e.g. `https://grafana.example.com/explore?left={"queries":[{"query":"{traceId}"}]}`. |
| `batchSize` | integer | `512` | Records per OTLP request, and the longest a record waits before being sent. |
| `exportInterval` | duration | `2s` |  |
| `queueSize` | integer | `8192` | Spans and log records held while the collector is slow; beyond it they are dropped and counted. Read at start-up. |

## ErrorPagesConfig

At `global.errorPages`.

| Field | Type | Default | Description |
|---|---|---|---|
| `format` | `auto` \| `html` \| `json` \| `text` | `auto` |  |
| `title` | string | – | Shown as the page heading's prefix, e.g. your product name. Empty: the host name. |
| `pages` | map of [ErrorPage](#errorpage) | – | Your own pages for Lygard's errors, by status ("403", "503", ...). `{status}`, `{reason}`, `{message}`, `{requestId}` and `{traceId}` are filled in. Route `errorPages` win over these. |

## ErrorPage

At `global.errorPages.pages.<name>`.

| Field | Type | Default | Description |
|---|---|---|---|
| `body` | string | – |  |
| `contentType` | string | `text/html; charset=utf-8` |  |

## ChallengeConfig

At `global.challenge`.

| Field | Type | Default | Description |
|---|---|---|---|
| `provider` | `pow` \| `turnstile` \| `hcaptcha` \| `recaptcha` | `pow` | `pow`: Proof of work in the browser: no third party, no tracking; `turnstile`; `hcaptcha`; `recaptcha`. |
| `difficulty` | integer | `18` | Proof of work: leading zero bits to find (each one doubles the work). 18 takes a phone about half a second. |
| `passTtl` | duration | `30m` | How long a solved challenge is valid for this client and host. |
| `siteKey` | string | – | Turnstile / hCaptcha / reCAPTCHA site key (public). The secret comes from the `LYGARD_CAPTCHA_SECRET` environment variable. |

## DiscoveryConfig

At `global.discovery`. API discovery: endpoints learnt from traffic (served at `/api/discovery`).

| Field | Type | Default | Description |
|---|---|---|---|
| `enabled` | bool | `true` |  |
| `maxEndpoints` | integer | `5000` | Endpoints remembered per replica; further new ones are counted as dropped. |
| `scanResponses` | bool | `true` | Look for emails, card numbers, tokens and keys in (uncompressed, text) response bodies. Each endpoint is scanned for its first responses, then about once a minute. |
| `scanBytes` | integer | `65536` | Bytes of each scanned response body examined. |
| `specs` | list of [ApiSpec](#apispec) | – | The APIs' own specs, to find shadow (undocumented) and unused endpoints. |

## ApiSpec

At `global.discovery.specs[]`.

| Field | Type | Default | Description |
|---|---|---|---|
| `hosts` | list of string | – | Hosts the spec describes; `*` for any, `*.example.com` for subdomains. |
| `file` | string | – | OpenAPI 3 or Swagger 2, JSON or YAML: a file path, or the document inline. |
| `document` | string | – |  |
| `basePath` | string | – | Prefixed to every spec path. Default: the spec's `basePath` or first server URL's path. |

## AnomalyConfig

At `global.anomaly`. Traffic anomaly detection: per-route baselines, flagged minutes at `/api/anomalies`.

| Field | Type | Default | Description |
|---|---|---|---|
| `enabled` | bool | `true` |  |
| `warmup` | duration | `30m` | Minutes of traffic a route needs before it is judged. |
| `sensitivity` | number | `4.0` | Standard deviations from the baseline that count as unusual. |
| `minRequests` | integer | `20` | Requests a minute must have before rates and latency are judged. |
| `minErrors` | integer | `5` | Failures a minute must have before an error spike or a new error kind is reported. |
| `webhook` | string | – | POSTed for each new anomaly: Slack-compatible `text` plus the structured events. |
| `webhookHeaders` | map of string | – |  |
| `alertCooldown` | duration | `15m` | The same route and kind is not alerted again within this. |
| `countryHeaders` | list of string | `["cf-ipcountry", "cloudfront-viewer-country",…` | Headers a CDN or load balancer sets to the client's ISO country code; honoured only on requests arriving from `trustedProxies`. |
| `geoipDatabase` | string | – | MaxMind GeoLite2/GeoIP2 Country or City database, for countries without such a header. |

## PluginSpec

At `global.plugins[]`. A proxy-wasm module (Envoy-compatible WASM filter).

| Field | Type | Default | Description |
|---|---|---|---|
| `name` | string | – |  |
| `file` | string | – | Path to the `.wasm` file… |
| `module` | string | – | …or the module itself, base64-encoded. |
| `configuration` | `serde_json::Value` | – | Handed to the plugin as its configuration: a string as-is, anything else as JSON. |
| `vmConfiguration` | string | – |  |
| `rootId` | string | – |  |
| `instances` | integer | `4` | Instances in the pool (Envoy runs one per worker thread). |
| `requestBody` | bool | `false` | Buffer request / response bodies (up to `maxBodyBytes`) and run the body callbacks. |
| `responseBody` | bool | `false` |  |
| `maxBodyBytes` | integer | `1048576` |  |
| `failOpen` | bool | `false` | On a plugin failure, carry on without it instead of answering 500. |
| `callTimeout` | duration | `100ms` | CPU time one callback may take before it is stopped. |
| `memoryLimitMb` | integer | `64` |  |

## Policy

At `global.defaultPolicy`. Every field optional so a route policy can override the default policy one field at a time.

| Field | Type | Default | Description |
|---|---|---|---|
| `requestTimeout` | duration | – |  |
| `retry` | [RetryPolicy](#retrypolicy) | – |  |
| `rateLimit` | [RateLimit](#ratelimit) | – |  |
| `waf` | [WafPolicy](#wafpolicy) | – |  |
| `ipAllow` | list of CIDR | – |  |
| `ipDeny` | list of CIDR | – |  |
| `ipDenyLists` | list of string | – | Named IP lists (`waf.ipLists`, or a preset such as `spamhaus-drop`) whose addresses get 403. |
| `basicAuth` | [BasicAuth](#basicauth) | – |  |
| `jwt` | [JwtPolicy](#jwtpolicy) | – |  |
| `cors` | [Cors](#cors) | – |  |
| `securityHeaders` | [SecurityHeaders](#securityheaders) | – |  |
| `maxBodyBytes` | integer | – |  |
| `httpsRedirect` | bool | – | Requests that arrive on a plaintext listener get a 308 to https. |
| `accessLog` | bool | – |  |
| `ja4Deny` | list of string | – | JA4 TLS fingerprints refused with 403. `*` matches any suffix, e.g. `t13d1516h2_*`. |
| `plugins` | list of string | – | proxy-wasm plugins (from `global.plugins`) the request runs through, in order. |
| `geoAllow` | list of string | – | Only clients from these countries (ISO 3166 alpha-2) get through; unknown country is refused. |
| `geoDeny` | list of string | – | Clients from these countries get 403; unknown country passes. |
| `extAuth` | [ExtAuth](#extauth) | – | Forward auth: ask an external service before proxying (oauth2-proxy, Authelia, ...). |
| `errorPages` | map of [ErrorPage](#errorpage) | – | Replace the body of responses with these status codes ("404", "503", ...). |

## RetryPolicy

At `global.defaultPolicy.retry`.

| Field | Type | Default | Description |
|---|---|---|---|
| `attempts` | integer | `2` |  |
| `on` | list of string | `["connect-failure", "502", "503"]` | `connect-failure`, `5xx`, `502`, `503`, `504`, `reset`, `timeout`. |
| `perTryTimeout` | duration | – |  |

## RateLimit

At `global.defaultPolicy.rateLimit`.

| Field | Type | Default | Description |
|---|---|---|---|
| `rps` | number | `100.0` | Sustained requests per second per key. |
| `burst` | integer | `200` |  |
| `key` | string | `ip` | `ip` (default), `header:<name>`, `route` (one bucket for everyone). |
| `scope` | string | `cluster` | `cluster` (default): replicas share usage, so the limit holds cluster-wide. `local`: each replica limits on its own. |

## WafPolicy

At `global.defaultPolicy.waf`.

| Field | Type | Default | Description |
|---|---|---|---|
| `mode` | `off` \| `detect` \| `block` | `off` | `off`; `detect`: Score and log, never block. The way to roll a WAF out without breaking anyone; `block`. |
| `engine` | string | `native` | `native` (Lygard's own rules) or `crs` (OWASP Core Rule Set via the SecLang engine). |
| `paranoia` | integer | `1` | 1 (fewest false positives) to 4 (most rules). |
| `anomalyThreshold` | integer | `5` | Anomaly score at which a request is blocked. A single critical rule scores 5. |
| `excludeRules` | list of integer | – |  |
| `excludeArgs` | list of string | – | Argument / header / cookie names never inspected, e.g. `password`, `html_body`. |
| `exclusions` | list of [WafExclusion](#wafexclusion) | – | Targeted false-positive fixes: some rules ignore one argument/header/cookie, optionally only under some paths — the rest of the request and every other rule stay protected. |
| `inspectBody` | bool | `true` |  |
| `maxInspectBytes` | integer | `65536` |  |
| `oversizeAction` | `pass` \| `reject` | `pass` | A request body larger than `maxInspectBytes`: `pass` (the rest goes through uninspected) or `reject` (413, in block mode), like ModSecurity's SecRequestBodyLimitAction. Rejecting stops payloads hidden behind padding, but also uploads larger than the limit. |
| `inspectResponse` | bool | `false` | CRS only: also inspect responses (data leakage, error disclosure, web shells — CRS 95x). Buffers up to `maxInspectBytes` of text-like responses before forwarding them. |
| `skipExtensions` | list of string | – | Static files not inspected, e.g. `[css, js, png, woff2]`: only GET/HEAD requests whose path ends in one of these extensions and that carry no query string and no body (attacks travel in those). Empty (the default) inspects everything. |
| `rulesets` | list of string | – | Rule packs added to the attack rules: built-in ones (`wordpress`, `php`, `sensitive-files`, `bad-bots`, `header-hardening`, ...) or groups from `waf.ruleGroups`. |
| `disabledCategories` | list of string | – | Attack categories switched off for this route, in either engine: `sqli`, `xss`, `lfi`, `rfi`, `rce`, `php`, `java`, `nodejs`, `ssrf`, `ssti`, `xxe`, `prototype-pollution`, `session`, `scanner`, `protocol`, `leakage`. |
| `rules` | list of [AccessRule](#accessrule) | – | Condition rules for this route, evaluated before the rule packs and the attack rules. |

## WafExclusion

At `global.defaultPolicy.waf.exclusions[]`.

| Field | Type | Default | Description |
|---|---|---|---|
| `rules` | list of integer | – | Rule ids; empty means every rule. |
| `target` | string | – | `args:<name>` (query, form and JSON fields, e.g. `args:user.bio`), `header:<name>`, `cookie:<name>`, `body` or `path`. Unset: the rules are skipped entirely. |
| `paths` | list of string | – | Path prefixes it applies to; empty means every path. |
| `methods` | list of string | – | Methods it applies to; empty means every method. |

## AccessRule

At `global.defaultPolicy.waf.rules[]`. A condition rule: every condition in `when` must hold (and none of `unless`).

| Field | Type | Default | Description |
|---|---|---|---|
| `id` | integer | `0` | Shown in WAF events and usable in `excludeRules`; must not collide with another rule. |
| `message` | string | – |  |
| `action` | `block` \| `allow` \| `count` \| `captcha` \| `score` | `block` | `block`: Refuse the request (in `block` mode; `detect` mode records it only); `allow`: Let the request through without any further WAF inspection; `count`: Record the match and carry on (`log` is accepted too); `captcha`: Browsers must pass a challenge first (`global.challenge`); a solved challenge counts, and clients that cannot run one are refused; `score`: Add `score` to the request's anomaly score. |
| `status` | integer | – | Response status for `block` (default 403). |
| `score` | integer | – | Anomaly score for `score` (default 5). |
| `when` | [RuleConditions](#ruleconditions) | – |  |
| `unless` | [RuleConditions](#ruleconditions) | – |  |

## RuleConditions

At `global.defaultPolicy.waf.rules[].when`.

| Field | Type | Default | Description |
|---|---|---|---|
| `methods` | list of string | – | Any of these methods. |
| `path` | [StringMatch](#stringmatch) | – | The decoded, normalised request path. |
| `queryString` | [StringMatch](#stringmatch) | – | The raw query string. |
| `headers` | map of [StringMatch](#stringmatch) | – | Request header (lower-case name) → condition. All must hold. |
| `args` | map of [StringMatch](#stringmatch) | – | Query argument → condition. All must hold. |
| `cookies` | map of [StringMatch](#stringmatch) | – | Cookie → condition. All must hold. |
| `ips` | list of CIDR | – | Client address in any of these ranges. |
| `ipLists` | list of string | – | Client address in any of these named lists. |
| `countries` | list of string | – | Client country (ISO 3166 alpha-2) is one of these. |

## StringMatch

At `global.defaultPolicy.waf.rules[].when.path`. One way to compare a value; set exactly one of the matchers. Comparisons ignore case unless `caseSensitive` is set. A missing header/argument/cookie only matches `present: false`.

| Field | Type | Default | Description |
|---|---|---|---|
| `exact` | string | – |  |
| `prefix` | string | – |  |
| `suffix` | string | – |  |
| `contains` | string | – |  |
| `regex` | string | – |  |
| `oneOf` | list of string | – | Any of these values exactly. |
| `present` | bool | – | Whether the header/argument/cookie is there at all. |
| `longerThan` | integer | – | Longer than this many bytes. |
| `not` | bool | – | Invert the result. |
| `caseSensitive` | bool | – |  |

## BasicAuth

At `global.defaultPolicy.basicAuth`.

| Field | Type | Default | Description |
|---|---|---|---|
| `realm` | string | – |  |
| `users` | list of string | – | htpasswd lines, bcrypt only (`htpasswd -B` or `lygard hash-password`). |

## JwtPolicy

At `global.defaultPolicy.jwt`.

| Field | Type | Default | Description |
|---|---|---|---|
| `issuer` | string | – |  |
| `audiences` | list of string | – |  |
| `jwksUrl` | string | – |  |
| `keys` | list of [JwtKey](#jwtkey) | – | Static keys: `{alg: RS256, pem: "..."}` or `{alg: HS256, secret: "..."}`. |
| `fromCookie` | string | – | Read the token from this cookie when there is no Authorization header. |
| `claimsToHeaders` | map of string | – | claim → request header forwarded to the backend. |
| `stripToken` | bool | `false` | Remove the Authorization header before proxying. |
| `requiredClaims` | map of list of string | – | Claims a valid token must also carry, else 403: claim (dotted path, e.g. `realm_access.roles`) → accepted values. A claim matches when it equals one value, or for arrays and space-separated strings (`scope`) contains one. All listed claims must match. |

## JwtKey

At `global.defaultPolicy.jwt.keys[]`.

| Field | Type | Default | Description |
|---|---|---|---|
| `kid` | string | – |  |
| `alg` | string | – |  |
| `pem` | string | – |  |
| `secret` | string | – |  |

## Cors

At `global.defaultPolicy.cors`.

| Field | Type | Default | Description |
|---|---|---|---|
| `allowOrigins` | list of string | – |  |
| `allowMethods` | list of string | `["GET", "POST", "PUT", "PATCH", "DELETE", "OP…` |  |
| `allowHeaders` | list of string | `["authorization", "content-type"]` |  |
| `exposeHeaders` | list of string | – |  |
| `allowCredentials` | bool | `false` |  |
| `maxAgeSeconds` | integer | `600` |  |

## SecurityHeaders

At `global.defaultPolicy.securityHeaders`.

| Field | Type | Default | Description |
|---|---|---|---|
| `hstsMaxAgeSeconds` | integer | `31536000` | Only sent over HTTPS: TLS here, or a trusted proxy's X-Forwarded-Proto. |
| `hstsIncludeSubdomains` | bool | `false` |  |
| `frameOptions` | string | `SAMEORIGIN` |  |
| `contentTypeNosniff` | bool | `true` |  |
| `referrerPolicy` | string | `strict-origin-when-cross-origin` |  |
| `contentSecurityPolicy` | string | – |  |
| `permissionsPolicy` | string | – |  |
| `crossOriginOpenerPolicy` | string | – | Cross-Origin-Opener-Policy, e.g. `same-origin`. |
| `crossOriginResourcePolicy` | string | – | Cross-Origin-Resource-Policy, e.g. `same-origin`. |
| `crossOriginEmbedderPolicy` | string | – | Cross-Origin-Embedder-Policy, e.g. `require-corp`. |
| `removeHeaders` | list of string | – | Response headers removed before they reach the client, e.g. `x-powered-by`, `x-aspnet-version` (they tell an attacker which software and version to target). |

## ExtAuth

At `global.defaultPolicy.extAuth`.

| Field | Type | Default | Description |
|---|---|---|---|
| `url` | string | – | http(s) URL of the auth endpoint, e.g. `http://oauth2-proxy.auth.svc/oauth2/auth`. |
| `method` | string | `GET` | `GET` (default) or `original` (the client's own method). |
| `timeout` | duration | `2s` |  |
| `forwardHeaders` | list of string | `["authorization", "cookie"]` | Client request headers sent to the auth service. |
| `responseHeaders` | list of string | – | Headers from the auth service's 2xx answer added to the proxied request. |
| `signinUrl` | string | – | On 401, redirect the browser here with `rd=<original URL>` (oauth2-proxy's /oauth2/start). |
| `failOpen` | bool | `false` | Let requests through when the auth service is unreachable (default: refuse with 503). |

## Acme

At `global.acme`.

| Field | Type | Default | Description |
|---|---|---|---|
| `email` | string | – | Contact address for the CA (expiry notices, account recovery). |
| `hosts` | list of string | – | One certificate per host. Wildcards need DNS-01 and are not supported here. |
| `directory` | string | `https://acme-v02.api.letsencrypt.org/directory` | ACME directory URL; Let's Encrypt staging is `https://acme-staging-v02.api.letsencrypt.org/directory`. |
| `stateDir` | string | `/var/lib/lygard/acme` | Account key, certificates and private keys are kept here (mode 0700). |

## Listener

At `listeners[]`.

| Field | Type | Default | Description |
|---|---|---|---|
| `name` | string | – |  |
| `address` | string | – |  |
| `protocol` | `http` \| `https` \| `tcp` \| `udp` | `http` | `http`; `https`; `tcp`: Layer 4: connections are relayed to an `l4Routes` backend as-is; `udp`: Layer 4: datagrams are relayed per client session. |
| `proxyProtocol` | bool | `false` | Require a PROXY protocol (v1 or v2) header on every connection. Strict on purpose: accepting headerless connections would let anyone who can reach the port forge a client address. |
| `proxyProtocolOptionalFrom` | list of CIDR | – | Sources allowed to connect *without* a PROXY header (the header is still honoured when present). For in-cluster clients that reach the load balancer address and are short- circuited to the pods by kube-proxy. Keep it to private ranges: a public client can never originate a TCP connection from one, so this cannot be used to spoof addresses. |
| `tls` | [ListenerTls](#listenertls) | `{"clientCaPem": null, "clientCertMode": "", "…` |  |
| `http3` | bool | `false` | HTTPS only: also serve HTTP/3 over QUIC on the same address and port (UDP), with the same certificates and routes, and advertise it with `Alt-Svc` on HTTP/1.1 and HTTP/2 responses. PROXY protocol does not apply: the client address is the UDP source. |
| `http3AdvertisePort` | integer | – | Port advertised in `Alt-Svc` when clients reach the listener on a different one (a load balancer mapping 443 to a node port). Default: the listener's own port. |

## ListenerTls

At `listeners[].tls`.

| Field | Type | Default | Description |
|---|---|---|---|
| `minVersion` | string | `1.2` | "1.2" or "1.3". |
| `http2` | bool | `true` |  |
| `defaultCertificate` | string | – | Served when no certificate matches the SNI. Unset → a self-signed placeholder is generated. |
| `passthroughOnly` | bool | `false` | TLS passthrough only: connections whose SNI matches no passthrough route are closed instead of being terminated here (Gateway API TLS listeners in Passthrough mode). |
| `clientCaPem` | string | – | Verify client certificates against these CAs (PEM bundle). |
| `clientCertMode` | string | – | `require` (default when a CA is set): no valid client certificate, no connection. `insecure-fallback`: certificates are requested but missing or invalid ones are let through. |
| `listenerHostnames` | list of string | – | Hostnames of the logical listeners (Gateway listeners) sharing this one, `*` for one without a hostname. A request whose Host selects a different one than its SNI gets 421. |

## Route

At `routes[]`.

| Field | Type | Default | Description |
|---|---|---|---|
| `name` | string | – |  |
| `listeners` | list of string | – | Listener names this route is served on; empty means all. |
| `hosts` | list of string | – | Exact (`api.example.com`) or wildcard (`*.example.com`); empty matches any host. |
| `excludeHosts` | list of string | – | Hosts this route must not serve even though `hosts` matches them — they belong to a more specific listener on the same port (Gateway API listener isolation). |
| `matches` | list of [RouteMatch](#routematch) | – | Any one match selects the route; empty is equivalent to `path: {type: prefix, value: /}`. |
| `backends` | list of [WeightedBackend](#weightedbackend) | – |  |
| `filters` | [Filters](#filters) | `{"mirrors": [], "requestHeaders": {"add": {},…` |  |
| `redirect` | [Redirect](#redirect) | – |  |
| `directResponse` | [DirectResponse](#directresponse) | – |  |
| `policy` | [Policy](#policy) | – |  |
| `source` | string | – | Where this route came from (e.g. `Ingress default/web`), shown in explain and the dashboard. |

## RouteMatch

At `routes[].matches[]`.

| Field | Type | Default | Description |
|---|---|---|---|
| `path` | [PathMatch](#pathmatch) | – |  |
| `method` | string | – |  |
| `headers` | list of [ValueMatch](#valuematch) | – |  |
| `query` | list of [ValueMatch](#valuematch) | – |  |

## PathMatch

At `routes[].matches[].path`.

| Field | Type | Default | Description |
|---|---|---|---|
| `type` | `prefix` \| `exact` \| `regex` \| `present` | `prefix` | `prefix`; `exact`; `regex`; `present`: Header/query only: the name exists, any value. |
| `value` | string | – |  |

## ValueMatch

At `routes[].matches[].headers[]`.

| Field | Type | Default | Description |
|---|---|---|---|
| `name` | string | – |  |
| `type` | `prefix` \| `exact` \| `regex` \| `present` | `prefix` | `prefix`; `exact`; `regex`; `present`: Header/query only: the name exists, any value. |
| `value` | string | – |  |

## WeightedBackend

At `routes[].backends[]`.

| Field | Type | Default | Description |
|---|---|---|---|
| `name` | string | – |  |
| `weight` | integer | `1` |  |
| `requestHeaders` | [HeaderMods](#headermods) | – | Request header changes applied only when this backend is chosen (Gateway API backendRefs[].filters). |

## HeaderMods

At `routes[].backends[].requestHeaders`.

| Field | Type | Default | Description |
|---|---|---|---|
| `set` | map of string | – |  |
| `add` | map of string | – |  |
| `remove` | list of string | – |  |

## Filters

At `routes[].filters`.

| Field | Type | Default | Description |
|---|---|---|---|
| `requestHeaders` | [HeaderMods](#headermods) | `{"add": {}, "remove": [], "set": {}}` |  |
| `responseHeaders` | [HeaderMods](#headermods) | `{"add": {}, "remove": [], "set": {}}` |  |
| `urlRewrite` | [UrlRewrite](#urlrewrite) | – |  |
| `mirrors` | list of [Mirror](#mirror) | – | Copies of each request sent to shadow backends; their responses are discarded. |

## UrlRewrite

At `routes[].filters.urlRewrite`.

| Field | Type | Default | Description |
|---|---|---|---|
| `hostname` | string | – |  |
| `replacePrefix` | string | – | Replace the matched path prefix with this value. |
| `replaceFull` | string | – | Replace the whole path. |
| `regex` | [RegexRewrite](#regexrewrite) | – | Regex substitution on the path, e.g. `{pattern: "^/api/(.*)", replacement: "/$1"}`. |

## RegexRewrite

At `routes[].filters.urlRewrite.regex`.

| Field | Type | Default | Description |
|---|---|---|---|
| `pattern` | string | – |  |
| `replacement` | string | – |  |

## Mirror

At `routes[].filters.mirrors[]`.

| Field | Type | Default | Description |
|---|---|---|---|
| `backend` | string | – |  |
| `percent` | number | `100.0` | Share of requests mirrored, 0-100. |

## Redirect

At `routes[].redirect`.

| Field | Type | Default | Description |
|---|---|---|---|
| `scheme` | string | – |  |
| `hostname` | string | – |  |
| `port` | integer | – |  |
| `replaceFullPath` | string | – |  |
| `replacePrefix` | string | – |  |
| `status` | integer | `302` |  |

## DirectResponse

At `routes[].directResponse`.

| Field | Type | Default | Description |
|---|---|---|---|
| `status` | integer | `200` |  |
| `body` | string | – |  |
| `contentType` | string | `text/plain; charset=utf-8` |  |

## Backend

At `backends[]`.

| Field | Type | Default | Description |
|---|---|---|---|
| `name` | string | – |  |
| `endpoints` | list of string | – | `host:port`, IPv6 as `[::1]:8080`. |
| `protocol` | `http` \| `https` \| `h2c` \| `h2` | `http` | `http`; `https`: HTTP/1.1 over TLS. Only http/1.1 is offered in ALPN: on HTTP/2 the backend would see the endpoint address as :authority instead of the client's Host; `h2c`: HTTP/2 over cleartext (gRPC backends); `h2`: HTTP/2 over TLS (gRPC backends behind TLS). |
| `lb` | `round_robin` \| `least_request` \| `random` \| `consistent_hash` | `least_request` | `round_robin`; `least_request`: Power of two random choices on in-flight requests. The default: it adapts to slow pods; `random`; `consistent_hash`: Rendezvous hashing on `hashOn`; stable when endpoints come and go. |
| `hashOn` | string | – | For consistent_hash: `ip`, `header:<name>` or `cookie:<name>`. |
| `connectTimeout` | duration | `5s` |  |
| `maxRequests` | integer | `10000` | Circuit breaker: in-flight requests above this get 503 instead of queueing. |
| `healthCheck` | [HealthCheck](#healthcheck) | – |  |
| `outlierDetection` | [OutlierDetection](#outlierdetection) | `{"consecutiveFailures": 5, "ejectionTime": "3…` |  |
| `tlsServerName` | string | – | TLS to the backend: SNI override and verification switch. |
| `tlsInsecureSkipVerify` | bool | `false` |  |
| `tlsCaPem` | string | – | Trust only this CA bundle (PEM) for the backend's certificate instead of public roots (Gateway API BackendTLSPolicy). |
| `tlsSubjectAltNames` | list of string | – | Accept the backend's certificate only if it carries one of these SANs (`dns:<name>` or `uri:<uri>`); the server name is then used for SNI only (BackendTLSPolicy subjectAltNames). |
| `tlsClientCertPem` | string | – | Client certificate (PEM chain and key) presented to the backend (mTLS; Gateway API spec.tls.backend.clientCertificateRef). |
| `tlsClientKeyPem` | string | – |  |
| `sessionPersistence` | [SessionPersistence](#sessionpersistence) | – | Sticky sessions: Lygard issues a cookie naming the endpoint that served the first request and sends the client's later requests there while it is available (Gateway API sessionPersistence, ingress-nginx cookie affinity). |
| `source` | string | – |  |

## HealthCheck

At `backends[].healthCheck`.

| Field | Type | Default | Description |
|---|---|---|---|
| `path` | string | `/` |  |
| `interval` | duration | `10s` |  |
| `timeout` | duration | `2s` |  |
| `healthyThreshold` | integer | `2` |  |
| `unhealthyThreshold` | integer | `3` |  |
| `expectedStatuses` | list of string | `["200-399"]` | Inclusive ranges, e.g. `["200-399"]`. |

## OutlierDetection

At `backends[].outlierDetection`.

| Field | Type | Default | Description |
|---|---|---|---|
| `consecutiveFailures` | integer | `5` | Consecutive connect failures or 5xx before an endpoint is ejected. 0 disables. |
| `ejectionTime` | duration | `30s` |  |
| `maxEjectionPercent` | integer | `50` | Never eject more than this share of a backend, so a bad deploy cannot empty it. |

## SessionPersistence

At `backends[].sessionPersistence`.

| Field | Type | Default | Description |
|---|---|---|---|
| `cookieName` | string | `lygard-session` |  |
| `ttl` | duration | – | Cookie lifetime; unset means a browser-session cookie. |

## Certificate

At `certificates[]`.

| Field | Type | Default | Description |
|---|---|---|---|
| `name` | string | – |  |
| `certPem` | string | – |  |
| `keyPem` | string | – |  |
| `certFile` | string | – |  |
| `keyFile` | string | – |  |
| `hosts` | list of string | – | Hostnames to serve this certificate for. Empty → taken from the certificate's SANs. |

## TlsRoute

At `tlsRoutes[]`.

| Field | Type | Default | Description |
|---|---|---|---|
| `name` | string | – |  |
| `listeners` | list of string | – | HTTPS listeners this applies to; empty means all. |
| `hosts` | list of string | – | SNI names, exact or `*.example.com`. |
| `backends` | list of [WeightedBackend](#weightedbackend) | – |  |
| `source` | string | – |  |
| `terminate` | bool | `false` | Terminate TLS with the listener's certificates and relay the decrypted stream (TLSRoute Terminate mode). Otherwise the encrypted connection is passed through unchanged. |

## L4Route

At `l4Routes[]`.

| Field | Type | Default | Description |
|---|---|---|---|
| `name` | string | – |  |
| `listeners` | list of string | – | `tcp` / `udp` listeners this route serves; empty means all of them. |
| `backends` | list of [WeightedBackend](#weightedbackend) | – |  |
| `source` | string | – |  |

## WafGlobal

At `waf`.

| Field | Type | Default | Description |
|---|---|---|---|
| `customRules` | list of [CustomRule](#customrule) | – |  |
| `crsPath` | string | `/usr/share/lygard/crs` | OWASP CRS directory (with `rules/`). The container image ships CRS v4 at this path. |
| `seclang` | string | – | Extra SecLang directives loaded after CRS: exclusions, custom SecRules. |
| `ruleGroups` | map of list of [AccessRule](#accessrule) | – | Named groups of condition rules, enabled per route with `waf.rulesets`. |
| `ipLists` | map of [IpListSpec](#iplistspec) | – | Named IP lists for `ipDenyLists` and rule conditions. Presets (`spamhaus-drop`, `firehol-level1`, `tor-exits`) need no entry here unless you change them. |

## CustomRule

At `waf.customRules[]`.

| Field | Type | Default | Description |
|---|---|---|---|
| `id` | integer | `0` |  |
| `message` | string | – |  |
| `pattern` | string | – | Case-insensitive regex, applied to normalised (decoded, lower-cased) values. |
| `targets` | list of string | `["args"]` | Any of `path`, `args`, `headers`, `cookies`, `body`, `user-agent`. |
| `score` | integer | `5` |  |
| `paranoia` | integer | `1` |  |

## IpListSpec

At `waf.ipLists.<name>`. A named IP list: inline ranges, files (e.g. a mounted ConfigMap) and/or URLs, refreshed in the background. One address or range per line; `#` and `;` start comments; Spamhaus JSON lines work.

| Field | Type | Default | Description |
|---|---|---|---|
| `cidrs` | list of CIDR | – |  |
| `files` | list of string | – |  |
| `urls` | list of string | – |  |
| `refresh` | duration | `1h` | How often files and URLs are read again. |
| `excludePrivate` | bool | `true` | Drop private, loopback, link-local and shared (CGNAT) ranges, so a list of bad public addresses never blocks your own network, health checks or load balancers. |
