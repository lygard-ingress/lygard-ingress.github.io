# Getting started

This guide takes about ten minutes. It installs Lygard, puts a site behind it with the WAF in
detect mode, and then follows one request from its error page to the cause. Kubernetes comes
first; [standalone](#standalone-vm-docker-or-a-laptop) (one binary and one YAML file) is below.

## Kubernetes

Supported: Kubernetes 1.31 and later.

### 1. Install

```sh
# Gateway API CRDs (skip on platforms that manage them)
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.6.3/standard-install.yaml

helm install lygard oci://ghcr.io/lygard-ingress/charts/lygard -n lygard --create-namespace \
  -f https://raw.githubusercontent.com/lygard-ingress/lygard/main/deploy/helm/lygard/presets/linode.yaml
```

Pick the preset for your cloud:

- **Layer 4 load balancers:** `aws`, `aws-automode`, `azure`, `gcp`, `digitalocean`, `linode`
  (plus `linode-premium`), `oke`, `hetzner`, `metallb`, `openshift`.
- **Behind the cloud's layer 7 load balancer:** `aws-alb`, `azure-appgw`, `azure-agc`, `gcp-l7`,
  `oke-l7`, `linode-l7`.

Each preset sets the load balancer type, PROXY protocol and health checks so that rollouts drop
no connections; [cloud providers](cloud-providers.md) has the details. To size the install, add
`presets/light.yaml` (smallest footprint) or `presets/high-scale.yaml` (many connections).

Everything Lygard needs is created at install time, including the admin token Secret and the
configuration ConfigMap. After that, it reads your Ingresses, routes, Services, policies and
Secrets.

The only things it writes are:

- the status of those objects;
- events and a leader-election lease in its own namespace;
- a ConfigMap in its own namespace, only if you use progressive rollouts;
- certificate Secrets, only if you turn on built-in ACME (`acme.enabled`).

It never changes your ConfigMaps or Secrets, and it never changes its own configuration at run
time. Configuration changes come from you, through `kubectl apply` or `helm upgrade`.

### 2. Put a site behind it

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  namespace: default
  annotations:
    lygard.io/waf: detect                       # record attacks, block nothing yet
    lygard.io/waf-rulesets: sensitive-files     # .env, .git, backups, keys
    lygard.io/rate-limit-rps: "50"
spec:
  ingressClassName: lygard
  rules:
    - host: app.example.com
      http:
        paths:
          - {path: /, pathType: Prefix, backend: {service: {name: web, port: {number: 80}}}}
```

```sh
kubectl apply -f web.yaml
kubectl get ingress web          # ADDRESS is Lygard's load balancer
curl -H "Host: app.example.com" http://<address>/
```

To use the Gateway API instead, create a `Gateway` with `gatewayClassName: lygard` and attach
`HTTPRoute`s to it.

### 3. Open the dashboard

```sh
kubectl -n lygard get secret lygard-admin -o jsonpath='{.data.token}' | base64 -d; echo
kubectl -n lygard port-forward svc/lygard-admin 9901
# open http://localhost:9901 and paste the token
```

The **Generate** section writes Ingresses, HTTPRoutes, LygardPolicies, Helm values and Secrets
for you, with every field explained. You apply what it produces; the dashboard does not change
the cluster. See [dashboard](dashboard.md).

### 4. Follow one request

Send something the WAF dislikes and look at the answer:

```sh
curl -i -H "Host: app.example.com" "http://<address>/.env"
# x-request-id: 4bf92f3577b34da6a3ce929d0e0e4736
```

In detect mode the request still goes through; in block mode the client gets a 403 page that
shows the same ID. Paste the ID into **Request inspector** in the dashboard. It shows:

- the cause in one sentence;
- each step of the request: route, WAF, backend, response;
- the rule that matched and how to change it.

The same ID is the trace ID in your tracing backend and appears in the access log. See
[observability](observability.md).

### 5. Turn protection on

1. Watch **Security → WAF events** and **Tools → WAF tuning** for a few days.
2. Fix any false positives the tuning view points out.
3. Then switch `lygard.io/waf` to `block`.

[WAF rules](waf-rules.md) covers rule packs (WordPress, PHP, bad bots, ...), custom rules with
`allow` / `block` / `count` / `captcha`, and IP reputation lists.

## Standalone (VM, Docker or a laptop)

The same data plane runs from one YAML file, without Kubernetes:

```yaml
# lygard.yaml
listeners:
  - {name: http, address: 0.0.0.0:8080}
backends:
  - name: app
    endpoints: [10.0.0.10:3000, 10.0.0.11:3000]
    healthCheck: {path: /healthz}
routes:
  - name: app
    hosts: [app.example.com]
    backends: [{name: app}]
    policy:
      waf: {mode: detect, rulesets: [sensitive-files]}
      rateLimit: {rps: 50, burst: 100}
```

```sh
lygard validate -c lygard.yaml                       # check it
lygard explain  -c lygard.yaml "http://app.example.com/.env"   # which route, which policy, which WAF verdict
lygard run      -c lygard.yaml                       # dashboard and /metrics on 127.0.0.1:9901
```

Lygard reloads the file when it changes and keeps serving the previous configuration if the new
one is invalid. For automatic HTTPS see [certificates](certificates.md). Every setting is listed
in the [configuration reference](configuration.md).

## Next

| To | Read |
|---|---|
| Configure routes and policy | [Annotations](annotations.md), [LygardPolicy](policy.md), [configuration reference](configuration.md) |
| Protect a site | [WAF rules](waf-rules.md), [authentication](authentication.md), [hardening](hardening.md) |
| See what happens | [Observability](observability.md), [dashboard](dashboard.md) |
| Run it in production | [Cloud providers](cloud-providers.md), [upgrading](upgrading.md), [performance](performance.md) |
| Move from ingress-nginx | [Migrating from ingress-nginx](migrating-from-ingress-nginx.md) |
