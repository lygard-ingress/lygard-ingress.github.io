# Cloud providers

Lygard is exposed through a Service of type `LoadBalancer`. Each cloud's load balancer differs in
how it passes the client address, whether it speaks PROXY protocol or UDP, and how fast it
notices a pod going away, so the chart ships one preset per cloud under
`deploy/helm/lygard/presets/`. Each preset sets the load
balancer type, PROXY protocol on both sides when needed, health checks, and drain timing
(`shutdownDelay`, `minReadySeconds`, `terminationGracePeriodSeconds`) longer than that load
balancer's failure-detection time, so rollouts and node drains drop no connections.

```sh
helm install lygard oci://ghcr.io/lygard-ingress/charts/lygard -n lygard --create-namespace \
  -f presets/<cloud>.yaml [-f presets/light.yaml | -f presets/high-scale.yaml]
```

| Cloud | Preset | Load balancer | Client IP | UDP | Notes |
|---|---|---|---|---|---|
| AWS EKS (AWS Load Balancer Controller) | `aws.yaml` | NLB, IP targets | PROXY v2 | Yes | Label the namespace `elbv2.k8s.aws/pod-readiness-gate-inject=enabled` so NLB target health gates rollouts. `shutdownDelay` 40 s covers the NLB's deregistration delay |
| AWS EKS Auto Mode | `aws-automode.yaml` | NLB (`eks.amazonaws.com/nlb`) | PROXY v2 via target group attribute | Yes | Auto Mode ignores the proxy-protocol annotation |
| Azure AKS | `azure.yaml` | Standard Load Balancer | Preserved (Local) | Yes | No PROXY protocol on public Azure LBs |
| Google GKE | `gcp.yaml` | Backend-service passthrough NLB | Preserved (Local) | Yes | Health check 3 s x 2 with Local |
| Akamai / Linode LKE | `linode.yaml` (+ `linode-premium.yaml`) | NodeBalancer | PROXY v2 | Premium + account opt-in | Basic: 10,000 concurrent connections, Premium: 100,000. The controller marks UDP ports `proxy-protocol: none` (the CCM rejects PROXY on UDP) |
| DigitalOcean DOKS | `digitalocean.yaml` | Regional LB (`REGIONAL`) | PROXY | No | The default type since DOKS 1.33 (`REGIONAL_NETWORK`) has no PROXY protocol; the preset pins `REGIONAL`. 10,000 connections per size unit |
| Oracle OKE | `oke.yaml` | Network Load Balancer | Preserved (Local) | Yes | Health checks tightened from the ~33 s default to ~7 s |
| Hetzner Cloud | `hetzner.yaml` | Cloud Load Balancer | PROXY | No | lb11 10,000 / lb21 20,000 / lb31 40,000 connections; set the location to your cluster's |
| Bare metal (MetalLB) | `metallb.yaml` | MetalLB L2 or BGP | Preserved (Local) | Yes | DaemonSet; no LB health checks, so readiness and drain timing matter |
| OpenShift 4.18+ | `openshift.yaml` + platform preset | Platform's | Platform's | Platform's | The Ingress Operator owns the Gateway API CRDs and blocks the experimental channel (no TCPRoute/UDPRoute/TLSRoute there); needs the `nonroot-v2` SCC |

HTTP/3 needs UDP and no PROXY protocol on the UDP port; [docs/http3.md](http3.md) has the
per-cloud table. The Azure, GCP, OKE and MetalLB presets turn it on.

Other clouds (Alibaba ACK, IBM IKS, ...) work as a generic `LoadBalancer` Service; take the
closest preset and set the provider's annotations.

## Behind a layer 7 load balancer

The presets above put Lygard behind each cloud's layer 4 load balancer, which passes TCP
through. Lygard can also sit behind the cloud's managed layer 7 load balancer, when you want the
cloud's TLS certificates (ACM, Key Vault, Google-managed), its edge WAF or CDN, or one load
balancer shared with other services. The cloud's own controller builds the load balancer from an
Ingress or Gateway that the chart creates (`frontend` in values.yaml) and sends every request to
Lygard, which still does the routing, WAF, auth and rate limiting.

