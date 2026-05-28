# Monitoring Stack — End-to-End Guide

How metrics flow from a Node.js microservice on a pod, through Prometheus,
into the Grafana dashboard you see in the browser. Written so you can debug
a missing-data problem at any layer without having to re-derive the whole
pipeline from scratch.

The stack itself is the standard **kube-prometheus-stack** Helm chart
(community chart maintained by `prometheus-community`). Nothing here is
custom — what's custom is the **ServiceMonitor**, the **per-service
instrumentation**, and the **Grafana dashboard ConfigMap**. Everything else
is chart defaults.

---

## 1. Components installed in the cluster

All six pods live in the `monitoring` namespace. Verify with:

```powershell
kubectl get pods -n monitoring
```

| Pod | What it does | Image / source |
|---|---|---|
| `prometheus-kube-prometheus-stack-prometheus-0` | TSDB + scraper. Polls every target on an interval, stores samples on a local PVC. | prom/prometheus |
| `alertmanager-kube-prometheus-stack-alertmanager-0` | Receives alerts from Prometheus, dedups, routes to receivers (Slack/email/PagerDuty). **Currently idle — no `PrometheusRule` is defined.** | prom/alertmanager |
| `kube-prometheus-stack-operator-...` | The Prometheus Operator. Watches for `ServiceMonitor` / `PodMonitor` / `PrometheusRule` CRDs and rewrites Prometheus's scrape and rules config so you never edit `prometheus.yml` by hand. | quay.io/prometheus-operator |
| `kube-prometheus-stack-grafana-...` (3/3 containers) | Container 1: Grafana itself. Container 2: dashboard-sidecar — watches ConfigMaps with label `grafana_dashboard: "1"` and POSTs them into Grafana over its API. Container 3: datasource-sidecar — same trick for datasources. | grafana/grafana |
| `kube-prometheus-stack-kube-state-metrics-...` | Subscribes to the K8s API and exports the *state* of objects as metrics (`kube_pod_container_status_restarts_total`, `kube_deployment_status_replicas`, …). This is what powers "Pod Restart Count" on the dashboard. | kube-state-metrics |
| `kube-prometheus-stack-prometheus-node-exporter-...` (DaemonSet) | Runs on every node, exposes node-level metrics: CPU, memory, disk, network. | prom/node-exporter |

The operator + the three "support" components (kube-state-metrics, node-exporter,
the sidecars in Grafana) are why this is called a **stack** — by itself, Prometheus
only knows about HTTP `/metrics` endpoints. The stack glues in the K8s-native
discovery and metadata.

---

## 2. Install / Setup

The chart is installed **independently of Argo CD**. Argo currently syncs only
the boutique application (`gitops/kustomization.yml`); the monitoring chart was
installed once via Helm and is not in GitOps.

Standard install (for reference; do not re-run unless you intend to reinstall):

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create namespace monitoring
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring
```

The Helm release name (`kube-prometheus-stack`) matters: the operator only picks
up ServiceMonitors carrying the label `release: kube-prometheus-stack`. That's
the matchmaking key, set on every ServiceMonitor in this repo.

> If you ever reinstall under a different release name, **every ServiceMonitor
> must be relabeled** or scraping silently stops working.

---

## 3. Instrumentation — where the metrics come from

Each Node.js microservice (gateway, auth, product-service, order-service,
orders, user-service) ships a metrics module at
`projects/boutique-microservices/backend/services/<svc>/src/metrics/metrics.js`.

The shape is identical across services (gateway shown):

```js
const promClient = require('prom-client');
const register = new promClient.Registry();

promClient.collectDefaultMetrics({ register });   // 1. free runtime metrics

const httpRequestDuration = new promClient.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code', 'service_name'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 2, 5],
  registers: [register],
});

const httpRequestTotal = new promClient.Counter({
  name: 'http_requests_total',
  labelNames: ['method', 'route', 'status_code', 'service_name'],
  registers: [register],
});

