# LygardPolicy

`LygardPolicy` (`lygard.io/v1alpha1`, short name `lgp`) attaches route policy to Ingresses,
HTTPRoutes and GRPCRoutes as a typed object instead of annotations: easier to review in GitOps,
reusable across several routes, and reported on in its own status. The CRD ships in the chart's
`crds/` directory.

```yaml
apiVersion: lygard.io/v1alpha1
kind: LygardPolicy
metadata:
  name: public-api
  namespace: shop
spec:
  targetRefs:
    - {kind: HTTPRoute, name: orders}
    - {kind: Ingress, name: legacy-web}
  policy:
    requestTimeout: 10s
    rateLimit: {rps: 50, burst: 100, key: "header:x-api-key"}
    waf: {mode: block, engine: crs, paranoia: 1}
    geoDeny: [KP, IR]
    extAuth:
      url: http://oauth2-proxy.auth.svc.cluster.local/oauth2/auth
      signinUrl: https://shop.example.com/oauth2/start
      responseHeaders: [X-Auth-Request-User, X-Auth-Request-Email]
    errorPages:
      "503": {body: "<h1>Back in a minute</h1>"}
```

```console
$ kubectl get lgp -n shop
NAME         ACCEPTED   REASON     AGE
public-api   True       Accepted   2m
```

## Rules

- Targets must be in the policy's namespace. Kinds: `Ingress`, `HTTPRoute`, `GRPCRoute`.
- Fields set in `spec.policy` override the same settings from the target's annotations; settings
  the policy leaves out keep their annotation value (or the global default).
- Several policies on one target apply oldest first, so a newer policy wins field by field.
- `spec.policy` is decoded strictly: an unknown or misspelled field rejects the whole policy
  (`Accepted=False`, `Reason=Invalid`, the field named in the message) instead of being ignored.
- `status.attachedTo` lists the objects the policy was applied to; a target that does not exist
  or is not served by Lygard gives `Reason=TargetNotFound`.

## Fields

`spec.policy` has the same fields as a route `policy` in the data plane configuration (camelCase):
`requestTimeout`, `retry`, `rateLimit`, `waf`, `ipAllow`, `ipDeny`, `ipDenyLists`, `geoAllow`, `geoDeny`, `jwt`,
`extAuth`, `cors`, `securityHeaders`, `maxBodyBytes`, `httpsRedirect`, `accessLog`, `ja4Deny`,
`plugins`, `errorPages`. Their meaning matches the annotations in [annotations.md](annotations.md).

WAF condition rules, rule packs, attack categories and IP lists (`waf.rules`, `waf.rulesets`,
`waf.disabledCategories`, `ipDenyLists`) are described in [waf-rules.md](waf-rules.md).

Basic auth needs a Secret and stays an annotation (`lygard.io/auth-basic-secret`).
