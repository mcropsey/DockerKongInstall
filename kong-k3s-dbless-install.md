# Install Kong 3.9.4 in DBLESS Mode on k3s (Kubernetes)

This is a step-by-step guide for running Kong in **DBLESS mode** on a 3-node **k3s** cluster
(Ubuntu 24.04). It uses the same routes and rate limits as the existing gateway (192.168.1.100).

Verified working on the k3s cluster **192.168.1.91 / .92 / .93** (Nov 2026).

> Need the cluster first? See
> [mcropsey/ubuntu-k3s-cluster-setup](https://github.com/mcropsey/ubuntu-k3s-cluster-setup):
> k3s v1.37.1, server on .91, agents on .92/.93, Traefik disabled.

---

## 0. What this gives you

- Kong **3.9.4** community (stock `kong:3.9.4` image), **2 replicas** spread across nodes
- **DBLESS** mode: no Postgres. The config is one YAML file (`kong.yml`), mounted from a ConfigMap.
- 9 services + 9 routes (crapi, mailhog, dvga, jenkins, juice-nomon, juice-shop, k8s, notes, vampi)
- **rate-limiting** on every service (1000/min, 20000/hour, per client IP), the same as .100
- **No Noname plugin** (see section 9)
- Proxy on **NodePort 30080** (http) / **30443** (https) on *every* node
- The Admin API is **cluster-internal only** (reach it with `kubectl port-forward`)

How it differs from the Docker DBLESS guide (`kong-dbless-install.md`):

| | Docker guide | This guide (k3s) |
|---|---|---|
| Runs as | 1 container on 1 host | Deployment, 2 pods, any node |
| Config file | `/opt/kong/kong.yml` bind mount | `kong.yml` → ConfigMap (Kustomize) |
| Apply changes | `docker restart kong` | `kubectl apply -k .` (rolling restart) |
| Proxy port | 80 / 8000 | NodePort 30080 / 30443 |
| Admin API | :8001 on the LAN | ClusterIP + `port-forward` |

---

## 1. Prerequisites

```bash
# kubectl works against the cluster (on ubuntu-1, no sudo needed)
kubectl get nodes          # all nodes Ready

# NodePorts 30080 / 30443 must be free
kubectl get svc -A | grep -E '3008|30443' || echo "free"
```

Why NodePorts and not :80/:443? If ubuntu-1 also runs Docker Kong, that Kong already holds
:80/:8000/:8001. A k3s `LoadBalancer` Service on :80 would grab :80 on **every** node,
.91 included, and collide with it.

---

## 2. Create a working directory

```bash
mkdir -p ~/kong-dbless && cd ~/kong-dbless
```

All the files below go in this directory.

---

## 3. Write `kong.yml` (the declarative config)

These are the 9 services/routes and rate limits from a live `deck gateway dump` of .100
(2026-11-03), with the Noname plugin removed:

```yaml
# Kong 3.9.4 DB-less config. Services/routes/rate limits mirror .100 (live decK dump 2026-11-03).
# The Noname plugin is NOT included (stock kong image).
_format_version: '3.0'
services:
- name: crapi-app
  url: http://192.168.1.101:8888
  routes:
  - name: crapi-root
    hosts:
    - crapi.cropseyit.com
    paths:
    - /
    preserve_host: true
    strip_path: false
    protocols:
    - http
    - https
  plugins:
  - name: rate-limiting
    config:
      minute: 1000
      hour: 20000
      limit_by: ip
      policy: local
      fault_tolerant: true
- name: crapi-mail
  url: http://192.168.1.101:8025
  routes:
  - name: crapi-mailhog
    hosts:
    - mail.cropseyit.com
    paths:
    - /
    preserve_host: true
    strip_path: false
    protocols:
    - http
    - https
  plugins:
  - name: rate-limiting
    config:
      minute: 1000
      hour: 20000
      limit_by: ip
      policy: local
      fault_tolerant: true
- name: dvga
  url: http://192.168.1.101:5013
  routes:
  - name: dvga-root
    hosts:
    - dvga.cropseyit.com
    paths:
    - /
    preserve_host: true
    strip_path: false
    protocols:
    - http
    - https
  plugins:
  - name: rate-limiting
    config:
      minute: 1000
      hour: 20000
      limit_by: ip
      policy: local
      fault_tolerant: true
- name: jenkins
  url: http://192.168.1.100:8080
  routes:
  - name: jenkins-root
    hosts:
    - jenkins.cropseyit.com
    paths:
    - /
    preserve_host: true
    strip_path: false
    protocols:
    - http
    - https
  plugins:
  - name: rate-limiting
    config:
      minute: 1000
      hour: 20000
      limit_by: ip
      policy: local
      fault_tolerant: true
- name: juice-nomon
  url: http://192.168.1.99:30300
  routes:
  - name: juice-nomon-root
    hosts:
    - juice-nomon.cropseyit.com
    paths:
    - /
    preserve_host: true
    strip_path: false
    protocols:
    - http
    - https
  plugins:
  - name: rate-limiting
    config:
      minute: 1000
      hour: 20000
      limit_by: ip
      policy: local
      fault_tolerant: true
- name: juice-shop
  url: http://192.168.1.101:3000
  routes:
  - name: juice-root
    hosts:
    - juice.cropseyit.com
    paths:
    - /
    preserve_host: true
    strip_path: false
    protocols:
    - http
    - https
  plugins:
  - name: rate-limiting
    config:
      minute: 1000
      hour: 20000
      limit_by: ip
      policy: local
      fault_tolerant: true
- name: k8s
  url: http://192.168.1.75:80
  routes:
  - name: k8s-root
    hosts:
    - k8s.cropseyit.com
    paths:
    - /
    preserve_host: true
    strip_path: false
    protocols:
    - http
    - https
  plugins:
  - name: rate-limiting
    config:
      minute: 1000
      hour: 20000
      limit_by: ip
      policy: local
      fault_tolerant: true
- name: notes
  url: http://192.168.1.98:8000
  routes:
  - name: notes-root
    hosts:
    - notes.cropseyit.com
    paths:
    - /
    preserve_host: true
    strip_path: false
    protocols:
    - http
    - https
  plugins:
  - name: rate-limiting
    config:
      minute: 1000
      hour: 20000
      limit_by: ip
      policy: local
      fault_tolerant: true
- name: vampi
  url: http://192.168.1.99:30500
  routes:
  - name: vampi-root
    hosts:
    - vampi.cropseyit.com
    paths:
    - /
    preserve_host: true
    strip_path: false
    protocols:
    - http
    - https
  plugins:
  - name: rate-limiting
    config:
      minute: 1000
      hour: 20000
      limit_by: ip
      policy: local
      fault_tolerant: true
```

To check the routes against .100 again later (run on .100), compare its services/routes
with this file:

```bash
deck gateway dump --kong-addr http://localhost:8001 -o - | grep -v NN_SOURCE_KEY
```

> The `NN_SOURCE_KEY` filter matters: a dump of .100 includes the Noname source key in
> plain text. Never commit it.

---

## 4. Write `namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kong
```

---

## 5. Write `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kong
  labels: {app: kong}
spec:
  replicas: 2
  selector:
    matchLabels: {app: kong}
  template:
    metadata:
      labels: {app: kong}
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels: {app: kong}
      containers:
        - name: kong
          image: kong:3.9.4
          env:
            - {name: KONG_DATABASE, value: "off"}
            - {name: KONG_DECLARATIVE_CONFIG, value: /kong/kong.yml}
            - {name: KONG_PROXY_LISTEN, value: "0.0.0.0:8000, 0.0.0.0:8443 ssl"}
            - {name: KONG_ADMIN_LISTEN, value: "0.0.0.0:8001"}
            - {name: KONG_STATUS_LISTEN, value: "0.0.0.0:8100"}
            - {name: KONG_PROXY_ACCESS_LOG, value: /dev/stdout}
            - {name: KONG_ADMIN_ACCESS_LOG, value: /dev/stdout}
            - {name: KONG_PROXY_ERROR_LOG, value: /dev/stderr}
            - {name: KONG_ADMIN_ERROR_LOG, value: /dev/stderr}
          ports:
            - {name: proxy, containerPort: 8000}
            - {name: proxy-ssl, containerPort: 8443}
            - {name: admin, containerPort: 8001}
            - {name: status, containerPort: 8100}
          readinessProbe:
            httpGet: {path: /status/ready, port: status}
            periodSeconds: 5
          livenessProbe:
            httpGet: {path: /status, port: status}
            initialDelaySeconds: 10
            periodSeconds: 10
          resources:
            requests: {cpu: 100m, memory: 256Mi}
            limits: {memory: 1Gi}
          volumeMounts:
            - {name: config, mountPath: /kong, readOnly: true}
      volumes:
        - name: config
          configMap:
            name: kong-declarative
```

Notes:
- `KONG_DATABASE=off` + `KONG_DECLARATIVE_CONFIG` turn on DBLESS mode.
- The readiness probe on `/status/ready` (status listener on :8100) keeps a pod out of
  the Service until it has loaded the config.
- `topologySpreadConstraints` puts the 2 replicas on different nodes.

---

## 6. Write `service.yaml`

```yaml
# Proxy on NodePorts (every node). NodePorts avoid :80/:443/:8000/:8001 because Docker Kong on .91 uses them.
apiVersion: v1
kind: Service
metadata:
  name: kong-proxy
spec:
  type: NodePort
  selector: {app: kong}
  ports:
    - {name: proxy, port: 80, targetPort: proxy, nodePort: 30080}
    - {name: proxy-ssl, port: 443, targetPort: proxy-ssl, nodePort: 30443}
---
# Admin API is cluster-internal only (no auth). Use: kubectl -n kong port-forward svc/kong-admin 8001
apiVersion: v1
kind: Service
metadata:
  name: kong-admin
spec:
  type: ClusterIP
  selector: {app: kong}
  ports:
    - {name: admin, port: 8001, targetPort: admin}
```

---

## 7. Write `kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: kong
resources:
  - namespace.yaml
  - deployment.yaml
  - service.yaml
# Hashed ConfigMap name: editing kong.yml + `kubectl apply -k` rolls the pods automatically.
configMapGenerator:
  - name: kong-declarative
    files:
      - kong.yml
```

`configMapGenerator` adds a content hash to the ConfigMap name (e.g. `kong-declarative-7g9df25mtd`).
When `kong.yml` changes, the name changes, so the Deployment rolls out new pods automatically.
You never have to restart Kong by hand.

---

## 8. Deploy and verify

```bash
cd ~/kong-dbless
kubectl apply -k .
kubectl -n kong rollout status deploy/kong
kubectl -n kong get pods,svc -o wide
```

Expected:

```
pod/kong-xxxxxxxxx-xxxxx   1/1   Running   ...   ubuntu-2
pod/kong-xxxxxxxxx-xxxxx   1/1   Running   ...   ubuntu-3
service/kong-admin   ClusterIP   ...   8001/TCP
service/kong-proxy   NodePort    ...   80:30080/TCP,443:30443/TCP
```

### Check the version, DBLESS mode and object counts

The `kong` image has no `curl` or `wget`, so check from the host through a port-forward:

```bash
kubectl -n kong port-forward svc/kong-admin 18001:8001 >/dev/null 2>&1 &
sleep 2
curl -s localhost:18001/ | python3 -c "import json,sys;d=json.load(sys.stdin);print(d['version'], d['configuration']['database'])"
#  3.9.4 off
for e in services routes plugins; do echo -n "$e: "; curl -s localhost:18001/$e | python3 -c "import json,sys;print(len(json.load(sys.stdin)['data']))"; done
#  services: 9   routes: 9   plugins: 9
kill %1
```

### Test every route (any node IP works)

```bash
for h in crapi mail dvga jenkins juice-nomon juice k8s notes vampi; do
  printf "%-12s k3s=%s  .100=%s\n" $h \
    $(curl -s -o /dev/null -w '%{http_code}' -H "Host: $h.cropseyit.com" http://192.168.1.92:30080/) \
    $(curl -s -o /dev/null -w '%{http_code}' -H "Host: $h.cropseyit.com" http://192.168.1.100/)
done
```

Result on 2026-11-03 (the same from .91, .92 and .93):

```
crapi        k3s=200  .100=200
mail         k3s=200  .100=200
dvga         k3s=200  .100=200
jenkins      k3s=403  .100=403     <- Jenkins' own login page, not Kong
juice-nomon  k3s=200  .100=200
juice        k3s=200  .100=200
k8s          k3s=200  .100=200
notes        k3s=200  .100=200
vampi        k3s=200  .100=200
```

Rate-limit headers and an unknown host:

```bash
curl -sI -H 'Host: juice.cropseyit.com' http://192.168.1.92:30080/ | grep -iE 'ratelimit-limit|^via'
#  X-RateLimit-Limit-Minute: 1000
#  X-RateLimit-Limit-Hour: 20000
#  Via: 1.1 kong/3.9.4
curl -s -H 'Host: nope.example' http://192.168.1.92:30080/
#  {"message":"no Route matched with those values", ...}
```

---

## 9. Things to know

- **No Noname plugin.** .100 runs a global `nonamesecurity` plugin. Adding it here would need
  the custom `kong-noname:3.9.4` image on every node (e.g. `docker save | sudo k3s ctr images import -`),
  `KONG_PLUGINS=bundled,nonamesecurity`, and the plugin block in `kong.yml`. That image contains
  the source key, so never push it to a registry, and never commit a `kong.yml` that holds the key.
- **The Admin API is read-only in DBLESS mode.** Changes go through `kong.yml` plus `kubectl apply -k .`,
  not `curl -X POST`.
- **Rate limits are counted per pod** (`policy: local`). With 2 replicas, one client can get up to
  about 2× the limit. For exact limits, use `replicas: 1` or a Redis policy.
- **A bad config doesn't take you down.** If `kong.yml` is invalid, the new pods fail readiness, and
  the old pods keep serving. Check with `kubectl -n kong logs deploy/kong`.
- **Docker Kong and k3s Kong can run side by side on .91.** They use different ports
  (:80/:8000/:8001 versus 30080/30443).

---

## 10. Update routes

```bash
cd ~/kong-dbless
vi kong.yml                       # edit services/routes
kubectl apply -k .                # new ConfigMap hash, so a rolling restart
kubectl -n kong rollout status deploy/kong
```

Optional: validate before applying (ubuntu-1 has Docker):

```bash
docker run --rm -v "$PWD:/k" -e KONG_DATABASE=off kong:3.9.4 kong config parse /k/kong.yml
# prints "parse successful"
```

---

## 11. Remove

```bash
cd ~/kong-dbless
kubectl delete -k .               # deletes namespace kong and everything in it
```