const httpRequestsInProgress = new promClient.Gauge({
  name: 'http_requests_in_progress',
  labelNames: ['method', 'route', 'service_name'],
  registers: [register],
});
```

Three things happen here:

### 3.1 Default runtime metrics
`collectDefaultMetrics()` registers a batch of Node.js process metrics for free:

- `nodejs_heap_size_used_bytes`, `nodejs_heap_size_total_bytes` — V8 heap.
- `nodejs_eventloop_lag_seconds` — how delayed the event loop is. Climbing
  values indicate the process is CPU-bound or blocked on sync work.
- `nodejs_gc_duration_seconds`, `process_cpu_seconds_total`,
  `process_resident_memory_bytes`, file descriptor count, etc.

These power the "Node.js Heap Memory" and "Event Loop Lag" panels on the dashboard.

### 3.2 Custom HTTP metrics

| Metric | Type | Labels | Used in |
|---|---|---|---|
| `http_requests_total` | Counter | method, route, status_code, service_name | Request rate, error rate panels |
| `http_request_duration_seconds` | Histogram (buckets 0.01–5s) | method, route, status_code, service_name | p95 / p99 latency panels |
| `http_requests_in_progress` | Gauge | method, route, service_name | "Active Requests" stat |
| `service_info` | Gauge | service_name, version | Version surface; useful for sanity checks |

### 3.3 The middleware

```js
function metricsMiddleware(req, res, next) {
  const start = Date.now();
  const route = req.route?.path || req.path || 'unknown';
  const serviceName = req.app.get('serviceName') || 'unknown';

  httpRequestsInProgress.inc({ method: req.method, route, service_name: serviceName });

  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    httpRequestDuration.observe({ ..., status_code: res.statusCode }, duration);
    httpRequestTotal.inc({ ..., status_code: res.statusCode });
    httpRequestsInProgress.dec({ ... });
  });

  next();
}
```

Key detail: `route` is taken from `req.route?.path` when available — that means
parameterised routes (`/products/:id`) collapse to one time-series instead of
producing one per product ID. **High cardinality is the #1 way to blow up
Prometheus**; using the route template avoids it.

`setupMetrics()` then mounts two endpoints on the Express app:
- `GET /metrics` → Prometheus text format (`register.metrics()`).
- `GET /health` → JSON heartbeat used for kubelet probes.

So every pod self-serves its metrics on its own port at `/metrics`.

---

## 4. Discovery — telling Prometheus what to scrape

Prometheus does **not** know about your services on its own. The Prometheus
Operator translates Kubernetes CRDs into scrape config.

### 4.1 The CRD

`gitops/k8s/backend/service-monitor.yml`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: boutique-services
  namespace: monitoring                # must live where the operator looks
  labels:
    release: kube-prometheus-stack     # matches operator's serviceMonitorSelector
spec:
  namespaceSelector:
    matchNames: [boutique]             # scrape Services in the boutique ns
  selector:
    matchLabels:
      app: gateway                     # only Services with this label
  endpoints:
    - port: http                       # named port on the Service
      path: /metrics
      interval: 15s
```

### 4.2 How the matchmaking works, step by step

1. Operator boots, reads its own config: "look at ServiceMonitors with label
   `release: kube-prometheus-stack` in any namespace."
2. It finds `boutique-services`.
3. It evaluates `namespaceSelector` → picks the `boutique` namespace.
4. Inside `boutique`, it lists every Service with label `app: gateway`. The
   gateway Service in `gitops/k8s/backend/gateway.yml` has `labels.app: gateway`
   and a port **named** `http` (line 48) — both must match.
5. The operator regenerates Prometheus's scrape job from these targets and hot-reloads.
6. Prometheus now hits `http://<gateway-pod-ip>:3001/metrics` every 15s.

### 4.3 Known limitation in this repo

The selector is `app: gateway` — so **only the gateway service is currently
being scraped**, despite the dashboard expecting data from all six. The other
five panels (`auth`, `product-service`, `order-service`, `orders`,
`user-service`) will render empty until the selector is broadened.

Fix (either approach):

```yaml
# Option A — list each app explicitly
selector:
  matchExpressions:
    - { key: app, operator: In, values: [gateway, auth, product-service, order-service, orders, user-service] }
```

```yaml
# Option B — add a common label like `monitoring: enabled` to every Service,
# then match on that. Cleaner long-term.
selector:
  matchLabels:
    monitoring: enabled
```

