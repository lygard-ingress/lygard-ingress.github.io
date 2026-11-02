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
