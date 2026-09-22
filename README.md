# Custom Naxsi & Crowdsec Configs

If you don't want crowdsec go check: https://github.com/stylersnico/Custom-Naxsi-Configs

My own Naxsi WAF & Crowdsec configurations for my self-hosted infrastructure.

The goal of this repository is to provide a hardened reverse-proxy in front of every self-hosted app I run, with support for:

* Naxsi WAF (core rules + per-app hand-written/community whitelists)
* Per-vhost connection & request rate limiting
* Per-vhost error/WAF logging, plus a 1h self-cleaning access log feeding CrowdSec (see below)
* [CrowdSec](https://crowdsec.net) behavioral IP banning on top of Naxsi's per-request filtering, enforced at the firewall (PF)
* A shared, locally-served Naxsi "request denied" page
* Global security headers (CSP, HSTS, X-Frame-Options, Permissions-Policy, ...) that always win over whatever the backend sends, via `proxy_hide_header`

Runs on **nginx-full** (FreeBSD 15) with the `naxsi`, `headers_more` and `lua` dynamic modules.

We also use Crowdsec bouncer if association with PF (and not Nginx LUA).

--------

## Structure

```
Reverse-NGINX/
├── nginx.conf              # main config: modules, TLS, global headers, rate/conn zones
├── sites-enabled/          # one vhost per app
├── naxsi/                  # naxsi_core.rules + per-app whitelists
├── newsyslog.conf.d/       # hourly, zero-retention rotation for the access logs below
└── errors/                 # shared naxsi_denied.html
CrowdSec/
├── acquis.yaml              # detection engine: reads every vhost's access.log
└── pf-bouncer.conf          # PF tables/rules CrowdSec's firewall bouncer maintains
```

--------

## Protected applications

Each app got naxsi in one of two modes:

* **Full scan** – naxsi runs on the whole app, whitelist entries added only after a confirmed false positive in `naxsi_error.log`.
* **Login only** – naxsi runs solely on the authentication endpoint(s); everything past a valid session is bypassed entirely. Used once an app's authenticated surface kept passing structured/serialized data through nearly every field, making field-by-field whitelisting not worth chasing (CheckMK is the case that established the pattern: 5 fixes across 4 endpoints before flipping).

| App | Mode | Notes |
|---|---|---|
| **WordPress** | Full scan | Upstream `wordpress.rules`/`wordpress-block.rules` + local extras (select2/imgareaselect assets, plugin meta-box brackets). `/wp-admin/load-{styles,scripts}.php` bypassed (unenumerable plugin handle list). |
| **Nextcloud** | Full scan | WebDAV (`/remote.php/dav`, `/public.php/webdav`) bypassed - raw XML bodies. Cookie double-encoding + OCS unknown-content-type fixed. |
| **Umami** | Full scan | CORS added on `/api/send` for cross-origin tracking. `title`/`url` fields whitelisted (arbitrary visitor-submitted data). |
| **Gitea** | Full scan | Git smart-HTTP and `/api/` bypassed. Fixed the `/compare/v1...v2` traversal false positive (git's own `...` syntax), login `redirect_to`. |
| **Wiki.js** | Hybrid | `/graphql` (the whole app API - login, page edits) bypassed entirely after 3 fields needed full whitelisting in a row. Static assets still scanned. |
| **Static site** | Full scan | GET/HEAD-only. |
| **CheckMK** | Login only | `login.py` + `user_login_two_factor.py`. |
| **Grafana** | Login only | `/login`. Embeddable in Home Assistant via a scoped CSP `frame-ancestors`. |
| **Home Assistant** | Login only | `/auth/`. `/api/websocket` untouched. |
| **gitea-mirror** | Login only | `/login` + `/api/auth/` (Better Auth). |
| **Passbolt** | Login only | `/auth/login.json` + `/auth/verify.json` (GPG challenge-response). |
| **Jellyfin** | Login only | `POST /Users/AuthenticateByName`. Media streaming untouched. |

No community-maintained naxsi ruleset exists for any of these apps except WordPress (`nbs-system/naxsi-rules`) - every other whitelist is hand-written from real traffic.

--------

## CrowdSec (behavioral IP banning)

Naxsi filters individual requests; it has no memory across requests. CrowdSec closes that gap: it reads traffic patterns (repeated 403s/404s, scanning, probing, community-shared malicious IPs) and bans the *source IP* at the firewall, before it can keep hammering any vhost.

```
client → PF (drops IPs CrowdSec banned) → nginx + Naxsi (per-request filtering) → backends
                     ▲
                     │ reads decisions from
              CrowdSec detection engine ← acquis.yaml ← each vhost's 1h access.log
```

* **Access logs are new and deliberately short-lived.** This repo previously ran with `access_log off` everywhere - CrowdSec's stock `crowdsecurity/nginx` parser needs them (its scenarios key off status codes/paths/user-agents that only appear in access logs, not `naxsi_error.log`). Each vhost now logs in the standard `combined` format to its own `access.log`, rotated hourly with zero archives kept (`Reverse-NGINX/newsyslog.conf.d/nginx-access.conf`) - so at most ~1h of traffic ever exists on disk, purely as CrowdSec's input, not as a retained record.
* **The bouncer (the part that actually blocks) is the PF firewall bouncer, not the nginx Lua bouncer**, even though this build has `ngx_http_lua_module` available. CrowdSec's official nginx bouncer (`cs-nginx-bouncer`) is only tested upstream on Debian/Ubuntu and installed via an apt-based script - there's no verified FreeBSD porting path for it, and guessing at its internal Lua API on a config that fronts all 12 apps wasn't worth the risk. `crowdsec-firewall-bouncer` is an official FreeBSD package (`pkg install crowdsec-firewall-bouncer`) with a documented `mode: pf`, so that's what's wired up - it bans at the network layer, which also protects everything else on this box, not just HTTP.

### Setup

```bash
pkg install crowdsec crowdsec-firewall-bouncer
```

1. Copy `CrowdSec/acquis.yaml` to `/usr/local/etc/crowdsec/acquis.yaml`.
2. Append `CrowdSec/pf-bouncer.conf` to `/etc/pf.conf` (above your existing `pass` rules), then `service pf reload`.
3. Install the detection collections:
   ```bash
   cscli collections install crowdsecurity/nginx
   cscli collections install crowdsecurity/http-cve
   cscli collections install crowdsecurity/base-http-scenarios
   ```
4. Register the bouncer and wire its key into the FreeBSD-generated `/usr/local/etc/crowdsec/bouncers/crowdsec-firewall-bouncer.yaml` (`mode: pf`) - **this file holds a live API key, it is not and should not be committed to this repo**:
   ```bash
   cscli bouncers add firewall-bouncer
   ```
5. `sysrc crowdsec_enable=YES crowdsec_firewall_enable=YES && service crowdsec start && service crowdsec_firewall start`

Verify with `cscli decisions list`, `cscli metrics`, and `pfctl -t crowdsec-blacklists -T show`.

--------

## Deployment

> :warning: **Naxsi log directories aren't created automatically.** Before reloading nginx with a new vhost, create its log folder or nginx will refuse to start:
```bash
mkdir -p /var/log/nginx/<vhost>
chown www:www /var/log/nginx/<vhost>
```

Then always test before reloading:
```bash
nginx -t && service nginx reload
```

--------

## Known limitations

* Nextcloud's own admin security panel may still warn about HSTS - that check hits a separate local web server on the Nextcloud host itself, outside this repo.
* Naxsi has no community-validated ruleset for anything but WordPress; every other app's whitelist is scoped to what's actually been seen in production traffic, not a security guarantee against everything.
* `naxsi_error.log` isn't ingested by CrowdSec - no parser exists yet for its `NAXSI_FMT` format, so CrowdSec's behavioral bans are based on raw traffic patterns (access logs), not on Naxsi's own block decisions.
* No nginx-level (Lua) CrowdSec bouncer - only the PF firewall bouncer is wired up; see the CrowdSec section above for why.
