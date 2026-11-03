# Benchmarks

Numbers here are measurements, not estimates, with the exact setup next to each. Absolute
throughput depends on the machine; compare ratios, and compare runs from the same session only.

The tables below come from a dedicated cloud VM: a Linode Dedicated 8 (8 dedicated vCPUs, AMD
EPYC 7713 at 2.0 GHz, 16 GB, Ubuntu 24.04), with Lygard 0.13.1 plus the CRS engine speedup
(main at f26649a), built there with Rust 1.99 and run with `oha` 1.16.0, 2026-11-02. Rounds of the
same configuration agree within a few percent.

## Data plane cost per feature

`bench/run.py`: one Lygard process proxies to a second one answering directly; `oha` drives 200
connections for 10 s, three rounds, medians (`bench/results-linode-dedicated8-2026-11-02.json`).
Load generator, Lygard and backend share the VM's 8 cores.

This is a **throughput** test: every connection sends its next request the moment the previous
one returns, so the VM's CPUs run at 100%. Latency under saturation is queueing, about
connections ÷ req/s (200 ÷ 49,900 = 4 ms), which is why even the backend alone shows 4 ms at
p99. Read the p99 column as "how the queue grows", not as the latency Lygard adds; that is in
[latency at a fixed rate](#latency-at-a-fixed-rate) below. The cost per request is the CPU
column.

| Configuration | req/s | CPU µs/request | p99 at saturation | RSS |
|---|---|---|---|---|
| Backend answering directly (ceiling) | 149,600 | – | 4.2 ms | – |
| Lygard, lightest (no logs, no OTel, no WAF) | 49,900 | 80 | 8.3 ms | 63 MB |
| + access log (JSON) | 49,800 | 80 | 8.4 ms | 63 MB |
| + API discovery (with response PII scan) | 49,100 | 81 | 8.4 ms | 64 MB |
| + anomaly detection | 51,400 | 78 | 8.0 ms | 62 MB |
| + OpenTelemetry traces, 5% sampling | 52,500 | 76 | 8.0 ms | 66 MB |
| + OpenTelemetry, 100% sampling + logs | 45,600 | 94 | 9.1 ms | 98 MB |
| + native WAF (block) | 48,300 | 85 | 8.6 ms | 102 MB |
| + OWASP CRS paranoia 1 (block) | 18,900 | 339 | 18.8 ms | 152 MB |
| + OWASP CRS, POST 1 KB JSON | 7,900 | 907 | 47.6 ms | 162 MB |
| + proxy-wasm plugin (headers only) | 28,600 | 157 | 12.8 ms | 82 MB |
| everything on (logs, discovery, anomaly, OTel 5%, CRS) | 17,900 | 358 | 19.8 ms | 156 MB |

The default profile (5% trace sampling, native WAF) costs nothing measurable over the lightest
path. Full CRS costs about 4x CPU per request on this server CPU (about 2.6x on an Apple M4 Pro)
and stays opt-in per route; what each CRS and OpenTelemetry setting costs is in
[performance.md](performance.md).

## Latency at a fixed rate

The same setup at a steady 5,000 requests per second over 50 connections, well below saturation
(`RATE=5000 CONC=50 python3 bench/run.py`; `oha -q 5000 --latency-correction`), three rounds of
10 s, medians (`bench/results-linode-dedicated8-rate5000-2026-11-02.json`). "Added" is the
median minus the backend's own.

| Configuration | p50 | Added at p50 | p99 |
|---|---|---|---|
| Backend answering directly | 0.17 ms | – | 0.3-0.7 ms |
| Lygard, lightest | 0.35 ms | 0.18 ms | 0.55 ms |
| + access log (JSON) | 0.35 ms | 0.19 ms | 0.63 ms |
| + API discovery | 0.36 ms | 0.19 ms | 0.63 ms |
| + anomaly detection | 0.34 ms | 0.18 ms | 0.63 ms |
| + OpenTelemetry traces, 5% sampling | 0.35 ms | 0.19 ms | 0.65 ms |
| + OpenTelemetry, 100% sampling + logs | 0.36 ms | 0.19 ms | 0.75 ms |
| + native WAF (block) | 0.35 ms | 0.18 ms | 0.62 ms |
| + OWASP CRS paranoia 1 (block) | 0.64 ms | 0.47 ms | 1.1 ms |
| + OWASP CRS, POST 1 KB JSON | 1.19 ms | 1.02 ms | 3.1 ms |
| + proxy-wasm plugin (headers only) | 0.44 ms | 0.27 ms | 0.73 ms |
| everything on (logs, discovery, anomaly, OTel 5%, CRS) | 0.65 ms | 0.48 ms | 1.1 ms |

Lygard adds about 0.2 ms at the median, and about 0.5 ms with OWASP CRS (1 ms when CRS inspects a
1 KB request body). p99 stays under 1 ms without CRS and about 1 ms with it. The p99 values vary
by up to 20% between rounds and include the load generator, which runs on the same VM; a
separate load-generator host would lower them slightly.

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
