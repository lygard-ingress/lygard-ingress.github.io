# Observability

Every request has one ID. Lygard puts it in five places:

- the `x-request-id` response header;
- the error page the visitor sees;
- the access log line;
- the OpenTelemetry trace;
- the request inspector.

If someone reports "I got a 403", ask for the ID shown on the page. One lookup then explains
what happened.

## One ID: request ID = trace ID

- **New requests.** When a request arrives without an `X-Request-Id` header, its request ID is
  its W3C trace ID: 32 hex characters. This holds whether the trace was continued from an
  incoming `traceparent` or started at Lygard.
- **Client-supplied IDs.** If the client or an upstream proxy sends a well-formed
  `X-Request-Id`, Lygard keeps that ID. It then also returns the trace ID in an `x-trace-id`
  response header, so either value finds the request.
- **Where the ID appears.** Both IDs are in the access log (`request_id`, `trace_id`,
  `span_id`). The server span carries `http.request.id` and `lygard.request_id`.
- **Downstream propagation.** Backends receive `x-request-id` and `traceparent`, so their own
  logs and spans join the same trace.

## Request inspector

**Dashboard → Request inspector**, or `GET /api/requests/<request ID or trace ID>`.

Each replica keeps its last `global.requestHistory` requests (default 4096, about 0.5 KB each).
A lookup checks this replica first, then asks the others. The answer contains:

- **summary:** the likely root cause in one sentence, for example "The backend did not answer
  in time (504) (upstream stage)".
- **timeline:** received → routing → WAF → upstream (backend, endpoint, attempts, time) →
  response.
- **why:** every response flag with its meaning and the usual fix (table below).
- **WAF matches:** each rule ID, category, message, the field it fired on and a sample of the
  value.
- **details:** client and country, authenticated user, user agent, route, backend, bytes, the
  internal error text, whether the trace was sampled, and a link to the trace when
  `global.tracing.traceLink` is set.

`GET /api/inspector?problems=1` lists recent failed or flagged requests. Requests older than
the history are still in the access log and in your tracing backend under the same ID.

```yaml
global:
  requestHistory: 4096
  tracing:
    otlpEndpoint: http://otel-collector.observability:4318
    sampleRatio: 0.05
    traceLink: "https://grafana.example.com/explore?left={\"queries\":[{\"query\":\"{traceId}\"}]}"
```

## Response flags

Flags record every decision Lygard took that changed the outcome. They appear in the access log
(`flags`), the inspector, `lygard.flags` on spans, and the WAF and request feeds.

| Flag | Stage | Meaning |
|---|---|---|
| `NR` | routing | No route matched the host and path (404) |
| `MIS` | routing | TLS SNI and Host select different certificates (421) |
| `PN` | routing | The path was normalised before routing |
| `IPD` / `IPL` / `GEO` / `JA4` | access | Refused by an IP list, an IP reputation list, a country rule or a TLS fingerprint (403) |
| `RL` | rate limit | Rate limit exceeded (429) |
| `AUTH` | authentication | Missing or rejected credentials (401/403) |
| `BODY` / `BT` | request | Body too large (413) or too slow (408) |
| `WAF` | WAF | Blocked by a rule or by the anomaly score (403) |
| `CHAL` | WAF | A challenge page was served (`captcha` rule) |
| `WAFT` | WAF | Body larger than `maxInspectBytes`; only its start was inspected |
| `WAFR` | WAF | Response blocked by response inspection |
| `UH` / `UO` | upstream | No healthy endpoint, or the concurrency limit was reached (503) |
| `UF` / `UT` / `UPE` | upstream | Connect failure (502), timeout (504), broken exchange (502) |
| `RTY` / `URX` | upstream | Retried / every retry failed |
| `PLG` / `EP` | response | A plugin or a custom error page changed the response |
| `DC` | client | The client disconnected before the response was complete |

## Error pages

Lygard's own error responses (WAF blocks, 401, 404, 429, 502, 503, 504 and so on) match what the
client asked for:

- **Browsers:** a small self-contained HTML page: status, a plain sentence, and the request ID.
- **API clients** (`Accept: application/json`):
  `{"status":403,"error":"Access denied","message":"...","requestId":"...","traceId":"..."}`.
- **Everyone else:** one line of text with the request ID.

