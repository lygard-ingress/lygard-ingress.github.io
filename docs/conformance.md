# Gateway API conformance

Lygard runs the official [Gateway API conformance suite](https://gateway-api.sigs.k8s.io/concepts/conformance/)
(v1.6.3, experimental channel) on every release. The report for each release is committed in
`controller/conformance/`.

## Unreleased (main)

Kubernetes 1.37.0 in kind with MetalLB (`LB=metallb tests/conformance-kind.sh v1.37.0
per-gateway`), `gatewayAPI.addresses: per-gateway`, 590 tests passed, 0 failed.

| Profile | Core | Extended |
|---|---|---|
| GATEWAY-HTTP | 36 passed, 0 failed | 42 passed, 0 failed |
| GATEWAY-GRPC | 14 passed, 0 failed | 2 passed, 0 failed |
| GATEWAY-TLS | 19 passed, 0 failed | 6 passed, 0 failed |
| GATEWAY-TCP | 18 passed, 0 failed | 2 passed, 0 failed |

New extended features claimed: `GatewayStaticAddresses`, `TLSRouteModeTerminate`,
`TLSRouteModeMixed`. Static addresses need a load balancer that honours a Service's
`loadBalancerIP` (MetalLB here); cloud-provider-kind does not.

## v0.13.0

Same results as v0.12.0 below for the Gateway profiles (report
`report-per-gateway-v0.13.0-k8s-v1.37.0.yaml`), and the service mesh profile:

| Profile | Core | Extended |
|---|---|---|
| MESH-HTTP (GAMMA) | 7 passed, 0 failed | 5 passed, 0 failed |

Mesh features claimed: `Mesh`, `MeshClusterIPMatching`, `MeshConsumerRoute`,
`MeshHTTPRouteQueryParamMatching`, `MeshHTTPRouteSchemeRedirect`, `MeshHTTPRouteRedirectPort`,
`MeshHTTPRouteRedirectPath`, `MeshHTTPRouteNamedRouteRule`, plus 303/307/308 redirects. Not yet:
`MeshHTTPRouteRewritePath`, `MeshHTTPRouteBackendRequestHeaderModification`, GRPCRoute in the
mesh. Reproduce with `MESH=1 tests/conformance-kind.sh v1.37.0`
(report `report-mesh-v0.13.0-k8s-v1.37.0.yaml`).

## v0.12.0

Kubernetes 1.37.0 in kind, `gatewayAPI.addresses: per-gateway`, 581 tests passed, 0 failed.

| Profile | Core | Extended |
|---|---|---|
| GATEWAY-HTTP | 36 passed, 0 failed | 41 passed, 0 failed |
| GATEWAY-GRPC | 14 passed, 0 failed | 1 passed, 0 failed |
| GATEWAY-TLS | 19 passed, 0 failed | 1 passed, 0 failed |
| GATEWAY-TCP | 18 passed, 0 failed | 1 passed, 0 failed |

Extended features claimed (and tested):

- **Gateway:** `GatewayAddressEmpty`, `GatewayPort8080`, `GatewayHTTPListenerIsolation`,
  `GatewayHTTPSListenerDetectMisdirectedRequests`, `GatewayFrontendClientCertificateValidation`,
  `GatewayFrontendClientCertificateValidationInsecureFallback`
- **HTTPRoute:** `HTTPRoute303RedirectStatusCode`, `HTTPRoute307RedirectStatusCode`,
  `HTTPRoute308RedirectStatusCode`, `HTTPRouteBackendProtocolH2C`, `HTTPRouteBackendProtocolWebSocket`,
  `HTTPRouteBackendRequestHeaderModification`, `HTTPRouteBackendTimeout`, `HTTPRouteCORS`,
  `HTTPRouteDestinationPortMatching`, `HTTPRouteHostRewrite`, `HTTPRouteMethodMatching`,
  `HTTPRouteNamedRouteRule`, `HTTPRouteParentRefPort`, `HTTPRoutePathRedirect`, `HTTPRoutePathRewrite`,
  `HTTPRoutePortRedirect`, `HTTPRouteQueryParamMatching`, `HTTPRouteRequestMirror`,
  `HTTPRouteRequestMultipleMirrors`, `HTTPRouteRequestPercentageMirror`, `HTTPRouteRequestTimeout`,
  `HTTPRouteResponseHeaderModification`, `HTTPRouteRetry`, `HTTPRouteRetryBackendTimeout`,
  `HTTPRouteRetryConnectionError`, `HTTPRouteSchemeRedirect`
- **GRPCRoute:** `GRPCRouteNamedRouteRule`
- **Other:** `BackendTLSPolicy`, `BackendTLSPolicySANValidation`, `ReferenceGrant`, `TCPRoute`

Not supported yet (see [ROADMAP.md](roadmap.md)):

| Feature | Why |
|---|---|
| `GatewayInfrastructure` | The test looks for Pods, ServiceAccounts or Services in the Gateway's namespace; Lygard's per-Gateway Service (which gets the labels and annotations) lives in Lygard's namespace |
| `ListenerSet` | Not implemented |

UDPRoute is served (DNS, QUIC passthrough) but has no conformance profile in v1.6.3. Neither do
two features Lygard implements: a Gateway client certificate to backends
(`spec.tls.backend.clientCertificateRef`) and HTTPRoute session persistence; Lygard's own tests
cover them.

In the default shared address mode (every Gateway behind one load balancer), frontend client
certificate validation and listener isolation across Gateways are not available; run conformance
in `per-gateway` mode to get the numbers above.

## Reproduce

Needs Docker, [kind](https://kind.sigs.k8s.io/) v0.33.0 or newer, and
[cloud-provider-kind](https://github.com/kubernetes-sigs/cloud-provider-kind) v0.12.0.

```sh
docker build -t lygard:matrix .
docker build -t lygard-controller:matrix controller
tests/conformance-kind.sh v1.37.0 per-gateway   # PROFILES=... FEATURES=... to narrow
```

The report is written to `controller/conformance/report-<mode>-v<version>-k8s-<kubernetes>.yaml`.

## Kubernetes versions

`tests/k8s-matrix/run.sh` covers what conformance does not: install from the chart, Ingress and
HTTPRoute routing, two data plane rollouts and one node drain under 200 requests per second, with
no failed request allowed. v0.12.0 results:

| Kubernetes | Routing | Rollout x2 under load | Node drain under load |
|---|---|---|---|
| 1.31.14 | Pass | 15,000 requests, 0 failed | 12,000 requests, 0 failed |
| 1.32.11 | Pass | 15,001 requests, 0 failed | 12,000 requests, 0 failed |
| 1.33.12 | Pass | 15,001 requests, 0 failed | 12,000 requests, 0 failed |
| 1.34.11 | Pass | 15,000 requests, 0 failed | 12,000 requests, 0 failed |
| 1.35.8 | Pass | 14,998 requests, 0 failed | 12,000 requests, 0 failed |
| 1.36.4 | Pass | 15,000 requests, 0 failed | 12,000 requests, 0 failed |
| 1.37.0 | Pass | 15,000 requests, 0 failed | 12,000 requests, 0 failed |
