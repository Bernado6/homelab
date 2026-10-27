# 19. Using the Metrics

## What

Step 18 installed the stack. This step makes it earn its keep:

1. Read the built-in dashboards and know which three to open.
2. Write PromQL against your own pods.
3. Scrape **Traefik**, so you can see requests per site (`portfolio.home.arpa` vs `nginx.home.arpa`).
4. Use a week of data to set `requests` and `limits` from evidence instead of guesses.

## Why

A monitoring stack that nobody looks at is just 1GB of RAM doing nothing. The point of this step is to build the habit: before changing a limit, adding an app, or buying RAM, open Grafana first.

For CKAD, `requests` and `limits` are tested directly. Knowing where the numbers come from makes them stick.

## When

A day or two after step 18, so there's enough history for the graphs to mean something. Then again whenever you add an app.

## How

### 1. The three dashboards worth knowing

Grafana → **Dashboards** → folder **Kubernetes**.

| Dashboard | Question it answers |
|---|---|
| **Kubernetes / Compute Resources / Cluster** | How full is the box? Top rows show CPU and memory *requests* and *limits* as a share of the node, which is the "can I fit another app" number. |
| **Kubernetes / Compute Resources / Namespace (Pods)** | Pick `portfolio` in the namespace dropdown: per-pod CPU and memory, and how close each pod is to its limit. |
| **Node Exporter / Nodes** | The server itself: memory, disk space on `/`, network, load. Disk space is the one to watch with Prometheus writing to it. |

### 2. PromQL you'll actually use

Open **`http://prometheus.home.arpa/graph`** or Grafana → **Explore**. Each query below is one line; replace `portfolio` with any namespace.

**Memory in use per pod (what the limit is compared against):**

```promql
sum(container_memory_working_set_bytes{namespace="portfolio", container!=""}) by (pod)
```

**CPU per pod, in cores, averaged over 5 minutes:**

```promql
sum(rate(container_cpu_usage_seconds_total{namespace="portfolio", container!=""}[5m])) by (pod)
```

**How close each container is to its memory limit (1.0 = about to be OOM-killed):**

```promql
sum(container_memory_working_set_bytes{namespace="portfolio", container!=""}) by (pod, container)
  /
sum(kube_pod_container_resource_limits{namespace="portfolio", resource="memory"}) by (pod, container)
```

**Restarts in the last hour, cluster-wide:**

```promql
sum(increase(kube_pod_container_status_restarts_total[1h])) by (namespace, pod) > 0
```

**Free memory on the server as a fraction:**

```promql
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes
```

Reading PromQL: a metric name selects a set of time series, `{…}` filters by label, `rate(…[5m])` turns a counter into a per-second rate, `sum(…) by (pod)` collapses the rest.

### 3. Scrape Traefik: requests per site

Traefik already counts every request it routes. It just isn't being scraped. Two pieces: tell Traefik to expose its metrics service, and give the operator a `ServiceMonitor` so Prometheus finds it.

K3s installs Traefik from a bundled Helm chart, and the supported way to change its values is a **`HelmChartConfig`** dropped into the server's manifests folder. K3s applies it automatically.

On the **server**:

```bash
sudo tee /var/lib/rancher/k3s/server/manifests/traefik-config.yaml > /dev/null <<'EOF'
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  valuesContent: |-
    metrics:
      prometheus:
        addEntryPointsLabels: true
        addRoutersLabels: true
        addServicesLabels: true
        service:
          enabled: true
        serviceMonitor:
          enabled: true
          namespace: kube-system
EOF
```

K3s notices the file and re-runs the Traefik chart. Back on the **laptop**, watch it roll:

```bash
kubectl -n kube-system rollout status deploy/traefik
kubectl -n kube-system get svc traefik-metrics
kubectl -n kube-system get servicemonitor traefik
```

Within a minute, **`http://prometheus.home.arpa/targets`** gains a `traefik` target. Then:

**Requests per second per router (one router per Ingress host):**

```promql
sum(rate(traefik_router_requests_total[5m])) by (router)
```

**Error rate per service:**

```promql
sum(rate(traefik_service_requests_total{code=~"5.."}[5m])) by (service)
  /
sum(rate(traefik_service_requests_total[5m])) by (service)
```

**p95 latency per service, in seconds:**

```promql
histogram_quantile(0.95, sum(rate(traefik_service_request_duration_seconds_bucket[5m])) by (le, service))
```

Hit `http://portfolio.home.arpa` a few times and watch the `portfolio-portfolio-…@kubernetes` router count climb.

For a ready-made dashboard: Grafana → **Dashboards → New → Import**, search for **Traefik Official Standalone Dashboard** (ID `17346`), pick the Prometheus datasource.

> If `traefik-metrics` or the `ServiceMonitor` doesn't appear, the bundled chart version may use different keys. `kubectl -n kube-system get helmchart traefik -o yaml` shows the chart version, and `kubectl -n kube-system logs job/helm-install-traefik` shows what the re-render did.

### 4. Set limits from evidence

After a few days, for each app:

1. Open **Namespace (Pods)** for that namespace, set the time range to **7 days**.
2. Note the **peak** memory and the **typical** CPU.
3. Set `requests.memory` to roughly the typical value, `limits.memory` to about 1.5× the peak, and `requests.cpu` to the typical. Leave `limits.cpu` off unless something misbehaves; CPU limits throttle, they don't protect anything.
4. Apply, then watch the pod on the same dashboard for a day.

Fill in your findings:

| App | Typical memory | Peak memory | Old limit | New limit |
|---|---|---|---|---|
| portfolio | | | 64Mi | |
| nginx | | | 64Mi | |
| hello | | | 64Mi | |

### 5. Alerts without Alertmanager

The chart installed a set of alert rules. **`http://prometheus.home.arpa/alerts`** shows them. `Watchdog` is **always firing** on purpose: it exists so a notification channel can prove it's alive. Anything else firing, especially `KubePodCrashLooping`, `KubeContainerWaiting`, or `NodeFilesystemSpaceFillingUp`, is worth reading. Wiring these to a phone is a later step, once Alertmanager comes back.

## Challenges and solutions

| Challenge | What was going on | Solution |
|---|---|---|
| Gaps in every graph, and the 7-day view had less than a day of data | I had switched the server off overnight. Prometheus only records while it runs; there's no catching up. | Nothing to fix, but it changes the plan: the limits table in section 4 needs a few days of **continuous** uptime. Leave the box on. It idles at about 10W. |
| Not much else went wrong | Dashboards, PromQL queries, and the Traefik scrape all worked first time. | The Traefik `HelmChartConfig` keys were right for the chart K3s v1.36 bundles. |

*Limits table in section 4 still to be filled after a few days of uptime.*

## Lessons

> 💡 **What I learned:**
> - **Three dashboards cover most questions.** Cluster for headroom, Namespace (Pods) for one app, Node Exporter for the box.
> - **`working_set_bytes` is the number that matters for memory.** It's what the OOM killer compares against the limit.
> - **Counters need `rate()`.** A raw `requests_total` graph only goes up. The rate is the thing you care about.
> - **Changing K3s's bundled Traefik is a file on the server, not a `helm upgrade`.** `HelmChartConfig` is K3s's hook for that, and it survives K3s upgrades.
> - **Limits are measurements, not guesses.** The first week of data usually shows the original numbers were off, in both directions.

---

[◀ Previous: Monitoring: Prometheus and Grafana](18-prometheus-grafana.md) · [🏠 Home](../../README.md) · [Next: HTTPS Everywhere With cert-manager ▶](../09-tls/20-https-with-cert-manager.md)
