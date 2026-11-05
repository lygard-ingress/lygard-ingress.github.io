# Dashboard

The dashboard is served on the admin port (`/dashboard`) and needs the admin token. A viewer
token gives a read-only view. It shows what Lygard is doing and helps you write configuration,
but it never changes your cluster:

- it does not create or edit Ingresses, routes, policies, ConfigMaps or Secrets;
- everything it produces is YAML or commands for you to review and apply with `kubectl apply`,
  `helm upgrade` or a Git commit.

This works in clusters where nobody is allowed to change ConfigMaps or Secrets after
installation, and it keeps Git as the source of truth.

```sh
kubectl -n lygard port-forward deploy/lygard 9901:9901
kubectl -n lygard get secret lygard-admin -o jsonpath='{.data.token}' | base64 -d   # the token
open http://localhost:9901/dashboard
```

| Section | What it is for |
|---|---|
| Overview, Traffic | Requests per second, errors, latency, per-route and per-backend health |
| Request inspector | Find one request by request ID or trace ID and see why it failed ([observability.md](observability.md)) |
| Security | WAF events, live requests, TLS fingerprints, anomalies |
| WAF rules | Toggle rule packs, categories, IP lists and headers for one route; test a request against the live rules ([waf-rules.md](waf-rules.md)) |
| Generate | Write Ingress, HTTPRoute, LygardPolicy, Helm values, Secrets and ConfigMaps; suggestions from traffic; the rule catalog |
| API | Endpoints learnt from traffic, shadow APIs, OpenAPI export |
| Tools | Traffic capture and replay, WAF tuning, explain a URL |
| Configuration | Certificates and the active configuration (secrets redacted) |

## Generate

Every field explains itself, and the explanation is written into the YAML as a comment, so the
file documents itself when someone reviews it later.

| Tab | Produces | Apply with |
|---|---|---|
| Ingress | An Ingress with `ingressClassName: lygard` and the chosen `lygard.io/*` annotations (WAF, rule packs, IP lists, rate limit, JWT, basic auth, headers, body size, timeout) | `kubectl apply -f` |
| HTTPRoute | A Gateway API HTTPRoute with header filters and timeouts | `kubectl apply -f` |
| LygardPolicy | All route policy as one object: WAF, packs, categories, IP lists, rate limit, JWT and required claims, CORS, security headers, a custom 403 page, body limit, timeout | `kubectl apply -f` |
| Helm values | Installation-wide settings: trusted proxies, OpenTelemetry and the trace link, error pages, captcha provider, named IP lists, request history | `helm upgrade -f values.yaml` |
| Secrets & ConfigMaps | A basic-auth Secret (Lygard hashes the password with bcrypt and stores nothing), the command for a TLS Secret, a captcha secret, an IP-list ConfigMap with its mount | `kubectl apply -f` |
| Suggestions | Advice from observed traffic, each with the reason and a ready LygardPolicy | review, then `kubectl apply -f` |
| Rule catalog | Every WAF rule, described below | – |

The Rule catalog lists every WAF rule with:

- what it detects and an example attack;
- what it inspects, and its paranoia level and score (attack rules) or its exact conditions
  (pack rules);
- its false-positive risk;
- YAML for three ways to change it on a route: switch it off, exclude one field, or record
  instead of blocking.

### Suggestions

Suggestions are heuristics over what API discovery (the API section) has seen on each route. Each
one says why it was made, so you can judge whether it fits.

| Suggestion | When |
|---|---|
| Turn the WAF on in detect mode | The route has traffic and no WAF |
| Protect sign-in endpoints | Paths such as `/login`, `/signin`, `/token` or `wp-login` and no rate limit (a rate limit plus a challenge) |
| Restrict admin and internal paths | `/admin`, `/internal`, `/actuator`, `/metrics` or `/debug` reachable without IP restriction or JWT |
| Sensitive data served without credentials | Responses contained emails, card numbers or tokens, and the requests carried no credentials |
| Set a body size limit | The route receives multipart uploads and has no `maxBodyBytes` |
| Add the WordPress or PHP pack | WordPress paths or `.php` scripts were seen |
| Skip WAF inspection for static files | More than a third of requests are plain static-file fetches |
| Endpoints failing often | 10% or more 5xx (look at them in the request inspector) |
| Slow endpoints without a timeout | Average above 2 s and no `requestTimeout` |
| Add security headers | The route sends none |

## API

Everything the dashboard shows comes from the admin API (`Authorization: Bearer <token>`).
These endpoints only read state: `/api/stats`, `/api/routes`, `/api/requests/<id>`,
`/api/inspector`, `/api/waf/events`, `/api/waf/tuning`, `/api/waf/rulesets`,
`/api/waf/catalog`, `/api/ip-lists` and `/api/suggestions`.

Two POST endpoints compute an answer without storing anything; both need the admin token:

- `POST /api/waf/test` runs a request you describe through a route's rules;
- `POST /api/generate/htpasswd` hashes a password.
