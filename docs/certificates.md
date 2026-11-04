# Certificates

Lygard can keep TLS Secrets issued and renewed through ACME (Let's Encrypt by default, any
RFC 8555 CA). Each host is validated in one of two ways:

- **HTTP-01**: the gateway serves the CA's token at `http://<host>/.well-known/acme-challenge/…`.
  Nothing to configure beyond `acme.enabled`, but the host must already point at Lygard and port
  80 must be reachable from the internet. Cannot issue wildcards.
- **DNS-01**: the controller writes a TXT record at `_acme-challenge.<host>` through your DNS
  provider's API. Works for hosts that are not public yet, behind other proxies, or on private
  networks, and is the only way to get a wildcard (`*.example.com`) certificate.

A host uses DNS-01 when a DNS-01 solver covers its zone (or one is named with
`lygard.io/acme-solver`), and HTTP-01 otherwise. Both can be mixed in one certificate.

## Enabling

```yaml
# values.yaml
acme:
  enabled: true
  email: ops@example.com          # expiry notices from the CA
  # server: https://acme-staging-v02.api.letsencrypt.org/directory   # while testing
```

Enabling grants the controller `get`, `create` and `update` on Secrets cluster-wide (to write the
issued Secrets next to the objects that ask for them, and to read DNS provider credentials).

## Requesting a certificate

An Ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  annotations:
    lygard.io/acme: "true"
spec:
  ingressClassName: lygard
  tls:
    - hosts: [example.com, "*.example.com"]
      secretName: example-wildcard-tls
  rules:
    - host: "*.example.com"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: {service: {name: web, port: {number: 80}}}
```

A Gateway HTTPS listener (the listener's hostname is the certificate's name):

```yaml
listeners:
  - name: https
    protocol: HTTPS
    port: 443
    hostname: "*.example.com"
    tls:
      certificateRefs: [{name: example-wildcard-tls}]
      options:
        lygard.io/acme: "true"
        # lygard.io/acme-solver: cloudflare     # optional, see below
```

HTTP-01 hosts on a Gateway need an HTTP listener on the same Gateway to answer from; DNS-01
hosts do not.

The controller orders a certificate when the Secret is missing, does not cover the hosts, was
issued by another CA directory, or expires within 30 days. A Secret that already exists without
the `lygard.io/acme-managed: "true"` annotation is never touched.

## DNS-01 solvers

```yaml
acme:
  enabled: true
  email: ops@example.com
  dns01:
    solvers:
      - name: cloudflare               # optional; defaults to the provider name
        provider: cloudflare
        zones: [example.com]           # hosts in these zones (and their subdomains)
        secretRef: {name: cloudflare-dns}   # namespace defaults to the release namespace
        config: {ttl: 120}             # non-secret, provider specific
      - provider: route53
        zones: [example.org, internal.example.com]
        secretRef: {name: route53-dns}
    nameservers: []                    # resolvers for the propagation check
    propagationTimeout: 2m
```

- **Solver choice**: the longest zone that contains the host wins (`api.internal.example.com`
  goes to `route53` above, `www.example.com` to `cloudflare`). A solver with no `zones` takes
  every host no other solver claims. Hosts no solver covers use HTTP-01.
- **Override**: `lygard.io/acme-solver: <name>` on an Ingress (annotation) or a Gateway listener
  (`tls.options`) uses that solver for all of its hosts; `lygard.io/acme-solver: http01` forces
  HTTP-01.
- **Credentials** are read from the Secret at each order, so rotating them needs no restart.
- **Common config** (every provider): `ttl` (record TTL, seconds), `propagationTimeout`,
  `pollingInterval` (Go durations, override the global values for this solver), `endpoint`
  (API base URL, e.g. for a sovereign cloud or a proxy).

### Cloudflare

| Secret key | |
|---|---|
| `apiToken` | API token with **Zone → Zone → Read** and **Zone → DNS → Edit** on the zone(s) |

Config: `zoneID` (skips the zone lookup; then the token needs only DNS Edit).

```sh
kubectl -n lygard create secret generic cloudflare-dns --from-literal=apiToken=...
```

### AWS Route 53 (`route53`)

| Secret key | |
|---|---|
| `accessKeyID`, `secretAccessKey` | IAM user keys |
| `sessionToken` | optional, for temporary keys |

Without a `secretRef`, the controller uses `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`
(`controller.env`), or IRSA: annotate the service account with
`controller.serviceAccountAnnotations: {eks.amazonaws.com/role-arn: arn:aws:iam::…:role/…}` and
the injected `AWS_ROLE_ARN` + `AWS_WEB_IDENTITY_TOKEN_FILE` are exchanged at STS
(`AssumeRoleWithWebIdentity`). Config: `hostedZoneID` (skips the lookup), `roleARN`,
`stsEndpoint`.

Minimal IAM policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {"Effect": "Allow", "Action": ["route53:ChangeResourceRecordSets", "route53:ListResourceRecordSets"],
     "Resource": "arn:aws:route53:::hostedzone/Z0123456789ABC"},
    {"Effect": "Allow", "Action": "route53:ListHostedZonesByName", "Resource": "*"}
  ]
}
```

