# WAF rules: packs, categories, custom rules, IP reputation

Lygard's WAF has two layers that run in this order:

1. **Condition rules.** Each rule says "requests that look like this: block, allow, count,
   captcha or score", based on method, path, query, headers, cookies, client address, IP list or
   country.
   Your own rules, your rule groups and the built-in packs (WordPress, PHP, ...) all use this
   format.
2. **Attack rules.** These score fragments of the request for SQL injection, XSS, command
   injection and similar attacks, using the native engine or OWASP CRS. You can switch them off
   per category.

IP reputation lists (`ipDenyLists`) refuse an address before either layer runs.

Everything is set per route: in a [`LygardPolicy`](policy.md), with annotations, or in the
route `policy` of a standalone `lygard.yaml`. Shared definitions (rule groups and IP lists) live
in the data plane configuration, which in Kubernetes is the chart's ConfigMap.

```yaml
apiVersion: lygard.io/v1alpha1
kind: LygardPolicy
metadata: {name: blog-waf, namespace: web}
spec:
  targetRefs: [{kind: Ingress, name: blog}]
  policy:
    ipDenyLists: [spamhaus-drop]
    waf:
      mode: block
      rulesets: [wordpress, php, sensitive-files, bad-bots]
      disabledCategories: [java]          # nothing Java behind this route
      excludeRules: [120009]              # keep readme.html reachable
      rules:
        - id: 1001
          message: CI runners skip the WAF
          action: allow
          when: {ips: [198.51.100.0/24], headers: {x-ci-token: {present: true}}}
        - id: 1002
          message: wp-admin from the office only
          status: 404
          when: {path: {prefix: /wp-admin}}
          unless: {ipLists: [office]}
    securityHeaders:
      contentSecurityPolicy: "default-src 'self'"
      crossOriginOpenerPolicy: same-origin
      removeHeaders: [x-powered-by]
```

## Built-in packs

Switch packs on with `waf.rulesets` or `lygard.io/waf-rulesets: wordpress,php`. Their rules use
ids 120000 to 129999. To turn one rule off, put its id in `excludeRules`. The dashboard lists
every rule in each pack.

| Pack | Blocks |
|---|---|
| `wordpress` | `xmlrpc.php`, `wp-config.php` and copies of it, user enumeration (`?author=N`, `/wp-json/wp/v2/users`, `?rest_route=`), PHP files under `wp-content/uploads`, direct PHP in `wp-includes` (except `wp-tinymce.php` and `ms-files.php`), the installer, `debug.log`, `readme.html`/`license.txt` |
| `php` | PHP-CGI argument injection (CVE-2012-1823, CVE-2024-4577), `phpinfo`/`info.php`, `.phps` files and backup or editor copies of `.php` files, PHPUnit `eval-stdin.php`, Composer files, phpMyAdmin and Adminer, session and temp files |
| `sensitive-files` | `.env` and other secrets files, VCS metadata, database dumps, site backup archives, private keys and keystores, `server-status` and `.htaccess`/`web.config`, framework debug endpoints (`/actuator`, `_profiler`, `debug/pprof`, ...), `.DS_Store` |
| `bad-bots` | A missing or empty `User-Agent`; scanners and spam bots by user agent (zgrab, masscan, nuclei, ...). API clients such as `curl` or `python-requests` are not blocked |
| `header-hardening` | `X-Original-URL`/`X-Rewrite-URL` (access-control bypass), `X-HTTP-Method-Override: TRACE/CONNECT`, oversized `User-Agent`/`Referer`, a spoofed `X-Forwarded-Host` |
| `admin-paths` | Admin paths of software you do not run: Joomla, Drupal installers, cPanel/Webmin/Plesk, Tomcat manager, JMX console, common web-shell names |

The packs block, so try them in `mode: detect` first. They also run when the attack rules are
skipped for a static file (`skipExtensions`).

## Attack categories

`disabledCategories` (or `lygard.io/waf-disabled-categories`) switches whole categories off,
in either engine: `sqli`, `xss`, `lfi`, `rfi`, `rce`, `php`, `java`, `nodejs`, `ssrf`, `ssti`,
`xxe`, `prototype-pollution`, `session`, `scanner`, `protocol`, `leakage`. With CRS, a category
covers its rule-id range: for example, `sqli` is 942000 to 942999 and `xss` is 941000 to
941999. `ssrf`, `nodejs` and `prototype-pollution` share CRS's generic-attack group (934),
`ssti` and `xxe` exist in the native engine only, and `leakage` applies to response inspection.

