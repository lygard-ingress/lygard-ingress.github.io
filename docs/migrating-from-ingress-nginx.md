# Migrating from ingress-nginx

The Kubernetes community ingress-nginx controller was retired in March 2026: its repositories are
archived and it receives no more releases or security fixes. Managed offerings built on it are
reaching end of support too (for example the AKS application routing add-on's NGINX in November
2026). Lygard reads the common `nginx.ingress.kubernetes.io/*` annotations, so most Ingresses move
by changing only the class.

## Steps

1. Install Lygard next to ingress-nginx (different class, different load balancer):

   ```sh
   helm install lygard oci://ghcr.io/lygard-ingress/charts/lygard -n lygard --create-namespace \
     -f https://raw.githubusercontent.com/lygard-ingress/lygard/main/deploy/helm/lygard/presets/aws.yaml
   ```

2. Copy an Ingress, change `spec.ingressClassName: nginx` to `lygard`, apply it under a new name,
   and test it through Lygard's load balancer address (`curl --resolve host:443:<lygard-ip>`).
   Check `kubectl -n lygard logs deploy/lygard-controller` for annotation warnings: every
   annotation Lygard cannot honour is named, with what to use instead.
3. Move DNS to Lygard's address, then delete the old Ingresses and ingress-nginx.

Alternatively take over the existing Ingresses in place: scale ingress-nginx to zero, then
install Lygard with `--set ingressClass.name=nginx --set ingressClass.create=false` (the
existing `nginx` IngressClass object is reused; Lygard serves Ingresses by class name). Point DNS
at Lygard's load balancer, and remove ingress-nginx once traffic has moved.

## Annotation mapping

Annotations that Lygard maps automatically:

| ingress-nginx | Lygard |
|---|---|
| `rewrite-target` | `lygard.io/rewrite-target` (prefix replacement; `$1` capture groups are not supported and are warned about) |
| `ssl-redirect`, `force-ssl-redirect` | same names |
| `proxy-body-size` | `max-body-size` (`0` = unlimited, Lygard's default) |
| `proxy-read-timeout` | `request-timeout` |
| `proxy-connect-timeout` | `connect-timeout` |
| `whitelist-source-range`, `allowlist-source-range`, `denylist-source-range` | `allow-cidrs`, `deny-cidrs` |
| `limit-rps` (+ `limit-burst-multiplier`, default 5), `limit-rpm` | `rate-limit-rps`, `rate-limit-burst` (cluster-wide by default) |
| `enable-cors` + `cors-allow-*`, `cors-expose-headers`, `cors-max-age` | `cors-*` with ingress-nginx's defaults |
| `auth-type: basic` + `auth-secret`, `auth-realm` | `auth-basic-secret`, `auth-realm` (label the Secret `lygard.io/secret=true`; bcrypt only) |
| `auth-url`, `auth-signin`, `auth-response-headers`, `auth-method` | external auth, same semantics |
| `backend-protocol: HTTPS / GRPC / GRPCS` | `backend-protocol` |
| `proxy-ssl-verify: "off"`, `proxy-ssl-name` | `backend-tls-insecure`, `backend-sni` |
| `affinity: cookie` + `session-cookie-name` | `lb: consistent_hash`, `hash-on: cookie:<name>` |
| `upstream-hash-by` (`$remote_addr`, `$http_x`, `$cookie_x`) | `hash-on` |
| `load-balance: round_robin / ewma / least_conn` | `lb` |
| `enable-modsecurity` (+ `enable-owasp-core-rules`) | `waf: detect` (+ `waf-engine: crs`) — DetectionOnly, as in ingress-nginx; set `lygard.io/waf: block` to enforce |

Annotations that are reported but not applied (each produces a warning with the replacement):
`server-snippet`, `configuration-snippet` (raw NGINX config), the `canary*` family (use HTTPRoute
weights or `lygard.io/rollout-canary`), `app-root`, `permanent-redirect`, `temporal-redirect`,
`from-to-www-redirect` (use an HTTPRoute `RequestRedirect`), `ssl-passthrough` (use a TLS
Gateway listener), `mirror-target` (HTTPRoute `RequestMirror`), `custom-http-errors` (use
`lygard.io/error-page-<code>`), `auth-tls-*` (client certificates: use a Gateway with frontend
validation), `limit-connections`, `satisfy`, `upstream-vhost`, `x-forwarded-prefix`.

Behaviour differences worth knowing:

- Lygard reloads nothing: Ingress, Secret and endpoint changes apply without worker restarts.
- Basic auth must use bcrypt (`htpasswd -B`); MD5/crypt lines are rejected.
- Refused or reset backend connections are retried once on another endpoint by default
  (ingress-nginx's `proxy_next_upstream error timeout`).
