# Hardening

What Lygard does by default, and the settings to review before exposing it to the internet.

## Defaults you get

- **Admin API and dashboard** (`global.adminAddress`, port 9901) need `Authorization: Bearer
  <token>`. Without a token they answer only on a loopback address; anywhere else they return 401
  and Lygard logs a warning. `/metrics`, `/healthz` and `/readyz` stay open for probes and
  scrapers. The Helm chart generates a token and keeps it in a Secret. Never publish the admin
  port through a route (for example to reach the dashboard over the internet) without a token:
  the proxy connects to it from loopback, where a token-less admin API is open. With a token set,
  such a route is safe; the Akamai Quick Deploy App works this way.
- **Containers** run as non-root with a read-only root filesystem, every capability dropped and
  the `RuntimeDefault` seccomp profile.
- **Forwarded headers**: `X-Forwarded-For`, `X-Forwarded-Proto` and `X-Real-IP` from clients are
  replaced, not trusted, unless the peer is in `trustedProxies`. Headers that Lygard fills from
  verified JWT claims (`jwt.claimsToHeaders`) or from the external auth service
  (`extAuth.responseHeaders`) are removed from the client's request first, so they cannot be
  forged when a token or session lacks them.
- **Paths** are normalized before routing, policies and the backend see them: `.` and `..`
  segments (also `..;` and percent-encoded dots), repeated slashes, and percent-encoded unreserved
  characters (`/%61dmin` is `/admin`). Encoded slashes (`%2F`) and backslashes are kept, because
  decoding them changes the path for many backends; block them with a route if your backend
  treats them as separators.
- **Request smuggling**: a request with both `Transfer-Encoding` and `Content-Length` is
  forwarded chunked only; hop-by-hop headers (including those named in `Connection`) are stripped.
- **HTTP/2**: rapid-reset protection, header list size limits, a cap on concurrent streams.
- **CORS**: with `allowCredentials`, list the origins. `allowOrigins: ["*"]` plus credentials lets
  any website make credentialed requests (the Gateway API requires echoing the origin then), and
  Lygard warns about it. The `null` origin matches only when listed.
- **ExternalName Services** are refused as backends unless the operator enables them
  (`controller.allowExternalNameServices`); even then loopback, link-local (cloud metadata) and
  other namespaces' in-cluster names are refused.

## Settings to review

| Setting | Default | Why |
|---|---|---|
| `global.bodyReadTimeout` | 30 s | Time to receive the part of a body Lygard holds before proxying (WAF inspection, plugins, mirrors); slow clients get 408 instead of holding a connection |
| `global.headerReadTimeout` | 15 s | Time to receive request headers |
| `global.maxConnections` | 50,000 | All connections, all listeners |
| `global.maxConnectionsPerIp` | 0 (off) | Per client address. Set it (e.g. 256) when clients connect directly; leave it off behind a load balancer that does not keep client addresses |
| `global.trustedProxies` / `trustedHops` | none / 0 | Only the load balancer's addresses: anyone listed can set the client address and scheme |
| `waf.maxInspectBytes` | 64 KB | Request body the WAF inspects |
| `waf.oversizeAction` | `pass` | `reject` refuses (413, in block mode) bodies larger than the inspected part, so a payload cannot hide behind padding; it also refuses larger uploads on that route |
| `controller.allowExternalNameServices` | false | See above |
| `networkPolicy.enabled` | false | Restricts the admin port to listed namespaces |

## Multi-tenant clusters

With Ingress, any namespace allowed to create Ingresses of Lygard's class can add rules for any
hostname, including a more specific path on another team's host, and can ask for an ACME
certificate for it. Ingress has no ownership model; this is the same with every Ingress
controller. Where namespaces are not trusted:

- Use the Gateway API: the platform team owns the Gateway, its listeners restrict hostnames, and
  `allowedRoutes` decides which namespaces may attach routes.
- Restrict who can create Ingresses of Lygard's class (RBAC, or an admission policy on
  `spec.ingressClassName`).
- Keep `controller.allowExternalNameServices` off and wasm plugins operator-defined (routes can
  only attach plugins the operator configured).

## Reporting

Security issues: see SECURITY.md.
