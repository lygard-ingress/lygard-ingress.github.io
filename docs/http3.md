# HTTP/3 (QUIC)

An HTTPS listener can also serve HTTP/3 over QUIC on UDP, at the same address and port, with the
same certificates (SNI), client-certificate settings, routes and policies (auth, WAF, rate limits,
plugins, access log, tracing). Off by default.

## Enabling it

Helm:

```yaml
http3:
  enabled: true
  # advertisePort: 443   # default: service.httpsPort
```

This sets `http3: true` on the chart's HTTPS listener, opens UDP 8443 on the pod (and in the
NetworkPolicy when enabled), and adds a UDP port next to TCP 443 on the data plane Service. A
LoadBalancer Service with both protocols needs Kubernetes 1.26+ (mixed-protocol LoadBalancer
Services are GA there) and a load balancer that forwards UDP (table below).

Gateway API HTTPS listeners on other ports, and every HTTPS listener in per-Gateway address mode,
follow the chart setting: the controller sets `http3` on their data plane listeners and adds a UDP
port with the same number to the Service (port name `q-...` beside the TCP `g-...`/`gw-...`).
TLS passthrough listeners terminate no TLS and get no HTTP/3.

Standalone configuration:

```yaml
listeners:
  - name: https
    address: 0.0.0.0:443
    protocol: https
    http3: true
    # http3AdvertisePort: 443   # when clients connect on another port than the listener's
```

`http3` is only accepted on an HTTPS listener that terminates TLS.

## How clients find it

Browsers start on HTTP/1.1 or HTTP/2 over TCP. Every response on a listener with HTTP/3 carries

```
Alt-Svc: h3=":443"; ma=86400
```

and the browser tries QUIC for later requests to that origin, falling back to TCP whenever UDP is
blocked. The port is the listener's own unless `http3AdvertisePort` (chart: `http3.advertisePort`,
default the Service's HTTPS port) says otherwise; behind a load balancer mapping 443 to 8443 it
must be the public port, or clients try a port nobody listens on. An `Alt-Svc` set by a backend
is passed through unchanged. Responses over HTTP/3 carry none.

## Per-cloud support

| Cloud | Preset | HTTP/3 | Why |
|---|---|---|---|
| Azure AKS | `azure.yaml` | On | Standard LB forwards UDP, client IP preserved |
| Google GKE | `gcp.yaml` | On | Backend-service NLB (`l4-rbs`) carries TCP and UDP on one address |
| Oracle OKE | `oke.yaml` | On | NLB is pass-through for UDP |
| Bare metal (MetalLB) | `metallb.yaml` | On | TCP and UDP ports of one Service share the IP |
| AWS EKS | `aws.yaml`, `aws-automode.yaml` | Off | TCP and UDP 443 become one `TCP_UDP` target group, and PROXY v2 on it would also prefix QUIC datagrams. `TCP_UDP` target groups always preserve the client IP: with `proxyProtocol: false` and the proxy-protocol annotation removed, HTTP/3 can be turned on |
| Akamai / Linode LKE | `linode.yaml` | Off | Basic NodeBalancers have no UDP. On Premium the CCM sets PROXY protocol per port number, so TCP 443 (PROXY v2) and UDP 443 (none) cannot share the port; the controller leaves UDP ports for HTTP/3 off per-Gateway Services while Linode's PROXY default is on |
| DigitalOcean | `digitalocean.yaml` | Off | No UDP |
| Hetzner Cloud | `hetzner.yaml` | Off | No UDP |

## Client IP

PROXY protocol does not apply to QUIC: Lygard takes the client address from the UDP source.
With a pass-through load balancer (the presets where HTTP/3 is on) that is the real client. A
load balancer or kube-proxy hop that SNATs UDP (`externalTrafficPolicy: Cluster` on most clouds)
replaces it with a node or balancer address, so IP allow-lists, rate limits per IP and the access
log would see that address for HTTP/3 requests; keep `externalTrafficPolicy: Local` when HTTP/3
is on without PROXY protocol. `X-Forwarded-For` from `trustedProxies` is honoured as on TCP.

## Firewalls

Open UDP 443 (or the HTTPS port) inbound wherever TCP 443 is open: cloud security groups, node
firewalls, NetworkPolicies outside the chart. Nothing breaks when UDP is blocked: clients time out
on QUIC and keep using TCP, at the cost of a slower first request after each `Alt-Svc` refresh.

## Limits and timeouts

- `maxConnections` counts QUIC connections together with TCP ones; over the limit a QUIC
  connection is refused (`CONNECTION_REFUSED`), counted in
  `lygard_connections_rejected_total{reason="max_connections"}`.
- `maxHeaderBytes` bounds the request header section (QPACK `MAX_FIELD_SECTION_SIZE`);
  `headerReadTimeout` bounds the wait for request headers; `tlsHandshakeTimeout` the handshake.
- `idleTimeout` (capped at 10 minutes) is the QUIC idle timeout; `http2MaxConcurrentStreams` also
  limits concurrent HTTP/3 requests per connection.
- Request and response bodies are streamed. Response trailers (gRPC status) are sent as HTTP/3
  trailers; request trailers reach the backend when its protocol carries them (HTTP/2).

## Draining and reloads

On shutdown, and when a listener is removed or HTTP/3 is turned off, each QUIC connection gets a
GOAWAY: requests already received finish (up to `drainTimeout`), new ones go to another replica,
and new QUIC connections to this replica are refused so clients fall back at once. A listener
change keeps its UDP socket, as with TCP, so a reload never closes the port; turning HTTP/3 off
keeps the socket refusing connections until the listener goes away.

## Metrics

- `lygard_quic_connections_total{listener}`: QUIC connections established.
- `lygard_quic_active_connections{listener}`: open QUIC connections.
- `lygard_quic_handshake_errors_total{listener,reason}`: `tls`, `protocol`, `timeout`,
  `client_closed`, `other`.
- HTTP/3 handshakes also count in `lygard_tls_handshakes_total{alpn="h3"}` and
  `lygard_tls_handshake_duration_seconds`. Requests count in the usual request metrics; the access
  log's `protocol` field is `HTTP/3`.

## Not supported

- **0-RTT** (early data): off, because early requests can be replayed by anyone who captured
  them.
- **WebTransport**, extended CONNECT (WebSockets over HTTP/3), HTTP/3 datagrams and server push.
  WebSockets keep working over HTTP/1.1.
- **PROXY protocol** on UDP, and QUIC towards backends: backends are reached over HTTP/1.1 or
  HTTP/2 as configured.
- JA4 fingerprints for QUIC handshakes.
