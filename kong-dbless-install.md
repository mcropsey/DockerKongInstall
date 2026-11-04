# Install Kong 3.9.4 in DBLESS Mode (Docker)

Step-by-step guide for installing Kong in **DBLESS mode** on an Ubuntu 24.04 host,
mirroring the routes from the existing gateway (192.168.1.100), with **no plugins**.

Verified working on **192.168.1.91** (Nov 2026).

---

## 0. What this gives you

- Kong **3.9.4** community, running as a Docker container
- **DBLESS** mode: no Postgres, config lives in one YAML file (`/opt/kong/kong.yml`)
- 9 services + 9 routes (notes, jenkins, k8s, dvga, juice-shop, vampi, juice-nomon, mailhog, crapi)
- **No plugins** installed/configured
- Host ports: **80** (proxy), **8000** (proxy), **8001** (admin API)

---

## 1. Prerequisites

This guide uses the Docker that's already installed on the host. Don't run
any Docker install script.

```bash
# Docker must already work without sudo
docker ps

# If that fails with "permission denied", add yourself to the docker group,
# then log out and back in:
sudo usermod -aG docker $USER

# Ports 80 / 8000 / 8001 must be free
sudo ss -tlnp | grep -E ':(80|8000|8001) '
```

---

## 2. Create the config directory and file

```bash
sudo mkdir -p /opt/kong
sudo touch /opt/kong/kong.yml
sudo chown $USER /opt/kong/kong.yml   # so you can edit it without sudo
```

Write `/opt/kong/kong.yml` (this is the exact config used on 192.168.1.91):

```yaml
_format_version: "3.0"
services:
  - name: notes
    host: 192.168.1.98
    port: 8000
    protocol: http
    connect_timeout: 60000
    read_timeout: 60000
    write_timeout: 60000
    retries: 5
    enabled: true
    routes:
      - name: notes-root
        hosts:
          - notes.cropseyit.com
        paths:
          - /
        strip_path: false
        protocols:
          - http
          - https
        preserve_host: true
        https_redirect_status_code: 426
  - name: jenkins
    host: 192.168.1.100
    port: 8080
    protocol: http
    connect_timeout: 60000
    read_timeout: 60000
    write_timeout: 60000
    retries: 5
    enabled: true
    routes:
      - name: jenkins-root
        hosts:
          - jenkins.cropseyit.com
        paths:
          - /
        strip_path: false
        protocols:
          - http
          - https
        preserve_host: true
        https_redirect_status_code: 426
  - name: k8s
    host: 192.168.1.75
    port: 80
    protocol: http
    connect_timeout: 60000
    read_timeout: 60000
    write_timeout: 60000
    retries: 5
    enabled: true
    routes:
      - name: k8s-root
        hosts:
          - k8s.cropseyit.com
        paths:
          - /
        strip_path: false
        protocols:
          - http
          - https
        preserve_host: true
        https_redirect_status_code: 426
  - name: dvga
    host: 192.168.1.101
    port: 5013
    protocol: http
    connect_timeout: 60000
    read_timeout: 60000
    write_timeout: 60000
    retries: 5
    enabled: true
    routes:
      - name: dvga-root
        hosts:
          - dvga.cropseyit.com
        paths:
          - /
        strip_path: false
        protocols:
          - http
          - https
        preserve_host: true
        https_redirect_status_code: 426
  - name: juice-shop
    host: 192.168.1.101
    port: 3000
    protocol: http
    connect_timeout: 60000
    read_timeout: 60000
    write_timeout: 60000
    retries: 5
    enabled: true
    routes:
      - name: juice-root
        hosts:
          - juice.cropseyit.com
        paths:
          - /
        strip_path: false
        protocols:
          - http
          - https
        preserve_host: true
        https_redirect_status_code: 426
  - name: vampi
    host: 192.168.1.99
    port: 30500
    protocol: http
    connect_timeout: 60000
    read_timeout: 60000
    write_timeout: 60000
    retries: 5
    enabled: true
    routes:
      - name: vampi-root
        hosts:
          - vampi.cropseyit.com
        paths:
          - /
        strip_path: false
        protocols:
          - http
          - https
        preserve_host: true
        https_redirect_status_code: 426
  - name: juice-nomon
    host: 192.168.1.99
    port: 30300
    protocol: http
    connect_timeout: 60000
    read_timeout: 60000
    write_timeout: 60000
    retries: 5
    enabled: true
    routes:
      - name: juice-nomon-root
        hosts:
          - juice-nomon.cropseyit.com
        paths:
          - /
        strip_path: false
        protocols:
          - http
          - https
        preserve_host: true
        https_redirect_status_code: 426
  - name: crapi-mail
    host: 192.168.1.101
    port: 8025
    protocol: http
    connect_timeout: 60000
    read_timeout: 60000
    write_timeout: 60000
    retries: 5
    enabled: true
    routes:
      - name: crapi-mailhog
        hosts:
          - mail.cropseyit.com
        paths:
          - /
        strip_path: false
        protocols:
          - http
          - https
        preserve_host: true
        https_redirect_status_code: 426
  - name: crapi-app
    host: 192.168.1.101
    port: 8888
    protocol: http
    connect_timeout: 60000
    read_timeout: 60000
    write_timeout: 60000
    retries: 5
    enabled: true
    routes:
      - name: crapi-root
        hosts:
          - crapi.cropseyit.com
        paths:
          - /
        strip_path: false
        protocols:
          - http
          - https
        preserve_host: true
        https_redirect_status_code: 426
```

