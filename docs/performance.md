# Performance tuning

What each feature and setting costs, measured, and the knobs that trade protection or detail
for CPU and memory. Numbers are CPU microseconds per request and resident memory of one data
plane process, from `bench/run.py` (three rounds, medians, Apple M-series laptop, 200
connections; `bench/results-laptop-2026-11-02.json`). Compare the ratios: absolute values move
with the machine. [benchmarks.md](benchmarks.md) has the method.

The lightest path (no access log, no OpenTelemetry, no WAF) costs **55 µs** per request.

## OWASP CRS

The Core Rule Set evaluates hundreds of rules per request, so it is the most expensive feature by
far. It is opt-in per route (`policy.waf.engine: crs` or `lygard.io/waf-engine: crs`).

| Setting | CPU µs/request | |
|---|---|---|
| CRS, paranoia 1, GET | 190 | the baseline |
| detect mode instead of block | 191 | same work, no blocking |
| paranoia 2 | 247 | +30%: more rules |
| POST with a 1 KB JSON body | 441 | every argument is inspected |
| ... with `maxInspectBytes: 256` | 197 | only the first 256 bytes of the body |
| ... with `inspectBody: false` | 199 | headers, path and query only |
| static file with `skipExtensions: [css]` | 55 | not inspected at all |
| native engine (`engine: native`) | 55 | Lygard's own rules: injection, traversal, scanners |

How to keep it affordable:

- **Static assets**: `skipExtensions: [css, js, png, jpg, svg, woff2, ico]` (annotation
  `lygard.io/waf-skip-extensions`). Only GET/HEAD requests without a query string and without a
  body are skipped, so a payload in `?id=` on a `.js` URL is still inspected.
- **Large bodies**: `maxInspectBytes` (default 64 KiB) bounds the work per request; the rest of
  the body streams through uninspected. Uploads and APIs carrying files: lower it, or turn
  `inspectBody` off for those routes and keep it on for forms and JSON APIs.
  CRS cost grows linearly with the inspected body: in the engine alone (one Apple M4 Pro core,
  `crs_throughput` benchmark, benign prose) a 1 KB JSON body costs 0.34 ms, 16 KB 2.2 ms and a
  64 KB form 8.7 ms. About two thirds of that is the rules' operators each scanning the value,
  the rest transformations (URL and entity decoding per rule chain). A shared literal prefilter
  across rules was measured and not adopted: only about 40% of the CRS regexes have a literal
  set that every match starts with, the regex engine already prefilters those internally, and
  they account for 14% of the time, less than the extra scan costs.
- **Paranoia**: stay at 1 unless a route needs 2+; each level adds rules and false positives.
- **Exclusions** (`lygard.io/waf-exclusions`) remove rules from specific arguments, which also
  saves their evaluation.
- **Where it matters**: CRS on the routes that take user input (login, search, forms, APIs), the
  native engine elsewhere.

Matched requests are recorded for `/api/waf/tuning` by a background thread; a flood of attacks
does not slow clean requests through shared locks. When the recorder falls behind, events are
dropped and counted in `lygard_waf_events_dropped_total` (blocking is unaffected).

## OpenTelemetry

Traces and logs are encoded off the request path and sent in batches.

| Setting | CPU µs/request | Memory (MB) |
|---|---|---|
| traces at 5% sampling (default) | 53 | 39 |
| traces at 100% + access logs as OTLP logs | 60 | 72 |
| ... `protocol: json` | 69 | 113 |
| ... `compression: gzip` | 60 | 80 |
| ... `logSampling: errors` | 59 | 49 |

- `protocol: protobuf` (default) is the cheapest encoding; `json` is there for collectors that
  require it.
- `compression: gzip` costs little CPU and cuts bandwidth several-fold: use it when the collector
  is across a WAN or a metered link.
- `logSampling: sampled` sends logs only for traced requests (logs and traces match one to one);
  `errors` sends only status 5xx or requests with a response flag. Both cut collector load too.
- `batchSize` (512) and `exportInterval` (2s) set how records are grouped; larger batches are
  cheaper for the collector, longer intervals delay data. `queueSize` (8192 per signal) is how
  much is held while the collector is slow; beyond it records are dropped and counted
  (`lygard_trace_spans_dropped_total`, `lygard_access_log_dropped_total`).
- `sampleRatio` sets head sampling for requests that arrive without a sampled `traceparent`.

## Access log

The stdout JSON access log costs about 2 µs per request (written by its own thread in batches).
`accessLog: false` globally or `lygard.io/access-log: "false"` per route turns it off.

## Memory per connection

An idle HTTP/1.1 keep-alive connection costs **10.6 KB** proxied (8.2 KB when Lygard answers
itself), measured on Linux with 10,000 idle connections (`bench/conns.py`): idle connections
hold no I/O buffers (they are taken when a request arrives and given back when it is done). A
data plane with a 1 GiB memory limit holds about 90,000 idle connections. Open connections are
capped by `maxConnections` (default 50,000 per pod); idle HTTP/1.1 connections are closed after
`idleTimeout` (120 s) without a request (before 0.13.1, after `headerReadTimeout`, 15 s).

Busy connections cost more than idle ones: each request in flight holds its read buffer and an
upstream connection. On a Linode NodeBalancer at 10,000 connections sending ~1,000 requests per
second (two pods, CPU-bound node), each pod held about 4,800 connections at 250-320 MB, roughly
30 KB per connection. Size the memory limit for the busy case.

For the most connections per pod: `presets/high-scale.yaml`. For the smallest footprint:
`presets/light.yaml`.

## Other features

| Feature | CPU µs/request |
|---|---|
| API discovery (with response PII scan) | 57 |
| anomaly detection | 53 |
| proxy-wasm plugin, headers only | 99 |
| native WAF (block) | 55 |

proxy-wasm plugins pay for crossing into the WebAssembly sandbox on every callback; a plugin that
also reads bodies (`requestBody`/`responseBody`) pays for buffering and copying them.

## Reproduce

```sh
cargo build --release
python3 bench/run.py                                  # all scenarios; ROUNDS=3 DUR=10s
ONLY="CRS|OTel" ROUNDS=3 python3 bench/run.py         # a subset
CONNS=10000 MODE=proxy python3 bench/conns.py         # memory per connection
```