Switching a category off removes protection. To fix a single false positive, prefer an
exclusion on one field (`waf.exclusions`, or the dashboard's tuning panel).

## Custom rules

```yaml
- id: 2001                  # unique; shown in WAF events; usable in excludeRules
  message: no uploads from outside the EU
  action: block             # block (default) | allow | count | captcha | score
  status: 403               # for block (default 403)
  score: 5                  # for score: added to the anomaly score
  when:                     # every condition must hold
    methods: [POST, PUT]
    path: {prefix: /api/upload}
    queryString: {longerThan: 4096}
    headers: {content-type: {contains: multipart}}
    args: {debug: {present: true}}
    cookies: {role: {oneOf: [admin, root]}}
    ips: [198.51.100.0/24]
    ipLists: [office]
    countries: [CN, RU]
  unless:                   # same conditions; when they hold, the rule does not apply
    countries: [DE, FR, NL]
```

- **Matchers.** Each value takes exactly one of `exact`, `prefix`, `suffix`, `contains`,
  `regex`, `oneOf`, `present` or `longerThan`, plus optional `not: true` and
  `caseSensitive: true`. Comparisons ignore case by default.
- **Missing values.** A header, argument or cookie that is not in the request only matches
  `present: false`.
- **Lists.** Within `ips`/`ipLists` (and within `countries`, `methods` and `oneOf`), any one
  entry is enough.
- **Path.** `path` is the decoded and normalised path that routing also uses, so `/%61dmin`,
  `//admin` and `/x/../admin` all count as `/admin`.
- **Order.** Rules run in order: the route's `rules` first, then each entry of `rulesets`. The
  first `block` or `allow` decides. `allow` skips the rest of the WAF, attack rules included.
  `log` and `score` let evaluation continue.
- **Detect mode.** In `mode: detect`, a `block` rule is recorded as flagged and the request goes
  through.
- **Validation.** Ids must not collide with the attack rules or with another rule on the route.
  A rule the data plane cannot compile (a bad regex, an unknown list or pack) rejects the whole
  configuration and reports where the error is, and the previous configuration keeps serving.

## Actions

| Action | What happens | Metric `action` label |
|---|---|---|
| `block` | Refused with `status` (default 403) and Lygard's error page. In `mode: detect` it is only recorded | `block` |
| `allow` | The request skips the rest of the WAF: later rules, packs and attack rules | `allow` |
| `count` | Recorded (WAF events, metrics, inspector); evaluation continues. `log` is accepted as an alias | `count` |
| `captcha` | Browsers without a valid pass get a challenge page; once solved, the rule counts for `passTtl`. Clients that cannot run a challenge (API clients, scripts) get 403 | `captcha`, `captcha_passed` |
| `score` | Adds `score` to the request's anomaly score; the attack rules' threshold decides | `score` |

Every match increments `lygard_waf_rule_actions_total{route,rule,action}`. A blocked or
challenged request shows the rule in the request inspector under its request ID.

### Challenges (`captcha`)

The default challenge is a proof of work that runs in the visitor's browser:

- **No third party:** nothing is sent outside your cluster, and nothing tracks the visitor.
- **Cost to visitors:** at the default `difficulty: 18`, about half a second on a phone, once
  per `passTtl`.
- **Who fails it:** a client that does not run JavaScript.

A solved challenge sets an HttpOnly cookie, `__lygard_pass`. It is signed with HMAC-SHA256 and
bound to the host and to the client's /24 (IPv4) or /64 (IPv6). The signing key is derived from
the admin token Secret that the chart creates at install, or from `LYGARD_CHALLENGE_SECRET`. So
every replica accepts every other replica's passes, and nothing is written to the cluster at
run time.

```yaml
global:
  challenge:
    provider: pow          # pow | turnstile | hcaptcha | recaptcha
    difficulty: 18         # pow only: each +1 doubles the work
    passTtl: 30m
    siteKey: ""            # turnstile / hcaptcha / recaptcha site key; the secret comes from
                           # the LYGARD_CAPTCHA_SECRET environment variable
```

Challenge `/checkout` and `/login` for new visitors while letting monitoring through:

```yaml
- id: 3001
  action: captcha
  message: challenge checkout and login
  when: {path: {regex: "^/(checkout|login)"}, methods: [GET]}
  unless: {headers: {user-agent: {contains: uptime-monitor}}}
```

Only challenge `GET` pages that browsers load. A `POST` from a form or an API call cannot show a
challenge page.

## Rule groups and IP lists (cluster-wide, via the ConfigMap)

The platform team defines reusable groups and IP lists once in the data plane configuration. In
Kubernetes, that is `config.waf` in the Helm values, rendered into the chart's ConfigMap. The
controller re-reads that ConfigMap on every sync, so `helm upgrade` (or editing the ConfigMap)
changes them without a restart. Routes then switch them on by name.

```yaml
# values.yaml
config:
  waf:
    ruleGroups:
      office-only-admin:
        - {id: 5001, status: 404, when: {path: {prefix: /admin}}, unless: {ipLists: [office]}}
    ipLists:
      office: {cidrs: [203.0.113.0/24, 2001:db8:10::/48]}
      blocked-by-security: {files: [/etc/lygard/ip-lists/blocked.txt], refresh: 1m}
      partner-feed: {urls: [https://feeds.example.com/bad-ips.txt], refresh: 30m}
extraVolumes:
  - {name: ip-lists, configMap: {name: lygard-ip-lists}}
extraVolumeMounts:
  - {name: ip-lists, mountPath: /etc/lygard/ip-lists, readOnly: true}
```

Teams use them by name: `rulesets: [office-only-admin]`, `ipDenyLists: [blocked-by-security]`,
or `ipLists: [office]` inside a rule. Per-application rules belong in a `LygardPolicy` in the
application's namespace. One policy can target several routes.

### IP lists

- **Presets.** These work without an entry in `ipLists`:

  | Preset | Source |
  |---|---|
  | `spamhaus-drop` | Spamhaus DROP, IPv4 and IPv6: netblocks run by spammers and cyber-criminals |
  | `firehol-level1` | FireHOL level 1 |
  | `tor-exits` | Tor exit relays |

  To change a preset's refresh interval or sources, define a list under its name.
- **Formats.** One address or CIDR per line, with `#` or `;` comments. Spamhaus JSON lines and
  FireHOL `.netset` files are read as they are.
- **Private ranges.** Private, loopback, link-local and CGNAT ranges are removed from every list
  (`excludePrivate: true`, the default), so a feed that includes bogons cannot block your
  cluster network, health checks or load balancers.
- **Refresh.** URLs and files are read again every `refresh` (default 1 hour). A failed fetch
  keeps the last good copy.
- **Lookup cost.** A lookup is one binary search over merged ranges, so list size does not
  change the cost per request.
- **Network access.** URL lists need outbound HTTPS from the data plane pods. Until the first
  fetch completes, a URL list is empty.
- **Monitoring.** Metrics: `lygard_ip_list_ranges{list}` and
  `lygard_ip_list_refresh_total{list,result}`. The dashboard shows each list's size, last
  refresh and last error.

## Security headers

`securityHeaders` adds the following response headers, unless the backend already set them:

- **Defaults:** HSTS (over HTTPS only), `X-Frame-Options`, `X-Content-Type-Options` and
  `Referrer-Policy`.
- **Optional:** `Content-Security-Policy`, `Permissions-Policy` and the cross-origin headers
  (`crossOriginOpenerPolicy`, `crossOriginResourcePolicy`, `crossOriginEmbedderPolicy`).

`removeHeaders` strips headers that reveal software versions, such as `x-powered-by` and
`x-aspnet-version`. The global `serverHeader` setting controls `Server`.

## From the dashboard

The **WAF rules** section of the dashboard edits the policy of one route at a time:

- tick packs, categories and IP lists;
- turn off single rules inside a pack;
- apply a security-header preset (basic or strict);
- add, reorder and remove custom rules.

The dashboard turns the edits into one of three outputs:

- a `LygardPolicy` (Kubernetes, GitOps);
- `kubectl annotate` commands (the subset that annotations support);
- the route `policy` block for a standalone `lygard.yaml`.

You apply the output with `kubectl apply` or by committing it, so changes go through review and
your Git repository stays the source of truth. The dashboard never writes to the cluster. The
same section shows the status of each IP list.

**Test a request** runs a request you describe through the route's live rules: IP lists,
condition rules, then the attack rules. It shows the decision and every rule that matched. The
API is `POST /api/waf/test`, which takes an admin token:

```sh
curl -H "Authorization: Bearer $TOKEN" localhost:9901/api/waf/test \
  -d '{"route":"blog","method":"GET","url":"/wp-config.php.bak","ip":"203.0.113.9"}'
```

The other endpoints are `GET /api/waf/rulesets` (packs, groups, categories and what each route
uses) and `GET /api/ip-lists`.
