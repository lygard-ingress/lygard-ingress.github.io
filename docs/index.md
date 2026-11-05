# Lygard

!!! note "Private preview"
    Lygard is in private preview: the source repository, container images and Helm chart are not
    public yet. The install commands below show how it will work once they are.

**A lightweight, secure ingress gateway and Kubernetes ingress controller — with the WAF, auth,
rate limiting and observability other gateways charge for, built in and free.**

Lygard has two parts, split the way Linkerd and Istio ambient are:

- **`lygard` — the data plane (Rust).** Terminates TLS, routes, load-balances, enforces WAF / auth /
  rate limits, and emits metrics, logs and traces. Runs standalone from one YAML file (Linux,
  macOS, Windows, VMs, Docker) or in Kubernetes, where it holds **no Kubernetes credentials at all**.
- **`lygard-controller` — the control plane (Go, client-go + official Gateway API types).** Watches
  `Ingress` (including ingress-nginx annotations), the **Gateway API** (`Gateway`, `HTTPRoute`,
  `GRPCRoute`, `TLSRoute`, `TCPRoute`, `UDPRoute`, `BackendTLSPolicy`, `ReferenceGrant`) and
  `LygardPolicy`, translates them into the same configuration the YAML file uses, streams it to
  the proxies (HTTPS long-poll, CA-pinned, token-authenticated), and writes status.

If the controller is down, proxies keep serving their last good configuration.

```
 Kubernetes API ──watch──▶ lygard-controller (Go) ──config over HTTPS──▶ lygard (Rust) ×N ◀── traffic
                                  │ status                                   │ peer rate-limit sync
                                  ▼                                          └──────────────────────
```

## Why

| | nginx OSS | Envoy | Traefik OSS | **Lygard** |
|---|---|---|---|---|
| Active health checks | paid (Plus) | ✅ | ✅ | ✅ |
| JWT / OIDC-token auth | paid (Plus) | filter config | paid | ✅ built in |
| WAF | ModSecurity module | WASM add-on | plugin | ✅ native, per-route tuning |
| Live dashboard | paid (Plus) | admin API | basic | ✅ built in |
| Explain "which route/policy/WAF verdict does this URL get" | ❌ | ❌ | ❌ | ✅ `lygard explain` |
| External dependencies | none | none | none | none |

## What you get

| | |
|---|---|
| **Routing** | Ingress, the Gateway API (HTTP, gRPC, TLS, TCP, UDP routes) and ingress-nginx annotations; HTTP/1.1, HTTP/2, HTTP/3; load balancing, retries, health checks, outlier detection, progressive rollouts with automatic rollback |
| **Security** | A native WAF and the OWASP Core Rule Set; rule packs (WordPress, PHP, sensitive files, bad bots, ...); custom rules that allow, block, count or challenge; IP reputation lists; rate limits; JWT, basic and forward auth; security headers; automatic certificates |
| **Observability** | One ID per request, equal to its trace ID, on error pages, logs and traces; a request inspector that explains any request; detailed metrics for every WAF rule and action; OpenTelemetry traces, metrics and logs; a built-in dashboard |
| **Easy to configure** | Annotations or a typed `LygardPolicy`; the dashboard generates Ingresses, HTTPRoutes, policies, Helm values and Secrets with every field explained, plus suggestions from your traffic. You apply them; the dashboard never changes your cluster |
| **Light** | 40–60 MB of memory per proxy and about 0.2 ms added at the median ([benchmarks](benchmarks.md)); no database, no Redis, no external services. The proxies hold no Kubernetes credentials |

The full list is in [features](features.md).

## Quick start

```sh
helm install lygard oci://ghcr.io/lygard-ingress/charts/lygard -n lygard --create-namespace \
  -f https://raw.githubusercontent.com/lygard-ingress/lygard/main/deploy/helm/lygard/presets/linode.yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  annotations:
    lygard.io/waf: detect
    lygard.io/rate-limit-rps: "50"
spec:
  ingressClassName: lygard
  rules:
    - host: app.example.com
      http:
        paths:
          - {path: /, pathType: Prefix, backend: {service: {name: web, port: {number: 80}}}}
```

[Getting started](getting-started.md) walks through installing on any cloud, opening the
dashboard, following a request by its ID, and running without Kubernetes from one YAML file.

## Documentation

| | |
|---|---|
| **Start** | [Getting started](getting-started.md) · [Features](features.md) · [Cloud providers](cloud-providers.md) · [Migrating from ingress-nginx](migrating-from-ingress-nginx.md) |
| **Configure** | [Annotations](annotations.md) · [LygardPolicy](policy.md) · [Configuration reference](configuration.md) · [Certificates](certificates.md) · [HTTP/3](http3.md) |
| **Secure** | [WAF rules](waf-rules.md) · [Authentication](authentication.md) · [Hardening](hardening.md) |
| **Observe** | [Observability](observability.md) · [Dashboard](dashboard.md) |
| **Operate** | [Upgrading](upgrading.md) · [Performance tuning](performance.md) · [Benchmarks](benchmarks.md) · [Conformance](conformance.md) |
| **Project** | [Roadmap](roadmap.md) · [Changelog](changelog.md) |

## Build

```sh
cargo test && cargo build --release                       # data plane
(cd controller && go test ./... && go build .)            # controller
docker buildx build --platform linux/amd64,linux/arm64 -t lygard:dev .
docker buildx build --platform linux/amd64,linux/arm64 -t lygard-controller:dev controller
```

## Contributing

Issues and pull requests are welcome — see CONTRIBUTING.md, the
Code of Conduct and GOVERNANCE.md. Report security issues
privately as described in SECURITY.md.

## License

Apache-2.0 (LICENSE, NOTICE). Lygard has no paid edition: every feature is
open source.