| Cloud | Preset | Load balancer | `trustedProxies` | Lygard `idleTimeout` |
|---|---|---|---|---|
| AWS EKS (AWS Load Balancer Controller) | `aws-alb.yaml` | Application Load Balancer, IP targets | ALB subnets or VPC CIDR (yours) | 120 s, above the ALB's 60 s |
| Azure AKS (AGIC) | `azure-appgw.yaml` | Application Gateway v2 | Application Gateway subnet (yours) | 120 s |
| Azure AKS | `azure-agc.yaml` | Application Gateway for Containers | Association subnet (yours) | 320 s, above AGC's 5 min |
| Google GKE (Gateway) | `gcp-l7.yaml` | Global external Application Load Balancer | Google front ends (in the preset); regional: add your proxy-only subnet | 620 s, above Google's 600 s |
| Oracle OKE (OCI Native Ingress Controller) | `oke-l7.yaml` | OCI flexible load balancer | Load balancer subnet (yours) | 320 s, above OCI's 300 s |
| Akamai / Linode LKE | `linode-l7.yaml` | NodeBalancer in HTTP/HTTPS mode | `192.168.255.0/24` (in the preset) | 120 s |

```sh
helm install lygard oci://ghcr.io/lygard-ingress/charts/lygard -n lygard --create-namespace \
  -f presets/aws-alb.yaml --set 'config.global.trustedProxies={10.0.0.0/16}' \
  --set-string 'frontend.ingress.annotations.alb\.ingress\.kubernetes\.io/certificate-arn=arn:aws:acm:...'
```

What the presets take care of, and what to keep in mind when writing your own:

- **Client address.** Lygard believes `X-Forwarded-For` only from `trustedProxies`, so set them to
  the addresses the load balancer connects from. The chart refuses to render a `frontend` without
  them: every client would appear as the load balancer. Google's load balancers append their own
  address after the client's (`<client>, <load balancer>`), so `gcp-l7.yaml` also sets
  `trustedHops: 1`. Application Gateway v2 writes entries as `IP:port`; Lygard reads that form.
- **HTTPS seen by the client.** The load balancer ends TLS and forwards plain HTTP. Lygard takes
  the scheme from `X-Forwarded-Proto` when the peer is trusted, so `httpsRedirect`, HSTS,
  redirects and the scheme passed to external auth and plugins behave as if Lygard ended TLS.
- **Idle connections.** The load balancer reuses connections to Lygard; if Lygard closes an idle
  one first, a request sent on it fails with a 502. Lygard's `idleTimeout` must be longer than
  the load balancer's backend idle timeout (the table above).
- **Health checks** go to Lygard's admin port (`9901`, `/readyz`), which fails as soon as a pod
  starts shutting down: ALB, AGIC and OCI through Ingress annotations, AGC and GKE through
  `HealthCheckPolicy`. The load balancer must be allowed to reach port 9901 on the pods.
- **Draining.** The presets turn on or shorten each load balancer's connection draining (off by
  default on Application Gateway and GKE, 300 s on ALB) and set `shutdownDelay: 30s`.
- **TLS to Lygard** is off: the load balancer sends HTTP to port 80. To re-encrypt, set
  `frontend.backendPort: https` and the cloud's backend-protocol setting (ALB does not verify
  backend certificates; Application Gateway v2 accepts only a well-known CA or an uploaded root).
- **HTTP/3** is the load balancer's to offer, not Lygard's, and PROXY protocol is off.

The Linode preset works differently: a NodeBalancer in HTTP mode is configured on Lygard's own
`LoadBalancer` Service (no Ingress), with the certificate in a Secret
(`lygard-nodebalancer-tls`); `service.httpsTargetPort: http` sends its decrypted 443 traffic to
Lygard's HTTP listener, and `externalTrafficPolicy: Local` keeps the NodeBalancer's address as the
peer. Tried on LKE 1.37 (2026-11-03): the NodeBalancer connects from 192.168.255.0/24 and sends
`X-Forwarded-For: <client>` and `X-Forwarded-Proto`; plain HTTP is redirected, HTTPS is served
with HSTS and no redirect loop, the real client address is logged (a client-supplied
X-Forwarded-For is ignored), and a rolling restart under load dropped no requests (1,434 of 1,434).
The other layer 7 presets have not been tried on their clouds yet.