---

## 5. Storage

Prometheus writes samples into a local TSDB on a PVC. Defaults from the chart:

- Retention: **15 days** (override with `prometheus.prometheusSpec.retention`).
- Storage size: chart default ~50Gi (override with `…retentionSize` or PVC spec).
- Sample resolution: 15s scrape interval × however long the series stays active.

If retention has not been customised, plan accordingly — historical
analysis beyond two weeks needs long-term storage (Thanos, Mimir, Cortex)
which is **not** installed.

---

## 6. Grafana — datasource + dashboard provisioning

### 6.1 Datasource

Auto-provisioned by the chart. The operator hands Grafana the URL
`http://kube-prometheus-stack-prometheus.monitoring.svc:9090` and registers it
with UID `prometheus`. Every panel in our dashboard references
`{"type": "prometheus", "uid": "prometheus"}` — that UID is the contract.

(For the local docker-compose dev setup, the same role is filled by
`projects/boutique-microservices/grafana/provisioning/datasources/prometheus.yml`,
which points at `http://prometheus:9090`. The two configs do not interact —
they live in different environments.)

### 6.2 Dashboard ConfigMap

`gitops/k8s/grafana-dashboard.yml` is a single ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboards
  namespace: monitoring
  labels:
    grafana_dashboard: "1"          # the magic label
data:
  boutique-microservices.json: |
    { ...full dashboard JSON... }
