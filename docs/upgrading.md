# Upgrading

```sh
helm upgrade lygard oci://ghcr.io/lygard-ingress/charts/lygard -n lygard \
  --reset-then-reuse-values --server-side=false --wait
```

- `--reset-then-reuse-values` takes the new chart's defaults and re-applies your own values;
  plain `--reuse-values` would keep old defaults and miss settings added since.
- `--server-side=false`: Helm 4 defaults to server-side apply, which conflicts with objects an
  earlier client-side install created (`field-manager conflicts`). Stay on one mode per release.
- Upgrades are zero-downtime by design: new data plane pods must be ready for `minReadySeconds`
  before old ones go, and old ones fail readiness, keep serving for `shutdownDelay` (answering
  HTTP/1 with `Connection: close` so clients move), then drain.
- Roll back with `helm rollback lygard <revision>`.
- CRDs in the chart's `crds/` directory (LygardPolicy) are installed on first install only, as
  Helm does for every chart. To update them: `kubectl apply --server-side -f deploy/helm/lygard/crds/`.

## Gateway API CRDs

Lygard does not install the Gateway API CRDs. Use the standard or experimental bundle of
Gateway API v1.6 (v1.5 still works: TCPRoute/UDPRoute are then read at v1alpha2):

```sh
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.6.3/experimental-install.yaml
```

On platforms that manage these CRDs (OpenShift 4.19+, AKS with the managed Gateway API, DOKS),
use the platform's copy.