Sources: AWS [X-Forwarded headers](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/x-forwarded-headers.html),
[ALB attributes](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/edit-load-balancer-attributes.html),
[controller annotations](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/ingress/annotations/);
Azure [how Application Gateway works](https://learn.microsoft.com/en-us/azure/application-gateway/how-application-gateway-works),
[AGIC annotations](https://learn.microsoft.com/en-us/azure/application-gateway/ingress-controller-annotations),
[AGC components](https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/application-gateway-for-containers-components),
[AGC health probes](https://learn.microsoft.com/en-us/azure/application-gateway/for-containers/custom-health-probe);
Google [Application Load Balancer headers and ranges](https://docs.cloud.google.com/load-balancing/docs/https),
[timeouts](https://docs.cloud.google.com/load-balancing/docs/https/request-distribution),
[Gateway policies](https://docs.cloud.google.com/kubernetes-engine/docs/how-to/configure-gateway-resources);
Oracle [load balancer headers](https://docs.oracle.com/en-us/iaas/Content/Balance/Reference/httpheaders.htm),
[connection reuse](https://docs.oracle.com/en-us/iaas/Content/Balance/Reference/connectionreuse.htm),
[native ingress annotations](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengsettingupnativeingresscontroller-configuring.htm).

## One load balancer per Gateway

With `gatewayAPI.addresses: per-gateway` each Gateway gets its own LoadBalancer Service, which
starts from the data plane Service's annotations. Two Gateway fields shape it:

- `spec.infrastructure.labels` and `annotations` are added to the Gateway's Service. Annotations
  set there override the inherited ones, so one Gateway can ask its cloud for a different load
  balancer type, an internal address or a firewall, for example.
- An IP address in `spec.addresses` becomes the Service's `loadBalancerIP`, for providers that
  honour it (MetalLB, Azure, GKE and others). Providers that reserve IPs through an annotation
  instead take it through `spec.infrastructure.annotations`. Until the load balancer reports the
  requested IP, the Gateway is `Programmed: False` with reason `AddressNotUsable`. Hostname and
  other address types are rejected (`Accepted: False`, `UnsupportedAddress`).

```yaml
spec:
  addresses:
    - {type: IPAddress, value: 192.0.2.10}
  infrastructure:
    labels: {team: payments}
    annotations:
      service.beta.kubernetes.io/azure-load-balancer-internal: "true"
```

In `shared` mode every Gateway uses the data plane Service, so `infrastructure` has no effect, and
an address the shared load balancer does not have leaves the Gateway `AddressNotUsable`.

## In-cluster callers and PROXY protocol

With PROXY protocol on, pods that call the load balancer's IP are short-circuited to Lygard by
kube-proxy and arrive without the PROXY header (cert-manager's HTTP-01 self-check is the usual
case). Lygard accepts headerless connections from `proxyProtocolOptionalFrom` (private ranges by
default) and peeks for the header from those sources, so both kinds of traffic work.

## Capacity

An idle keep-alive connection costs about 11 KB of data plane memory, so a pod with a 4 GiB limit
holds about 200,000 (`presets/high-scale.yaml`). The load balancer usually runs out first: check
its connection limit in the table above, and add load balancers (per-Gateway address mode gives
each Gateway its own) or move to a larger tier before adding pods.

### Measured: Linode NodeBalancer (basic)

One VM in the cluster's region opening keep-alive HTTP/1.1 connections through a basic
NodeBalancer (PROXY v2) to two data plane pods, each connection sending a request every 10 s
(0.13.0, 2026-11-02):

| Connections | Kept open | Failed requests | Note |
|---|---|---|---|
| 2,000 / 5,000 | all | 0 | p50 3-4 ms |
| 8,000 / 9,500 / 10,000 | all | 0 | latency up (p50 270-400 ms): the 2-CPU node was CPU-bound, Lygard used 0.2-0.5 CPU per pod |
| 10,500 / 12,000 | 10,000 | every connection above 10,000 | accepted by the NodeBalancer, closed at its first request |

The 10,000 limit is exact and the failure is silent at connect time: clients see the TCP
connection open and then reset. Watch the NodeBalancer's connection graph, not connect errors.

## Kubernetes versions

Every release is tested on Kubernetes 1.31 to 1.37 in kind (`tests/k8s-matrix/run.sh`): routing
through Ingress and Gateway API, two data plane rollouts and a node drain under steady load, with
zero failed requests allowed.
