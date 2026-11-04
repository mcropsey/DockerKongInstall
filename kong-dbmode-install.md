# Install Kong 3.9.4 in DB Mode (Docker + Postgres)

Step-by-step guide for installing Kong in **DB mode** (backed by PostgreSQL) on an
Ubuntu 24.04 host, mirroring the routes from the existing gateway (192.168.1.100),
with **no plugins**.

Verified working on **192.168.1.91** (ubuntu-1, Nov 2026). Every command below was
run as written.

This is the DB-mode companion to [`kong-dbless-install.md`](kong-dbless-install.md).
Same Kong version, same 9 services/routes, same ports, and **the same
`/opt/kong/kong.yml` file is still the source of truth**. The difference is that
Kong stores its config in Postgres, and [decK](https://docs.konghq.com/deck/)
syncs `kong.yml` into the database. You edit the YAML and run one command. You
don't use curl and you don't restart Kong.

---

## 0. What this gives you

- Kong **3.9.4** community, running as a Docker container
- **PostgreSQL 16** in a second container (`kong-db`), data on a named volume
- **DB mode**: Kong reads its config from Postgres
- **`/opt/kong/kong.yml` is the source of truth**: services, routes, plugins, and
  consumers are all defined there and pushed with decK (`kong-apply.sh apply`).
  Changes reach the proxy within about 5 seconds, with no Kong restart.
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
> not just read it. Only expose 8001 on a trusted LAN. decK runs on this host
> and talks to `localhost:8001`, so you can also publish it as
> `-p 127.0.0.1:8001:8001` to keep it off the LAN entirely.

---

## 7. Install decK

decK is a single binary. It reads `kong.yml`, compares it with what's in Kong,
and creates, updates, or deletes entities through the Admin API so that Kong
matches the file exactly.

```bash
DECK_VERSION=1.69.0
curl -sL "https://github.com/Kong/deck/releases/download/v${DECK_VERSION}/deck_${DECK_VERSION}_linux_amd64.tar.gz" \
  | sudo tar -xz -C /usr/local/bin deck
deck version
```

---

## 8. Write `kong.yml` (the source of truth)

This is the same declarative file the DBLESS guide uses. In DB mode it is not
mounted into the container. decK pushes it into the database instead.

```bash
sudo mkdir -p /opt/kong
sudo chown -R $USER:$USER /opt/kong  # you own the dir: edit kong.yml + create kong-apply.sh without sudo
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

---

## 9. Create the apply script and push the config

`/opt/kong/kong-apply.sh` wraps decK so every change follows the same steps:
validate, back up the current state, then sync. Run `diff` first to preview.

```bash
cat > /opt/kong/kong-apply.sh <<'EOF'
#!/usr/bin/env bash
# Make Kong (DB mode) match /opt/kong/kong.yml exactly.
#   kong-apply.sh validate   check the YAML against the running Kong
#   kong-apply.sh diff       show what would change (no changes made)
#   kong-apply.sh apply      validate + backup + sync (prints every change)
#   kong-apply.sh backup     dump current Kong state to ~/kong-backups/
#   kong-apply.sh restore F  sync a backup file back into Kong
set -euo pipefail
KONG_YML=${KONG_YML:-/opt/kong/kong.yml}
KONG_ADDR=${KONG_ADDR:-http://localhost:8001}
BACKUP_DIR=${BACKUP_DIR:-$HOME/kong-backups}
D="--kong-addr $KONG_ADDR"

backup() {
  mkdir -p "$BACKUP_DIR"
  f="$BACKUP_DIR/kong-$(date +%F-%H%M%S).yaml"
  deck gateway dump $D -o "$f" --yes >/dev/null
  echo "backup: $f"
}

case "${1:-}" in
  validate) deck gateway validate $D "$KONG_YML" ;;
  diff)     deck gateway diff $D "$KONG_YML" ;;
  apply)    deck gateway validate $D "$KONG_YML"
            backup
            deck gateway sync $D "$KONG_YML" ;;
  backup)   backup ;;
  restore)  [ -n "${2:-}" ] || { echo "usage: $0 restore FILE"; exit 1; }
            backup
            deck gateway sync $D "$2" ;;
  *) sed -n '2,7p' "$0"; exit 1 ;;
esac
EOF
chmod +x /opt/kong/kong-apply.sh
```

Do the first apply. It creates all 9 services and routes in the empty database:

```bash
/opt/kong/kong-apply.sh apply
```

Expect `Summary: Created: 18  Updated: 0  Deleted: 0` (9 services and 9 routes).
Each Kong worker checks the database for changes every 5 seconds
(`KONG_WORKER_STATE_UPDATE_FREQUENCY`, default 5), so a change can take up to
about 5 seconds to reach the proxy. No restart is needed.

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
> or, easier, let decK export it as ready-to-use YAML:
> `deck gateway dump --kong-addr http://OLD_HOST:8001 -o old.yaml`