The pages never show rule IDs, backend names or internal errors; those go to the logs and the
inspector. Responses from your backends pass through unchanged unless the route sets
`errorPages`.

```yaml
global:
  errorPages:
    format: auto            # auto | html | json | text
    title: Example Shop     # page heading prefix (default: the host name)
    pages:                  # your own page for Lygard's errors; route errorPages win
      "503": {body: "<h1>{status} {reason}</h1><p>Back soon. Reference: {requestId}</p>", contentType: text/html}
```

Placeholders in `global.errorPages.pages` and in route `errorPages` are `{status}`,
`{reason}`, `{message}`, `{requestId}` and `{traceId}`.

## Metrics

Metrics are served at `/metrics` on the admin port in Prometheus format. They can also be
pushed over OTLP (`tracing.exportMetrics`). Each label set is bounded: routes, backends, rule
IDs and fixed outcome names, never paths or client addresses.

| Area | Metrics |
|---|---|
| Traffic | `lygard_http_requests_total{listener,route,method,code,backend}`, `lygard_http_request_duration_seconds{route,code_class}`, `lygard_http_request_bytes_total`, `lygard_http_response_bytes_total`, `lygard_http_active_requests` |
| WAF verdicts | `lygard_waf_decisions_total{route,mode,decision}`, where `decision` is `clean`, `flagged`, `blocked` or `blocked_response` |
| WAF rules | `lygard_waf_rule_matches_total{route,rule,category}`: every rule that matched; `lygard_waf_inspect_duration_seconds{route}`: time spent in the WAF |
| WAF actions | `lygard_waf_rule_actions_total{route,rule,action}`, where `action` is `block`, `allow`, `count`, `score`, `captcha` or `captcha_passed` |
| Challenges | `lygard_challenges_total{result}`, where `result` is `issued`, `passed` or `failed` |
| IP lists | `lygard_ip_list_ranges{list}`, `lygard_ip_list_refresh_total{list,result}`, `lygard_ip_filter_denied_total{route}` |
| Access control | `lygard_ratelimit_decisions_total`, `lygard_auth_results_total{route,type,result}`, `lygard_ja4_denied_total` |
| Upstream | `lygard_upstream_requests_total{backend,endpoint,code_class}`, `lygard_upstream_errors_total{backend,kind}`, `lygard_upstream_retries_total`, `lygard_upstream_response_duration_seconds{backend}`, `lygard_backend_endpoints{backend,state}`, `lygard_health_checks_total`, `lygard_outlier_ejections_total`, `lygard_circuit_breaker_overflow_total` |
| TLS and connections | `lygard_tls_handshakes_total`, `lygard_tls_handshake_errors_total`, `lygard_certificate_expiry_timestamp_seconds`, `lygard_active_connections`, `lygard_connections_rejected_total` |
| Lygard itself | `lygard_config_reloads_total`, `lygard_config_version`, `lygard_controlplane_connected`, `lygard_otlp_exports_total`, `lygard_build_info` |

Useful queries:

```promql
# Blocked share per route
sum by (route) (rate(lygard_waf_decisions_total{decision="blocked"}[5m]))
  / sum by (route) (rate(lygard_http_requests_total[5m]))
# The 10 rules that block most
topk(10, sum by (rule, category) (rate(lygard_waf_rule_matches_total[1h])))
# Challenge solve rate (low = mostly bots)
rate(lygard_challenges_total{result="passed"}[1h]) / rate(lygard_challenges_total{result="issued"}[1h])
```

The chart ships a Grafana dashboard (`deploy/helm/lygard/dashboards/lygard.json`).

## Traces and logs (OpenTelemetry)

Set `global.tracing.otlpEndpoint` to export over OTLP/HTTP: protobuf by default, gzip optional.

- **Spans.** One server span per request, carrying the request ID, route, backend, flags, WAF
  score and rules, upstream time, attempts and error. With `upstreamSpans`, Lygard also emits
  one client span per backend attempt.
- **Logs.** `exportLogs` sends access log records as OTLP logs with their trace and span IDs.
  `logSampling: sampled` keeps logs and traces in step.
- **Sampling.** `sampleRatio` applies to requests without a sampled parent. Requests that are not
  sampled are still in the inspector and the access log under their ID.

What each setting costs is in [performance.md](performance.md#opentelemetry).
