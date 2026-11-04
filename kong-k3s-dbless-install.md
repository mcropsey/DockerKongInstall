# Install Kong 3.9.4 in DBLESS Mode on k3s (Kubernetes)

This is a step-by-step guide for running Kong in **DBLESS mode** on a 3-node **k3s** cluster
(Ubuntu 24.04). It uses the same routes and rate limits as the existing gateway (192.168.1.100).

Verified working on the k3s cluster **192.168.1.91 / .92 / .93** (Nov 2026).

> Need the cluster first? See
> [mcropsey/ubuntu-k3s-cluster-setup](https://github.com/mcropsey/ubuntu-k3s-cluster-setup):
> k3s v1.37.1, server on .91, agents on .92/.93, Traefik disabled (Kong is the ingress).

---

## 0. What this gives you

- Kong **3.9.4** community (stock `kong:3.9.4` image), run as a **DaemonSet**: one Kong pod on every node
- **DBLESS** mode: no Postgres. The config is one YAML file (`kong.yml`), mounted from a ConfigMap.
- 9 services + 9 routes (crapi, mailhog, dvga, jenkins, juice-nomon, juice-shop, k8s, notes, vampi)
- **rate-limiting** on every service (1000/min, 20000/hour, per client IP), the same as .100
- Proxy on **port 80** (http) and **443** (https) on **all 3 node IPs**, through the k3s built-in
  load balancer (ServiceLB). NodePorts 30080/30443 also work.
- The real client IP reaches Kong, so per-IP rate limiting works
- Zero-downtime config changes and restarts (tested: 1200/1200 requests OK during a rolling restart)
- The Admin API is **cluster-internal only** (reach it with `kubectl port-forward`)

How it differs from the Docker DBLESS guide (`kong-dbless-install.md`):

| | Docker guide | This guide (k3s) |
|---|---|---|
| Runs as | 1 container on 1 host | DaemonSet, 1 pod per node |
| Config file | `/opt/kong/kong.yml` bind mount | `kong.yml` → ConfigMap (Kustomize) |
| Apply changes | `docker restart kong` | `kubectl apply -k .` (rolling, no downtime) |
| Proxy port | 80 / 8000 on one host | 80 / 443 on every node (plus 30080 / 30443) |
| Admin API | :8001 on the LAN | ClusterIP + `port-forward` |

---

## 1. Prerequisites

```bash
# kubectl works against the cluster (on ubuntu-1, no sudo needed)
kubectl get nodes          # all nodes Ready

# Ports 80 and 443 must be free on EVERY node: no Traefik, no Docker Kong, no other web server
for n in 192.168.1.91 192.168.1.92 192.168.1.93; do
  echo "$n: $(ssh $n 'sudo ss -tlnH "( sport = :80 or sport = :443 )"' | wc -l) listeners (want 0)"
done
kubectl get svc -A | grep LoadBalancer || echo "no other LoadBalancer services"
```

> **Don't run Docker Kong (the other guides) on these nodes at the same time.** It also binds :80.
> Remove this deployment first (section 11), or use a different host.

---

## 2. Create a working directory

```bash
mkdir -p ~/kong-dbless && cd ~/kong-dbless
```

All the files below go in this directory.

---

## 3. Write `kong.yml` (the declarative config)

These are the 9 services/routes and rate limits from the live config on .100 (2026-11-03):

```yaml
# Kong 3.9.4 DB-less config. Services/routes/rate limits mirror .100 (live decK dump 2026-11-03).
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

To compare against .100's routes later:

```bash
curl -s http://192.168.1.100:8001/routes | python3 -c "import json,sys;[print(r['name'], r['hosts']) for r in json.load(sys.stdin)['data']]"
```

---

## 4. Write `namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: kong
```

---

## 5. Write `daemonset.yaml`

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kong
  labels: {app: kong}
spec:
  # One Kong pod per node (kong-proxy uses externalTrafficPolicy Local, so every node needs a local pod).
  # maxSurge: the new pod is Ready before the old one stops, so :80 never drops during a config change.
  updateStrategy:
    type: RollingUpdate
    rollingUpdate: {maxSurge: 1, maxUnavailable: 0}
  selector:
    matchLabels: {app: kong}
  template:
    metadata:
      labels: {app: kong}
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - name: kong
          image: kong:3.9.4
          # Wait for the endpoint to be removed from the Service before Kong stops, then let it finish in-flight requests.
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 10 && kong quit --wait=15"]
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
- **Why a DaemonSet, not a Deployment:** the proxy Service uses `externalTrafficPolicy: Local`
  (section 6), so a node only answers on :80 if a Kong pod runs **on that node**. A DaemonSet
  guarantees exactly one per node. With a Deployment, two pods can end up on one node, and
  another node silently stops serving :80.
- **`maxSurge: 1, maxUnavailable: 0`:** during an update, the new pod on a node becomes Ready
  before the old one stops.
- **`preStop` (sleep 10 + `kong quit --wait=15`):** gives the Service time to stop sending
  traffic to the old pod, then lets Kong finish in-flight requests. Without it, about 0.5% of
  requests failed during a rolling restart. With it, none did.
- The readiness probe on `/status/ready` (status listener on :8100) keeps a pod out of the
  Service until it has loaded the config.

---

## 6. Write `service.yaml`

```yaml
# Proxy on :80/:443 on EVERY node via k3s ServiceLB (klipper-lb); NodePorts 30080/30443 also work.
# Don't run Docker Kong on these nodes at the same time; it also binds :80.
apiVersion: v1
kind: Service
metadata:
  name: kong-proxy
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local   # keep the real client IP (rate-limiting is per IP)
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

Notes:
- `type: LoadBalancer` on k3s is handled by **ServiceLB** (klipper-lb). It starts a small
  `svclb-kong-proxy-*` pod on each node (in `kube-system`) that binds host ports 80/443 and
  forwards them to Kong. The Service's EXTERNAL-IP becomes the list of node IPs.
- `externalTrafficPolicy: Local` keeps the **real client IP**. With the default (`Cluster`),
  Kong would see a node IP, and per-IP rate limiting would limit whole nodes instead of clients.

---

## 7. Write `kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: kong
resources:
  - namespace.yaml
  - daemonset.yaml
  - service.yaml
# Hashed ConfigMap name: editing kong.yml + `kubectl apply -k` rolls the pods automatically.
configMapGenerator:
  - name: kong-declarative
    files:
      - kong.yml
```

`configMapGenerator` adds a content hash to the ConfigMap name (e.g. `kong-declarative-t2d94dhth5`).
When `kong.yml` changes, the name changes, so the DaemonSet rolls out new pods automatically.
You never have to restart Kong by hand.

---

## 8. Deploy and verify

```bash
cd ~/kong-dbless
kubectl apply -k .
kubectl -n kong rollout status ds/kong
kubectl -n kong get pods,svc -o wide
```

Expected (one Kong pod per node, and all 3 node IPs as EXTERNAL-IP):

```
pod/kong-xxxxx   1/1   Running   ...   ubuntu-1
pod/kong-xxxxx   1/1   Running   ...   ubuntu-2
pod/kong-xxxxx   1/1   Running   ...   ubuntu-3
service/kong-admin   ClusterIP      ...   <none>                                   8001/TCP
service/kong-proxy   LoadBalancer   ...   192.168.1.91,192.168.1.92,192.168.1.93   80:30080/TCP,443:30443/TCP
```

If EXTERNAL-IP stays `<pending>` or is missing a node, something else holds :80/:443 on that node.
Check `kubectl -n kube-system get pods -l svccontroller.k3s.cattle.io/svcname=kong-proxy -o wide`.
A `Pending` svclb pod means its ports are taken.

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

### Test every route on port 80, on every node

```bash
for n in 91 92 93; do
  for h in crapi mail dvga jenkins juice-nomon juice k8s notes vampi; do
    printf "%s " $(curl -s -o /dev/null -w '%{http_code}' -H "Host: $h.cropseyit.com" http://192.168.1.$n/)
  done; echo " <- 192.168.1.$n:80"
done
```

Result on 2026-11-03 (the order is crapi mail dvga jenkins juice-nomon juice k8s notes vampi). It's identical to .100:

```
200 200 200 403 200 200 200 200 200  <- 192.168.1.91:80
200 200 200 403 200 200 200 200 200  <- 192.168.1.92:80
200 200 200 403 200 200 200 200 200  <- 192.168.1.93:80
```

The jenkins 403 is Jenkins' own login page, not Kong.

HTTPS, rate-limit headers and an unknown host:

```bash
curl -sk -o /dev/null -w '%{http_code}\n' -H 'Host: juice.cropseyit.com' https://192.168.1.91/
#  200   (Kong's self-signed default cert, hence -k)
curl -sI -H 'Host: juice.cropseyit.com' http://192.168.1.91/ | grep -iE 'ratelimit-limit|^via'
#  X-RateLimit-Limit-Minute: 1000
#  X-RateLimit-Limit-Hour: 20000
#  Via: 1.1 kong/3.9.4
curl -s -H 'Host: nope.example' http://192.168.1.91/
#  {"message":"no Route matched with those values", ...}
```

To check that Kong sees the real client IP, look at the first field of the access log:

```bash
curl -s -o /dev/null -H 'Host: notes.cropseyit.com' http://192.168.1.93/ip-check
for p in $(kubectl -n kong get pod -o name); do kubectl -n kong logs $p --since=1m | grep ip-check; done
#  192.168.1.98 - - [...] "GET /ip-check HTTP/1.1" ...      <- your machine's IP, not a node IP
```

### Zero-downtime test (optional)

```bash
( for i in $(seq 1 400); do for n in 91 92 93; do
    curl -s -o /dev/null -m 3 -w '%{http_code}\n' -H 'Host: juice.cropseyit.com' http://192.168.1.$n/
  done; sleep 0.2; done ) > /tmp/load.txt &
sleep 2; kubectl -n kong rollout restart ds/kong; kubectl -n kong rollout status ds/kong; wait
sort /tmp/load.txt | uniq -c        # 2026-11-03: 1200 200
```

---

## 9. Things to know

- **Kong is the ingress.** There is no Traefik and no Kubernetes `Ingress` objects. Routing is by
  `Host` header, from `kong.yml`. Point DNS for `*.cropseyit.com` (or a front proxy/tunnel) at any
  or all of the node IPs.
- **Nothing else can use :80/:443 on these nodes**, including Docker Kong from the other guides.
- **The Admin API is read-only in DBLESS mode.** Changes go through `kong.yml` plus `kubectl apply -k .`,
  not `curl -X POST`.
- **Rate limits are counted per pod** (`policy: local`), so in effect per node. A client that
  spreads requests across all 3 node IPs can get up to 3× the limit. For exact cluster-wide
  limits, use a Redis policy.
- **A bad config doesn't take you down.** If `kong.yml` is invalid, the new pod fails readiness, and
  because of `maxUnavailable: 0` the old pods keep serving. Check with `kubectl -n kong logs ds/kong`.
- **If a node goes down,** its IP stops answering. The other two keep serving.

---

## 10. Update routes

```bash
cd ~/kong-dbless
vi kong.yml                       # edit services/routes
kubectl apply -k .                # new ConfigMap hash, so a rolling restart with no downtime
kubectl -n kong rollout status ds/kong
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
kubectl delete -k .               # deletes namespace kong and everything in it; frees :80/:443 on all nodes
```
