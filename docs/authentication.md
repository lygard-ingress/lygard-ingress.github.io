# Authentication

Lygard authenticates at the edge in four ways, which can be combined on one route (basic and JWT
first, then external auth, then the WAF and plugins).

## Basic auth

```sh
htpasswd -nbB admin 's3cret' > auth
kubectl create secret generic admin-htpasswd --from-file=auth
kubectl label secret admin-htpasswd lygard.io/secret=true
```

```yaml
metadata:
  annotations:
    lygard.io/auth-basic-secret: admin-htpasswd
    lygard.io/auth-realm: Admin
```

Only bcrypt lines are accepted. A missing Secret makes the route refuse every request.

## JWT

```yaml
metadata:
  annotations:
    lygard.io/jwt-issuer: https://login.example.com/realms/shop
    lygard.io/jwt-audiences: orders-api
    lygard.io/jwt-jwks-url: https://login.example.com/realms/shop/protocol/openid-connect/certs
    lygard.io/jwt-required-claims: "scope=orders:write; realm_access.roles=admin|ops"
    lygard.io/jwt-claims-to-headers: "sub:x-user, email:x-email"
```

No token or an invalid one: 401 with `WWW-Authenticate: Bearer`. A valid token without the
required claims: 403. JWKS documents are cached and refreshed every 10 minutes (and on an
unknown `kid`, at most every 30 s).

## External auth and OIDC login (forward auth)

Before proxying, Lygard sends a sub-request to an auth service with the client's credentials and
the original request's method, URL and client address (`X-Forwarded-Method`, `X-Forwarded-Proto`,
`X-Forwarded-Host`, `X-Forwarded-Uri`, `X-Original-URL`, `X-Forwarded-For`). A 2xx lets the request
through; any other answer is returned to the client unchanged (status, `Location`,
`WWW-Authenticate`, `Set-Cookie`, body). This is the contract of ingress-nginx `auth-url` and
Traefik ForwardAuth, so oauth2-proxy, Authelia, Pomerium and similar work as they are, which is
also how to add an OIDC login (Google, Entra ID, Okta, Keycloak, ...) to any route.

With [oauth2-proxy](https://oauth2-proxy.github.io/oauth2-proxy/) in namespace `auth`:

```yaml
metadata:
  annotations:
    lygard.io/auth-url: http://oauth2-proxy.auth.svc.cluster.local/oauth2/auth
    lygard.io/auth-signin: https://app.example.com/oauth2/start
    lygard.io/auth-response-headers: X-Auth-Request-User, X-Auth-Request-Email
```

Route `/oauth2/*` on the same host to oauth2-proxy itself (a second Ingress path or HTTPRoute
rule). On a 401 the browser is redirected to `auth-signin?rd=<original URL>`. If the auth
service is unreachable or times out (`auth-timeout`, default 2 s) the request gets 503, unless
`auth-fail-open: "true"`.

## Client certificates (mTLS) on Gateway listeners

Gateway API frontend validation is supported in `gatewayAPI.addresses: per-gateway` mode (each
Gateway has its own listeners and load balancer, so its client CA applies only to it):

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata: {name: partners, namespace: b2b}
spec:
  gatewayClassName: lygard
  tls:
    frontend:
      default:
        validation:
          caCertificateRefs:
            - {kind: ConfigMap, name: partner-ca}   # key ca.crt
          mode: AllowValidOnly                      # or AllowInsecureFallback
  listeners:
    - name: https
      port: 443
      protocol: HTTPS
      tls: {certificateRefs: [{name: partners-tls}]}
```

`AllowValidOnly` refuses the TLS handshake without a certificate signed by one of the CAs.
`AllowInsecureFallback` asks for one but lets clients without (or with an invalid one) through,
and sets the Gateway condition `InsecureFrontendValidationMode`. `perPort` overrides the default
for one port. An invalid CA reference sets the listener to `Accepted=False`
(`NoValidCACertificate`) with `ResolvedRefs=False` (`InvalidCACertificateRef` /
`InvalidCACertificateKind`).

In the default shared address mode, a Gateway that asks for client validation has its HTTPS
listeners marked `Accepted=False` (`UnsupportedValue`) rather than served without the check.