The hosted zone is found by name; private zones are skipped.

### Google Cloud DNS (`clouddns`)

| Secret key | |
|---|---|
| `serviceAccountJSON` | a service account key file (JSON) |

Without a `secretRef`, the token comes from the metadata server: GKE Workload Identity
(`controller.serviceAccountAnnotations: {iam.gke.io/gcp-service-account: …}`) or the node's
service account. Config: `project` (defaults to the key's `project_id`, or the metadata
server's project), `managedZone` (skips the lookup). Role: **DNS Administrator**
(`roles/dns.admin`) on the project, or a custom role with `dns.changes.create`,
`dns.changes.get`, `dns.managedZones.list`, `dns.resourceRecordSets.list`,
`dns.resourceRecordSets.create`, `dns.resourceRecordSets.update`,
`dns.resourceRecordSets.delete`. Private zones are skipped.

### Azure DNS (`azuredns`)

| Secret key | |
|---|---|
| `tenantID`, `clientID` | the service principal (may also be set in `config`) |
| `clientSecret` | its secret |

Config (required): `subscriptionID`, `resourceGroup`. Optional: `zone` (skips listing the
resource group's zones), `authorityHost` (sovereign clouds). Without a client secret, Azure
Workload Identity is used: `controller.podLabels: {azure.workload.identity/use: "true"}` and
`controller.serviceAccountAnnotations: {azure.workload.identity/client-id: …}`. Role: **DNS Zone
Contributor** on the zone (or resource group). Public zones only (`Microsoft.Network/dnsZones`).

### DigitalOcean (`digitalocean`)

| Secret key | |
|---|---|
| `token` | personal access token with write scope (or the `domain:*` scopes) |

Config: `zone` (skips listing the account's domains). Minimum TTL is 30.

### Linode / Akamai Cloud (`linode`)

| Secret key | |
|---|---|
| `token` | personal access token with **Domains: Read/Write** |

Config: `zone`. Linode publishes zone changes every few minutes, so set
`config: {propagationTimeout: 10m}` on this solver. TTL defaults to 300.

### RFC 2136 (`rfc2136`)

Dynamic updates to BIND, Knot, PowerDNS, Windows DNS and others.

| Secret key | |
|---|---|
| `tsigKeyName`, `tsigSecret` | TSIG key name and base64 secret (omit both for unsigned updates) |

Config: `nameserver` (required, `host[:port]`, the primary), `zone` (default: asked from the
nameserver with an SOA query), `tsigAlgorithm` (default `hmac-sha256`), `transport` (`udp`
or `tcp`). BIND example:

```
key "lygard" { algorithm hmac-sha256; secret "…"; };
zone "example.com" { type primary; file "example.com.db";
  update-policy { grant lygard name _acme-challenge.example.com. TXT; }; };
```

### Webhook (`webhook`)

The escape hatch for any other provider: the controller POSTs JSON to `config.url`

```json
{"fqdn": "_acme-challenge.example.com", "value": "…", "action": "present"}
```

with `action` `present` or `cleanup`, and expects a 2xx. Secret key `token` (optional) is sent as
`Authorization: Bearer <token>`. `present` must add the value (keep others at the same name:
`example.com` and `*.example.com` use the same name at once); `cleanup` removes only that value.

## Wildcard certificates

A `*.example.com` host can only be validated with DNS-01. List the apex too if you serve it
(`[example.com, "*.example.com"]`); both TXT values live at `_acme-challenge.example.com` at the
same time, and every provider above adds and removes them independently. A wildcard host that no
DNS-01 solver covers is not ordered: the Ingress or Gateway gets a `CertificateFailed` event
saying so, and the controller retries with backoff.

## CNAME delegation

You can keep DNS credentials away from your main zone: create a CNAME once, and give the
controller credentials only for a small validation zone.

```
_acme-challenge.example.com.  CNAME  _acme-challenge.example.com.acme.example.net.
```

The controller resolves `_acme-challenge.<domain>` before writing, follows CNAMEs (up to 10), and
writes the TXT record at the target. The solver's `zones` name the hosts' zone (it picks the
solver), while its credentials only need access to the target's zone, which the provider looks
up by itself: e.g. `zones: [example.com]` with a token scoped to `acme.example.net`.

## Propagation

Before asking the CA to validate, the controller finds the zone apex (SOA), asks every
authoritative nameserver of the zone directly (no recursion) until all of them serve the value,
polling every `pollingInterval` (5s) up to `propagationTimeout` (2m). If none of the nameservers
can be reached (egress to port 53 blocked), the recursive resolvers decide instead. After the
timeout it asks the CA anyway and logs a warning. `acme.dns01.nameservers` sets the recursive
resolvers (default: the pod's `/etc/resolv.conf`, then 1.1.1.1 and 8.8.8.8).

## Troubleshooting

- **Events** on the Ingress or Gateway: `CertificateIssuing`, `CertificateIssued`,
  `CertificateFailed` (with the CA's or the provider's error and the next attempt):
  `kubectl describe ingress web`.
- **Status**: `curl http://<controller>:9902/acme` lists every managed Secret with its state,
  hosts, expiry and last error; metrics `lygard_controller_acme_certificate_ready` and
  `lygard_controller_acme_certificate_expiry_timestamp_seconds`.
- **Logs**: `kubectl -n lygard logs deploy/lygard-controller | grep acme` shows the solvers at
  start-up (`DNS-01 solver … for zones …`), propagation warnings and clean-up failures.
- `… need a DNS-01 solver, and none in acme.dns01 covers this zone`: add the zone to a solver.
- `no DNS-01 solver named "…"`: `lygard.io/acme-solver` names a solver that is not configured.
- `secret …/… not found`: the solver's `secretRef` (default namespace: the release's).
- `TXT … not visible after 2m0s`: the provider accepted the change but the nameservers did not
  serve it in time; raise `propagationTimeout` (Linode, some RFC 2136 secondaries) and check
  the record with `dig +norecurse TXT _acme-challenge.example.com @<nameserver>`.
- HTTP 401/403 from a provider: the token lacks a permission listed above.
- Let's Encrypt rate limits apply (50 certificates per registered domain per week); test against
  the staging directory first.

## Standalone: automatic HTTPS

Outside Kubernetes (one YAML file, `lygard run -c`), the data plane gets certificates itself:

```yaml
global:
  acme:
    email: admin@example.com          # contact for the CA (expiry notices)
    hosts: [example.com, www.example.com]
    # directory: https://acme-staging-v02.api.letsencrypt.org/directory   # Let's Encrypt staging
    # stateDir: /var/lib/lygard/acme                                       # the default
listeners:
  - {name: http, address: "0.0.0.0:80"}
  - {name: https, address: "0.0.0.0:443", protocol: https}
```

- One certificate per host, through HTTP-01: the CA must reach each host on port 80, and Lygard
  answers `/.well-known/acme-challenge/` on every HTTP listener before routing (also on routes
  with `httpsRedirect`). Wildcards need DNS-01 and are refused here; use the controller.
- Certificates, private keys and the ACME account are written to `stateDir` (directory mode
  0700, files 0600) and served after a restart without a new order. Configured certificates win
  over issued ones for the same host.
- Renewal starts 30 days before expiry. A failed order is retried after 1 h, 2 h, 4 h ... up to a
  day, and the error is logged (`ACME certificate failed`).
- Until a host's certificate arrives, TLS for it uses a self-signed fallback.


## Client certificates to backends

When backends require mutual TLS, put the gateway's client certificate in a `kubernetes.io/tls`
Secret and reference it from the Gateway:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata: {name: gw, namespace: infra}
spec:
  gatewayClassName: lygard
  tls:
    backend:
      clientCertificateRef: {name: gateway-client}   # Secret in the Gateway's namespace
  listeners: [...]
```

Lygard presents it on every TLS connection to a backend (one with a BackendTLSPolicy, or
`appProtocol: https`). A Secret in another namespace needs a ReferenceGrant from Gateways to
Secrets. A route attached to several Gateways uses the certificate only if they all reference the
same Secret. Standalone, set `tlsClientCertPem` and `tlsClientKeyPem` on the backend.