> **Tip:** if you're copying config from an existing (DB-mode) Kong, pull it
> from its Admin API first:
>
> ```bash
> curl -s http://OLD_HOST:8001/services
> curl -s http://OLD_HOST:8001/routes
> curl -s http://OLD_HOST:8001/plugins   # skip these — "no plugins" install
> curl -s http://OLD_HOST:8001/upstreams
> ```

---

## 3. Pull the image

Use the **same version** as the gateway you're mirroring (check with
`curl -s http://OLD_HOST:8001/ | jq .version` — here it's 3.9.4):

```bash
docker pull kong:3.9.4
```

---

## 4. Start Kong in DBLESS mode

```bash
docker run -d \
  --name kong \
  --restart unless-stopped \
  -p 80:8000 \
  -p 8000:8000 \
  -p 8001:8001 \
  -v /opt/kong/kong.yml:/etc/kong/kong.yml:ro \
  -e KONG_DATABASE=off \
  -e KONG_DECLARATIVE_CONFIG=/etc/kong/kong.yml \
  -e KONG_ADMIN_LISTEN=0.0.0.0:8001 \
  -e KONG_PROXY_ACCESS_LOG=/dev/stdout \
  -e KONG_PROXY_ERROR_LOG=/dev/stderr \
  -e KONG_ADMIN_ACCESS_LOG=/dev/stdout \
  -e KONG_ADMIN_ERROR_LOG=/dev/stderr \
  -e KONG_LOG_LEVEL=notice \
  -e KONG_NGINX_PROXY_REAL_IP_HEADER=X-Real-IP \
  kong:3.9.4
```

### What each piece does

| Flag | Why |
|---|---|
| `-p 80:8000` | **Critical:** the old gateway (.100) serves traffic on port 80, so map 80→8000. Missing this was the cause of "not routing" during the first install. |
| `-p 8000:8000` | Direct proxy access (used by monitoring/tests) |
| `-p 8001:8001` | Admin API |
| `-v /opt/kong/kong.yml:/etc/kong/kong.yml:ro` | DBLESS declarative config |
| `KONG_DATABASE=off` | **DBLESS mode** — no Postgres |
| `KONG_DECLARATIVE_CONFIG=...` | Tells DBLESS where the config is |
| `KONG_ADMIN_LISTEN=0.0.0.0:8001` | **Critical gotcha:** since Kong 3.9 the admin API defaults to `127.0.0.1:8001` only, so it's unreachable from outside the container unless you set this |
| `KONG_NGINX_PROXY_REAL_IP_HEADER=X-Real-IP` | Matches .100 config (real client IP behind LB/Cloudflare) |
| `--restart unless-stopped` | Survives reboots |

---

## 5. Verify

```bash
# Health (wait until "healthy")
docker inspect kong --format '{{.State.Health.Status}}'

# Routes (expect 9, each with its host)
curl -s http://localhost:8001/routes \
  | python3 -c 'import json,sys; [print(r["name"], r["hosts"], r["paths"]) for r in json.load(sys.stdin)["data"]]'

# Services (expect 9)
curl -s http://localhost:8001/services \
  | python3 -c 'import json,sys; [print(s["name"], s["host"]+":"+str(s["port"])) for s in json.load(sys.stdin)["data"]]'

# Plugins (expect 0 — this is a no-plugins install)
curl -s http://localhost:8001/plugins | python3 -c 'import json,sys; print(len(json.load(sys.stdin)["data"]))'

# Live proxy test over port 80 (like a browser)
for h in notes jenkins k8s juice mail dvga vampi juice-nomon crapi; do
  curl -s -o /dev/null -m 10 -w "$h -> %{http_code}\n" \
    "http://$(hostname -I | awk '{print $1}')/" -H "Host: $h.cropseyit.com"
done
```

Expected results (Jenkins returns 403 because of its own auth page — that's a
*successful* proxy, not a Kong error):

```
notes -> 200
jenkins -> 403
k8s -> 200
juice -> 200
mail -> 200
dvga -> 200
vampi -> 200
juice-nomon -> 200
crapi -> 200
```

---

## 6. Day-2 operations

```bash
docker logs kong --tail 50          # access + error logs
docker restart kong                 # reload config after editing kong.yml
docker inspect kong --format '{{.State.Health.Status}}'
```

**To add/change a route:** edit `/opt/kong/kong.yml`, then `docker restart kong`.

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Host port 80 refuses connections, 8000 works | Forgot `-p 80:8000` (the actual bug hit on 192.168.1.91) |
| Admin API (8001) unreachable from outside, "connection reset" | Kong 3.9+ binds admin to `127.0.0.1` by default — set `KONG_ADMIN_LISTEN=0.0.0.0:8001` |
| `503 service_unavailable` | Route matched but upstream host:port wrong/unreachable — check the service entry and network reachability from the Kong host |
| `404` on a known domain | Route has no matching `hosts` entry, or config wasn't reloaded (`docker restart kong`) |
| All routes answer for any Host | `hosts` list missing on the routes |
| Config changes not taking effect | DBLESS loads config at startup — `docker restart kong` required |

---

## Appendix: how .100 (reference) is set up

- Kong 3.9.4 community, **DB mode** (Postgres 16 in a `kong-db` podman container)
- Custom image `localhost/kong-noname:3.9.4-5.0.1-src4` (bundles the `nonamesecurity` plugin)
- Plugins: `rate-limiting` on every service + one global `nonamesecurity`
  plugin (intentionally **not** replicated here)
- Proxy exposed on host port 80, admin on 8001
