# Benchmarks

Numbers here are measurements, not estimates, with the exact setup next to each. Absolute
throughput depends on the machine; compare ratios, and compare runs from the same session only
(the same laptop's ceiling moved by 20% between a morning and an afternoon).

## Data plane cost per feature

`bench/run.py`: one Lygard process proxies to a second one answering directly; `oha` drives 200
connections for 10 s, three rounds, medians. Apple M-series laptop, release build (mimalloc),
2026-11-02 (`bench/results-laptop-2026-11-02.json`).

| Configuration | req/s | CPU µs/request | p99 | RSS |
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
python3 bench/conns.py                   # CONNS=10000 MODE=proxy|server
tests/bench-k8s/route-scale.sh v1.37.0   # needs kind, oha and the :matrix images
```

Pull requests that touch the data plane run an interleaved A/B of the PR against its base branch
on the same runner (`.github/workflows/perf.yml`, `bench/gate.py`) and fail on a CPU-per-request
or p99 regression above 10%.
