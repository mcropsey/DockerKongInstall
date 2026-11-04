# Install Kong 3.9.4 in DB Mode (Docker + Postgres)

Step-by-step guide for installing Kong in **DB mode** (backed by PostgreSQL) on an
Ubuntu 24.04 host, mirroring the routes from the existing gateway (192.168.1.100),
with **no plugins**.

This is the DB-mode companion to [`kong-dbless-install.md`](kong-dbless-install.md).
Same Kong version, same 9 services/routes, same ports — the difference is that
config lives in Postgres and can be changed live through the Admin API (or decK)
without restarting Kong.

---

## 0. What this gives you

- Kong **3.9.4** community, running as a Docker container
- **PostgreSQL 16** in a second container (`kong-db`), data on a named volume
- **DB mode**: config stored in Postgres; changes via Admin API take effect immediately
- 9 services + 9 routes (notes, jenkins, k8s, dvga, juice-shop, vampi, juice-nomon, mailhog, crapi)
- **No plugins** installed/configured
- Host ports: **80** (proxy), **8000** (proxy), **8001** (admin API)
- Postgres is **not** published on the host — only reachable on the private Docker network

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

# If a DBLESS Kong from the other guide is running on this host, remove it first
docker rm -f kong 2>/dev/null || true
```

---

## 2. Create the network, volume and DB password

```bash
docker network create kongnet
docker volume create kong-db-data

# Generate a DB password and keep it out of shell history / the repo
openssl rand -hex 24 > ~/.kong-db-password
chmod 600 ~/.kong-db-password
```

---

## 3. Pull the images

Use the **same version** as the gateway you're mirroring (check with
`curl -s http://OLD_HOST:8001/ | jq .version` — here it's 3.9.4):

```bash
docker pull kong:3.9.4
docker pull postgres:16
```

---

## 4. Start PostgreSQL

```bash
docker run -d \
  --name kong-db \
  --restart unless-stopped \
  --network kongnet \
  -v kong-db-data:/var/lib/postgresql/data \
  -e POSTGRES_USER=kong \
  -e POSTGRES_DB=kong \
  -e POSTGRES_PASSWORD="$(cat ~/.kong-db-password)" \
  --health-cmd 'pg_isready -U kong' \
  --health-interval 5s \
  --health-retries 10 \
  postgres:16

# Wait until it reports "healthy"
until [ "$(docker inspect kong-db --format '{{.State.Health.Status}}')" = healthy ]; do sleep 2; done
echo "postgres is up"
```

---

## 5. Bootstrap the Kong database (one time)

Runs Kong's migrations against the empty database, then exits:

```bash
docker run --rm \
  --network kongnet \
  -e KONG_DATABASE=postgres \
  -e KONG_PG_HOST=kong-db \
  -e KONG_PG_USER=kong \
  -e KONG_PG_PASSWORD="$(cat ~/.kong-db-password)" \
  kong:3.9.4 kong migrations bootstrap
```

Expect it to finish with `Database is up-to-date`. (If you ever upgrade Kong
later, run `kong migrations up` + `kong migrations finish` with the new image
instead of `bootstrap`.)

---

## 6. Start Kong in DB mode

```bash
docker run -d \
  --name kong \
  --restart unless-stopped \
  --network kongnet \
  -p 80:8000 \
  -p 8000:8000 \
  -p 8001:8001 \
  -e KONG_DATABASE=postgres \
  -e KONG_PG_HOST=kong-db \
  -e KONG_PG_USER=kong \
  -e KONG_PG_PASSWORD="$(cat ~/.kong-db-password)" \
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
| `--network kongnet` | Lets Kong reach Postgres by name (`kong-db`) without exposing 5432 on the host |
| `-p 80:8000` | **Critical:** the old gateway (.100) serves traffic on port 80, so map 80→8000. Missing this was the cause of "not routing" during the first DBLESS install. |
| `-p 8000:8000` | Direct proxy access (used by monitoring/tests) |
| `-p 8001:8001` | Admin API |
| `KONG_DATABASE=postgres` | **DB mode** — config is stored in Postgres |
| `KONG_PG_HOST / USER / PASSWORD` | Postgres connection (database name defaults to `kong`) |
| `KONG_ADMIN_LISTEN=0.0.0.0:8001` | **Critical gotcha:** since Kong 3.9 the admin API defaults to `127.0.0.1:8001` only, so it's unreachable from outside the container unless you set this |
| `KONG_NGINX_PROXY_REAL_IP_HEADER=X-Real-IP` | Matches .100 config (real client IP behind LB/Cloudflare) |
| `--restart unless-stopped` | Survives reboots (both containers) |

> **Security note:** in DB mode the Admin API can **change** the live config,
> not just read it. Only expose 8001 on a trusted LAN (or bind it to
> `127.0.0.1:8001` and use SSH tunnelling).

---

## 7. Load the services and routes

In DB mode the config goes into Postgres. Write the same declarative file used
by the DBLESS guide, then import it **once** into the database.

```bash
sudo mkdir -p /opt/kong
sudo touch /opt/kong/kong.yml
sudo chown $USER /opt/kong/kong.yml   # so you can edit it without sudo
```

Write `/opt/kong/kong.yml`:

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

Import it into the database (uses a throwaway Kong container on the same network):

```bash
docker run --rm \
  --network kongnet \
  -v /opt/kong/kong.yml:/tmp/kong.yml:ro \
  -e KONG_DATABASE=postgres \
  -e KONG_PG_HOST=kong-db \
  -e KONG_PG_USER=kong \
  -e KONG_PG_PASSWORD="$(cat ~/.kong-db-password)" \
  kong:3.9.4 kong config db_import /tmp/kong.yml
