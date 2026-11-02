# Benchmarks

Numbers here are measurements, not estimates, with the exact setup next to each. Absolute
throughput depends on the machine; compare ratios, and compare runs from the same session only
(the same laptop's ceiling moved by 20% between a morning and an afternoon).

## Data plane cost per feature

`bench/run.py`: one Lygard process proxies to a second one answering directly; `oha` drives 200
connections for 10 s, three rounds, medians. Apple M-series laptop, release build (mimalloc),
2026-11-02 (`bench/results-laptop-2026-11-02.json`).

This is a **throughput** test: every connection sends its next request the moment the previous
one returns, so the load generator, Lygard and the backend run the laptop's CPUs at 100%. Latency
under saturation is queueing, about connections ÷ req/s (200 ÷ 54,500 = 3.7 ms), which is why even
the backend alone shows 2 ms. Read the p99 column as "how the queue grows", not as the latency
Lygard adds; that is in [latency at a fixed rate](#latency-at-a-fixed-rate) below. The cost per
request is the CPU column.

| Configuration | req/s | CPU µs/request | p99 at saturation | RSS |
|---|---|---|---|---|
| Backend answering directly (ceiling) | 108,300 | – | 2.1 ms | – |
| Lygard, lightest (no logs, no OTel, no WAF) | 54,500 | 55 | 4.5 ms | 37 MB |
| + access log (JSON) | 53,500 | 57 | 5.2 ms | 34 MB |
| + API discovery (with response PII scan) | 54,700 | 57 | 4.6 ms | 32 MB |
| + anomaly detection | 54,900 | 53 | 4.7 ms | 36 MB |
| + OpenTelemetry traces, 5% sampling | 55,500 | 53 | 4.4 ms | 39 MB |
| + OpenTelemetry, 100% sampling + logs | 53,900 | 60 | 4.9 ms | 72 MB |
| + native WAF (block) | 53,500 | 55 | 6.2 ms | 36 MB |
| + OWASP CRS paranoia 1 (block) | 37,200 | 190 | 11.9 ms | 55 MB |
| + OWASP CRS, POST 1 KB JSON | 18,100 | 441 | 24.2 ms | 67 MB |
| + proxy-wasm plugin (headers only) | 48,500 | 99 | 7.8 ms | 39 MB |

The default profile (5% trace sampling, native WAF) costs nothing measurable over the lightest
path. Full CRS costs about 3.5x CPU per request and stays opt-in per route; what each CRS and
OpenTelemetry setting costs is in [performance.md](performance.md).

## Latency at a fixed rate

The same setup at a steady 5,000 requests per second over 50 connections, well below saturation
(`RATE=5000 CONC=50 python3 bench/run.py`; `oha -q 5000 --latency-correction`), three rounds of
10 s, medians (`bench/results-laptop-rate5000-2026-11-02.json`). "Added" is the median minus the
backend's own.

| Configuration | p50 | Added at p50 |
|---|---|---|
| Backend answering directly | 0.19 ms | – |
| Lygard, lightest | 0.36 ms | 0.17 ms |
| + access log (JSON) | 0.45 ms | 0.26 ms |
| + API discovery | 0.47 ms | 0.28 ms |
| + anomaly detection | 0.46 ms | 0.27 ms |
| + OpenTelemetry traces, 5% sampling | 0.48 ms | 0.29 ms |
| + OpenTelemetry, 100% sampling + logs | 0.45 ms | 0.26 ms |
| + native WAF (block) | 0.43 ms | 0.24 ms |
| + OWASP CRS paranoia 1 (block) | 0.48 ms | 0.29 ms |
| + OWASP CRS, POST 1 KB JSON | 0.69 ms | 0.50 ms |
| + proxy-wasm plugin (headers only) | 0.44 ms | 0.25 ms |
| everything on (logs, discovery, anomaly, OTel 5%, CRS) | 0.47 ms | 0.28 ms |

Lygard adds about 0.2-0.3 ms at the median, 0.5 ms with full CRS inspecting a request body.

Tail latency is not reported from this laptop: p99 for the same configuration ranged from 0.7 ms
to 120 ms between rounds (the backend alone: 0.5-2.9 ms), because the OS schedules the load
generator, both proxies and everything else on the same cores. Tails need dedicated Linux
machines with the load generator on its own host.

## Memory

`bench/conns.py`: RSS growth per idle connection, 10,000 connections, three rounds, Linux
(`rust:1.99` container); the macOS process memory figures are noisy (memory compression).

| | Per connection |
|---|---|
| Idle TCP connection | ~6 KB |
| Idle HTTP/1.1 keep-alive, Lygard answering directly | ~8 KB (32 KB in 0.12) |
| Idle HTTP/1.1 keep-alive, proxied (incl. pooled upstream share) | ~11 KB (35 KB in 0.12) |
| Idle process | ~48 MB |

Idle HTTP/1 connections hold no I/O buffers: hyper is carried with a small patch
(`third_party/hyper`, see `LYGARD-PATCH.md`) that takes the read and header-write buffers when a
request arrives and returns them when the connection goes quiet.

## Route scale in Kubernetes

`tests/bench-k8s/route-scale.sh`: kind on Kubernetes 1.37 (one control plane, two workers), two
data plane and two controller replicas, 50 connections for 20 s through the newest route.
Throughput is capped by Docker Desktop's port forwarding, not by Lygard.

| HTTPRoutes | Newest route serving after | p50 | p99 | Data plane memory (2 pods) | Controller memory (2 pods) |
|---|---|---|---|---|---|
| 10 | 0.5 s | 3.4 ms | 16.5 ms | 75 MiB | 21 MiB |
| 100 | 1.1 s | 2.9 ms | 16.2 ms | 79 MiB | 26 MiB |
| 500 | 3.7 s | 2.9 ms | 16.8 ms | 84 MiB | 35 MiB |
| 1,000 | 4.8 s | 2.9 ms | 16.3 ms | 95 MiB | 52 MiB |
| 2,000 | 9.0 s | 2.9 ms | 16.8 ms | 107 MiB | 75 MiB |

Latency stays flat with route count; propagation time includes `kubectl apply` of the routes.

## Reproduce

```sh
cargo build --release
python3 bench/run.py                     # ROUNDS=3 DUR=10s ONLY="lightest|CRS"
RATE=5000 CONC=50 python3 bench/run.py   # latency at a fixed rate instead of saturation
python3 bench/conns.py                   # CONNS=10000 MODE=proxy|server
tests/bench-k8s/route-scale.sh v1.37.0   # needs kind, oha and the :matrix images
```

Pull requests that touch the data plane run an interleaved A/B of the PR against its base branch
on the same runner (`.github/workflows/perf.yml`, `bench/gate.py`) and fail on a CPU-per-request
or p99 regression above 10%.