```

The Grafana pod's **dashboard sidecar** container is watching the K8s API for
ConfigMaps with this label. When it sees one, it POSTs the JSON to Grafana's
internal API. Update the ConfigMap → sidecar re-POSTs → dashboard updates,
**no Grafana restart required**.

This is why you don't edit dashboards in the Grafana UI and save them — those
edits don't make it back to git. Edit the JSON in the ConfigMap and let
Argo/sidecar do the rest.

### 6.3 Templating variable

The dashboard has one variable: `$service`. Defined as:

```
label_values(up, job)
```

`up` is a synthetic metric Prometheus emits for every scrape target (1 if
the last scrape succeeded, 0 otherwise). `label_values` extracts the distinct
values of the `job` label. So the dropdown is "every job Prometheus is
currently scraping" — dynamically populated, no hard-coding.

---

## 7. The dashboard panels in detail

Source: `gitops/k8s/grafana-dashboard.yml`. Each panel pairs a PromQL query
with a panel type.

### 7.1 Request Rate — `$service`
```promql
sum by (status_code) (rate(http_requests_total{job=~"$service"}[$__rate_interval]))
```
- `rate()` over a counter → per-second request rate.
- `$__rate_interval` is a Grafana-managed window (≥ 4× scrape interval) so the
  rate is statistically stable across zoom levels.
- Sliced by `status_code` so 2xx/4xx/5xx show as separate lines.

### 7.2 Response Time — p95 / p99
```promql
histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket{job=~"$service"}[$__rate_interval])))
```
- `histogram_quantile` interpolates a quantile from bucket counts.
- `sum by (le)` aggregates buckets across replicas — required before the
  quantile function, or you'll get per-pod quantiles which are meaningless.

### 7.3 Active Requests
```promql
sum(http_requests_in_progress{job=~"$service"})
```
Straight gauge sum. Spikes here usually correlate with rising p99.

### 7.4 Error Rate
```promql
sum(rate(http_requests_total{job=~"$service",status_code=~"5.."}[$__rate_interval]))
/
sum(rate(http_requests_total{job=~"$service"}[$__rate_interval]))
```
5xx rate over total rate. Threshold is set red at 5%.

### 7.5 Request Rate by Service
```promql
sum by (job) (rate(http_requests_total[$__rate_interval]))
```
No `$service` filter — gives a side-by-side comparison of all services on
one chart.

### 7.6 Node.js Heap Memory by Service
```promql
nodejs_heap_size_used_bytes{job=~"gateway|auth|product-service|order-service|orders|user-service"}
nodejs_heap_size_total_bytes{...}
```
Two queries → two lines per service. Used vs total tells you whether the
service is approaching GC pressure.

### 7.7 Node.js Event Loop Lag
```promql
nodejs_eventloop_lag_seconds{job=~"…"}
```
If this trends > 100ms, the service is failing to keep up with async work.

### 7.8 Pod CPU / Memory (boutique namespace)
```promql
sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="boutique", container!="", container!="POD"}[$__rate_interval]))
sum by (pod) (container_memory_working_set_bytes{namespace="boutique", container!="", container!="POD"})
```
These metrics come from **cAdvisor**, which runs inside every kubelet and
exposes per-container stats. Prometheus scrapes it automatically because the
chart configures a kubelet scrape job out of the box. The
`container!="", container!="POD"` filters drop the synthetic "pod sandbox"
container so you only see real workload containers.

### 7.9 Pod Restart Count
```promql
kube_pod_container_status_restarts_total{namespace="boutique", container!=""}
```
This comes from **kube-state-metrics**, not from your app — it's how K8s
object state shows up in PromQL.

### 7.10 Service Health (UP/DOWN)
```promql
up{job=~"gateway|auth|product-service|order-service|orders|user-service"}
```
The synthetic `up` metric. 1 = scrape succeeded, 0 = scrape failed. With value
mappings, the stat panel renders "UP" green / "DOWN" red. The cleanest
first-look "is anything on fire" signal.

### 7.11 HTTP Error Rate by Service (4xx / 5xx)
```promql
sum by (job) (rate(http_requests_total{job=~"…", status_code=~"5.."}[$__rate_interval]))
sum by (job) (rate(http_requests_total{job=~"…", status_code=~"4.."}[$__rate_interval]))
```
Wide cross-service view of error rates — useful for spotting "which service
is the source of the spike."

---

## 8. End-to-end flow — one diagram

```
 ┌─────────────────────────────────────────────────────────┐
 │ boutique namespace                                      │
 │                                                         │
 │   ┌──────────────────┐                                  │
 │   │ gateway pod      │  ──── GET /metrics ────┐         │
 │   │ (Express +       │                        │         │
 │   │  prom-client)    │                        │         │
 │   └──────────────────┘                        │         │
 │   ┌──────────────────┐                        │         │
 │   │ auth pod  (etc.) │  ──── GET /metrics ────┤         │
 │   └──────────────────┘                        │         │
 └───────────────────────────────────────────────┼─────────┘
                                                 │ every 15s
                                                 ▼
 ┌─────────────────────────────────────────────────────────┐
 │ monitoring namespace                                    │
 │                                                         │
 │   ServiceMonitor ──watched by──► Prometheus Operator    │
 │   (boutique-services)            (regenerates scrape    │
 │                                   config, hot-reloads)  │
 │                                                         │
 │   ┌────────────────────┐    ┌───────────────────────┐   │
 │   │ Prometheus         │◄───┤ kube-state-metrics    │   │
 │   │ (TSDB + scraper)   │    │ (K8s object state)    │   │
 │   │                    │    └───────────────────────┘   │
 │   │                    │    ┌───────────────────────┐   │
 │   │                    │◄───┤ node-exporter (DS)    │   │
 │   │                    │    │ (host metrics)        │   │
 │   │                    │    └───────────────────────┘   │
 │   │                    │    ┌───────────────────────┐   │
 │   │                    │◄───┤ kubelet / cAdvisor    │   │
 │   │                    │    │ (container metrics)   │   │
 │   └──────────┬─────────┘    └───────────────────────┘   │
 │              │ PromQL                                   │
 │              ▼                                          │
 │   ┌────────────────────────────────────────────────┐    │
 │   │ Grafana                                        │    │
 │   │   - datasource: prometheus (UID "prometheus")  │    │
 │   │   - dashboard sidecar loads ConfigMaps         │    │
 │   │     labelled grafana_dashboard=1               │    │
 │   └────────────────────────────────────────────────┘    │
 │                                                         │
 │   Alertmanager (installed, idle — no PrometheusRule)    │
 └─────────────────────────────────────────────────────────┘
                              │
                              ▼
                  port-forward / Ingress
                              │
                              ▼
                          You, in a browser