---

## 10. Verify

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

## 11. Day-2 operations: edit the YAML, then apply

```bash
docker logs kong --tail 50          # access + error logs
docker logs kong-db --tail 50       # postgres logs
docker inspect kong --format '{{.State.Health.Status}}'
```

**Every change (routes, services, plugins, consumers) follows the same steps:**

```bash
vi /opt/kong/kong.yml               # 1. edit the source of truth
/opt/kong/kong-apply.sh diff        # 2. preview (nothing is changed yet)
/opt/kong/kong-apply.sh apply       # 3. backup + push; live in ~5s, no restart
```

> **Important: `apply` is a FULL sync.** Anything in Kong that isn't in
> `kong.yml` gets **deleted**. Don't make changes with curl or Kong Manager;
> they'll be wiped on the next apply. If you have to fix something by hand in
> an emergency, copy the change into `kong.yml` afterwards.

### Example: add a service and route

Append under `services:` in `kong.yml`:

```yaml
  - name: example
    host: 192.168.1.50
    port: 8080
    protocol: http
    routes:
      - name: example-root
        hosts:
          - example.cropseyit.com
        paths:
          - /
        strip_path: false
        preserve_host: true
```

### Example: remove a route or service

Delete its block from `kong.yml` and run `apply`. decK deletes it from Kong.

### Example: add plugins

Plugins are YAML as well. A plugin can belong to one service, or be global
(top-level `plugins:` key):

```yaml
services:
  - name: juice-shop
    # ...existing fields...
    plugins:
      - name: rate-limiting        # applies to this service only
        config:
          minute: 60
          policy: local

plugins:                           # top-level = global, applies to all traffic
  - name: correlation-id
    config:
      header_name: X-Request-ID
      generator: uuid
      echo_downstream: true
```

Bundled plugins (rate-limiting, key-auth, cors, etc.) work without any other
setup. A **custom** plugin (such as `nonamesecurity`) also has to be in the
image and enabled with `KONG_PLUGINS=bundled,<name>` on the `kong` container
before decK can configure it.

### Secrets in `kong.yml`

If `kong.yml` is committed to git, don't put secrets (API keys, credentials) in
it directly. decK can read them from environment variables that start with `DECK_`:

```yaml
    config:
      api_key: ${{ env "DECK_UPSTREAM_API_KEY" }}
```

```bash
DECK_UPSTREAM_API_KEY=... /opt/kong/kong-apply.sh apply
```

### Roll back

Every `apply` (and `restore`) saves the state from just before it ran. To undo
a change, restore the newest backup taken before that change:

```bash
ls ~/kong-backups/
/opt/kong/kong-apply.sh restore ~/kong-backups/kong-YYYY-MM-DD-HHMMSS.yaml
```

### Backup / restore the database (full Postgres dump)

`kong.yml` and `~/kong-backups/` already cover the Kong config. A database dump
also captures anything outside decK's scope:

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
| A route/plugin you added with curl disappeared | `kong-apply.sh apply` is a full sync, so it removes anything not in `kong.yml`. Add the route or plugin to the YAML |
| Right after `apply`, the old route/plugin still answers for a few seconds | Normal in DB mode: workers pick up changes within ~5 s. Wait and retry |
| decK: `error connecting to Kong` | Kong isn't up yet, or the Admin API isn't on `localhost:8001` (set `KONG_ADDR=http://HOST:8001`) |
| decK: `plugin 'X' not enabled` / schema error | Custom plugin isn't in the image or `KONG_PLUGINS`, or there's a typo in the plugin `config` |
| `apply` shows unexpected deletes | Something was changed outside the YAML. Stop and copy it into `kong.yml` first if you want to keep it |
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
| Source of truth | `/opt/kong/kong.yml` | `/opt/kong/kong.yml` (same file) |
| How config reaches Kong | mounted read-only, loaded at startup | decK syncs it into Postgres |
| Change config | edit YAML + `docker restart kong` | edit YAML + `kong-apply.sh apply` (live, no restart) |
| Plugins with DB-only features (e.g. `rate-limiting` `policy: cluster`) | not available | available |
| Extra steps | none | network, DB password, `migrations bootstrap`, decK |
| Backups | copy the YAML | YAML + automatic `~/kong-backups/` on every apply (+ optional `pg_dump`) |

---

## Appendix: how .100 (reference) is set up

- Kong 3.9.4 community, **DB mode** (Postgres 16 in a `kong-db` podman container)
- Custom image `localhost/kong-noname:3.9.4-5.0.1-src4` (bundles the `nonamesecurity` plugin)
- Plugins: `rate-limiting` on every service + one global `nonamesecurity`
  plugin (intentionally **not** replicated here)
- Proxy exposed on host port 80, admin on 8001