```

No restart is needed — the running Kong picks up the new entities from the
database automatically (within a few seconds).

> **Note:** `db_import` is meant for loading an **empty** database. Running it
> again on top of existing entities will fail on duplicate names. For ongoing
> changes use the Admin API or decK (section 9).

> **Tip:** if you're copying config from an existing (DB-mode) Kong, pull it
> from its Admin API first:
>
> ```bash
> curl -s http://OLD_HOST:8001/services
> curl -s http://OLD_HOST:8001/routes
> curl -s http://OLD_HOST:8001/plugins   # skip these — "no plugins" install
> curl -s http://OLD_HOST:8001/upstreams
> ```
>
> or, with decK installed: `deck gateway dump --kong-addr http://OLD_HOST:8001 -o old.yaml`

---

## 8. Verify

```bash
# Health (wait until "healthy") — both containers
docker inspect kong-db --format '{{.State.Health.Status}}'
docker inspect kong    --format '{{.State.Health.Status}}'

# Confirm DB mode (expect "postgres")
curl -s http://localhost:8001/ \
  | python3 -c 'import json,sys; print(json.load(sys.stdin)["configuration"]["database"])'

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

## 9. Day-2 operations

```bash
docker logs kong --tail 50          # access + error logs
docker logs kong-db --tail 50       # postgres logs
docker restart kong                 # rarely needed in DB mode
docker inspect kong --format '{{.State.Health.Status}}'
```

**To add/change a route** — no restart needed, changes are live immediately.

Option A, Admin API:

```bash
# add a service + route
curl -s -X POST http://localhost:8001/services \
  -d name=example -d url=http://192.168.1.50:8080
curl -s -X POST http://localhost:8001/services/example/routes \
  -d name=example-root -d 'hosts[]=example.cropseyit.com' -d 'paths[]=/' \
  -d strip_path=false -d preserve_host=true

# change an upstream port
curl -s -X PATCH http://localhost:8001/services/example -d port=8081

# delete
curl -s -X DELETE http://localhost:8001/routes/example-root
curl -s -X DELETE http://localhost:8001/services/example
```

Option B, decK (keeps a YAML file as source of truth):

```bash
deck gateway dump --kong-addr http://localhost:8001 -o kong.yaml   # export current state
# ...edit kong.yaml...
deck gateway diff --kong-addr http://localhost:8001 kong.yaml      # preview
deck gateway sync --kong-addr http://localhost:8001 kong.yaml      # apply (FULL sync — deletes anything not in the file)
```

Pick one approach per gateway: if you manage it with decK, don't make ad-hoc
`curl` changes, or the next `sync` will delete them.

### Backup / restore the database

```bash
# backup
docker exec kong-db pg_dump -U kong kong | gzip > ~/kong-db-$(date +%F).sql.gz

# restore (into an empty DB)
gunzip -c ~/kong-db-YYYY-MM-DD.sql.gz | docker exec -i kong-db psql -U kong kong
```

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Host port 80 refuses connections, 8000 works | Forgot `-p 80:8000` |
| Admin API (8001) unreachable from outside, "connection reset" | Kong 3.9+ binds admin to `127.0.0.1` by default — set `KONG_ADMIN_LISTEN=0.0.0.0:8001` |
| Kong container keeps restarting, logs say `Database needs bootstrapping` | Skipped step 5 — run `kong migrations bootstrap` |
| `connection refused` / `could not resolve kong-db` in Kong logs | Kong not on `kongnet`, or `kong-db` not running/healthy yet |
| `password authentication failed for user "kong"` | Password mismatch. Postgres only reads `POSTGRES_PASSWORD` on the **first** start of an empty volume — if you regenerated `~/.kong-db-password` later, either reset it in Postgres or recreate the volume |
| `db_import` fails with `UNIQUE violation` | Entities already exist — use the Admin API / decK for changes, not a second import |
| `503 service_unavailable` | Route matched but upstream host:port wrong/unreachable — check the service entry and network reachability from the Kong host |
| `404` on a known domain | Route has no matching `hosts` entry |
| All routes answer for any Host | `hosts` list missing on the routes |

### Start over (destroys all Kong config)

```bash
docker rm -f kong kong-db
docker volume rm kong-db-data
docker network rm kongnet
```

---

## DB mode vs DBLESS at a glance

| | DBLESS (`kong-dbless-install.md`) | DB mode (this guide) |
|---|---|---|
| Containers | `kong` | `kong` + `kong-db` (Postgres 16) |
| Config source | `/opt/kong/kong.yml` mounted read-only | Postgres (`kong-db-data` volume) |
| Change config | edit YAML + `docker restart kong` | Admin API / decK, live, no restart |
| Admin API writes | read-only | full read/write |
| Extra steps | none | network, DB password, `migrations bootstrap`, one-time `db_import` |
| Backups | copy the YAML | `pg_dump` (or `deck gateway dump`) |

---

## Appendix: how .100 (reference) is set up

- Kong 3.9.4 community, **DB mode** (Postgres 16 in a `kong-db` podman container)
- Custom image `localhost/kong-noname:3.9.4-5.0.1-src4` (bundles the `nonamesecurity` plugin)
- Plugins: `rate-limiting` on every service + one global `nonamesecurity`
  plugin (intentionally **not** replicated here)
- Proxy exposed on host port 80, admin on 8001