```

---

## 9. How to access Grafana / Prometheus

### Grafana

```powershell
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3001:80
```

Then open <http://localhost:3001>. Default credentials from the chart:
- User: `admin`
- Password: stored in the `kube-prometheus-stack-grafana` Secret.

```powershell
kubectl -n monitoring get secret kube-prometheus-stack-grafana `
  -o jsonpath="{.data.admin-password}" | base64 -d
```

### Prometheus UI

```powershell
kubectl -n monitoring port-forward svc/kube-prometheus-stack-prometheus 9090:9090
```

Useful tabs:
- **Status → Targets**: confirms which pods Prometheus is actually scraping
  and the last error if a scrape failed. First place to look when a panel is empty.
- **Status → Service Discovery**: shows the raw discovered targets before
  relabeling — useful for ServiceMonitor debugging.
- **Graph**: ad-hoc PromQL.

---

## 10. Troubleshooting flowchart

**Dashboard panel is empty.**
1. In Grafana, "Inspect → Query" — read the actual PromQL.
2. In Prometheus UI, run the inner query (drop `histogram_quantile`, `rate`,
   etc., just query the base metric like `http_requests_total{job="auth"}`).
   - No data? → Step 3.
   - Data present? → It's a query / templating issue, not a scrape issue.
3. In Prometheus UI → Status → Targets, find the job.
   - Target missing entirely? → ServiceMonitor isn't matching that Service.
     Check labels (`release`, `app`), namespace selector, and the Service's
     **port name** (must be `http`, not just port number).
   - Target present but `DOWN`? → Scrape error visible inline. Common
     causes: pod not exposing `/metrics`, wrong port, NetworkPolicy blocking
     the operator's source IP, container not ready.
4. `kubectl -n boutique exec <pod> -- wget -qO- localhost:<port>/metrics`
   confirms the app is actually serving metrics from inside the pod.

**ServiceMonitor exists but operator isn't picking it up.**
- Wrong namespace (must be `monitoring`, *or* the operator must be configured
  with `serviceMonitorNamespaceSelector` to look elsewhere — we use the simpler
  same-namespace approach).
- Missing the `release: kube-prometheus-stack` label.
- Look at the operator pod logs: it logs which SMs it loaded.

**Grafana shows the dashboard but with "Datasource not found".**
- The dashboard JSON references `uid: "prometheus"`. If your datasource has a
  different UID (e.g. auto-generated GUID), every panel breaks. Set the UID
  explicitly when provisioning the datasource, or edit the dashboard JSON to
  match.

---

## 11. Known gaps (good follow-up work)

1. **ServiceMonitor selector is too narrow** — only gateway is scraped today.
   See §4.3.
2. **No PrometheusRule / alerts.** Alertmanager is running but has nothing to
   route. Easy first set of alerts to add:
   - `up == 0 for 2m` per service (service down).
   - 5xx error rate > 5% for 5m.
   - p99 latency > 1s for 10m.
   - Pod restart rate > 1/hour.
3. **Stack not in GitOps.** The chart was installed by hand. If the cluster is
   ever rebuilt, monitoring won't come back automatically. Putting the chart
   into Argo (as an `Application` of type `Helm`) closes that gap.
4. **No long-term storage.** 15-day retention is fine for ops; not enough for
   capacity planning or postmortems on old incidents.
5. **No log pipeline.** Metrics-only. Adding Loki + Promtail (or
   Loki + Grafana Alloy) under the same Grafana would give logs alongside
   metrics with shared label conventions.

---

## 12. Quick reference — file locations

| What | Where |
|---|---|
| Per-service instrumentation | `projects/boutique-microservices/backend/services/<svc>/src/metrics/metrics.js` |
| ServiceMonitor (cluster) | `gitops/k8s/backend/service-monitor.yml` |
| Grafana dashboard ConfigMap | `gitops/k8s/grafana-dashboard.yml` |
| Local-dev Prometheus config (docker-compose) | `projects/boutique-microservices/prometheus/prometheus.yml` |
| Local-dev Grafana datasource | `projects/boutique-microservices/grafana/provisioning/datasources/prometheus.yml` |
| Gateway Service (port name `http`) | `gitops/k8s/backend/gateway.yml` |
