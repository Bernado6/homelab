# 19. Using the Metrics

Step 18 ended with dashboards full of lines. This step is about reading them, asking Prometheus your own questions, and turning the answers into decisions. It's the page where the resource limits that have been guesses since step 13 get replaced with measured numbers, which is the single most practical thing monitoring does on a small box.

## What

Step 18 installed the stack. This step makes it earn its keep:

1. Read the built-in dashboards and know which three to open.
2. Write PromQL against your own pods.
3. Scrape **Traefik**, so you can see requests per site (`portfolio.home.arpa` vs `nginx.home.arpa`).
4. Use a week of data to set `requests` and `limits` from evidence instead of guesses.

## Background: reading a Grafana dashboard

A Grafana dashboard is a grid of **panels**, each running one or more PromQL queries against Prometheus and drawing the result. The things that control what you see:

| Control | Where | What it does |
|---|---|---|
| **Time range** | Top right, e.g. *Last 1 hour* | The window every panel shows. `Last 7 days` is the one for sizing limits |
| **Refresh** | Next to the time range | How often panels re-query. `10s` for watching something happen, off for reading |
| **Variables** | Dropdowns at the top left (`datasource`, `cluster`, `namespace`) | Substituted into every query on the page. Picking `portfolio` in `namespace` filters the whole dashboard to that namespace |
| **Legend** | Below or beside each graph | One line per series, with the label values that distinguish them. Click one to isolate it |
| **Panel menu** | The three dots on a panel | **Explore** opens the panel's query in the query editor, which is how you learn what a dashboard is actually asking |

The last row is the useful habit: every built-in panel is a worked example of PromQL. When a graph shows something interesting, open it in Explore and read the query.

Two numbers on the Cluster dashboard are worth understanding precisely, because they answer "can I fit another app?":

| Stat | What it is | What it tells you |
|---|---|---|
| **Memory Utilisation** | Memory actually in use across the node, as a share of total | How busy the box is right now |
| **Memory Requests Commitment** | The sum of every pod's `requests.memory`, as a share of the node's allocatable memory | How much the **scheduler** thinks is spoken for. When this reaches 100%, new pods stay `Pending` even if actual usage is low |
| **Memory Limits Commitment** | The sum of every pod's `limits.memory` as a share | Over 100% is normal and fine; it means pods could in theory all peak at once, which they won't |

On my box the first screenshot after install read 37% utilisation, 17% requests, 39% limits. Lots of room, in all three senses.

## Background: PromQL, the five building blocks

PromQL is the query language Prometheus and Grafana speak. Everything in this page is built from five ideas.

### 1. Select series by name and labels

```promql
container_memory_working_set_bytes{namespace="portfolio", container!=""}
```

A metric name picks every series with that name, and `{…}` narrows them by label. `=` is exact match, `!=` is not-equal, `=~` and `!~` match a regular expression. The `container!=""` here drops the per-pod and per-cgroup aggregate rows the kubelet also exports, leaving one series per real container.

### 2. Instant vector vs range vector

The query above returns the **latest value** of each matching series: an *instant vector*. Append a duration in square brackets, `[5m]`, and you get the **last five minutes of samples** for each series instead: a *range vector*. You can't graph a range vector directly; you feed it to a function that reduces it back to one value per series.

### 3. `rate()` turns a counter into something readable

```promql
rate(container_cpu_usage_seconds_total{namespace="portfolio"}[5m])
```

A counter only goes up. `container_cpu_usage_seconds_total` is "total CPU seconds consumed since the container started", and plotting it gives a line that climbs forever. `rate(…[5m])` computes the per-second increase averaged over the window, so CPU-seconds per second, which is **cores**. A value of `0.25` means a quarter of a core. `rate` also handles counter resets when a container restarts.

`increase(…[1h])` is the same idea but gives the total increase over the window instead of a per-second rate; it's the right tool for "how many restarts in the last hour".

### 4. Aggregate with `sum by (…)`

```promql
sum(rate(container_cpu_usage_seconds_total{namespace="portfolio", container!=""}[5m])) by (pod)
```

Without `sum`, you get one series per container. `sum(…) by (pod)` adds up every series that shares the same `pod` label and throws away the other labels, giving one line per pod. Other aggregations work the same way: `max`, `avg`, `count`, `topk(5, …)`.

### 5. Arithmetic between vectors

```promql
sum(container_memory_working_set_bytes{…}) by (pod, container)
  /
sum(kube_pod_container_resource_limits{…, resource="memory"}) by (pod, container)
```

Dividing two vectors divides series **with matching label sets**. Both sides are aggregated `by (pod, container)` so the labels line up, and the result is "usage as a fraction of limit" per container. If the labels don't match on both sides, you get an empty result, which is the most common PromQL mistake.

### The three metric types, in practice

| Type | Example metric | How to query it |
|---|---|---|
| Gauge | `container_memory_working_set_bytes`, `node_memory_MemAvailable_bytes` | Use directly; aggregate with `sum` or `max` |
| Counter | `container_cpu_usage_seconds_total`, `traefik_router_requests_total`, `kube_pod_container_status_restarts_total` | Always wrap in `rate()` or `increase()` first |
| Histogram | `traefik_service_request_duration_seconds_bucket` | `histogram_quantile(0.95, sum(rate(…_bucket[5m])) by (le, …))` |

A histogram is a set of counters, one per bucket (`le="0.1"` means "requests that took 0.1 seconds or less"). `histogram_quantile` interpolates a percentile from those bucket counts, which is why the `le` label must be kept in the `by (…)`.

### Where the metric names come from

| Prefix | Source | What it describes |
|---|---|---|
| `container_…` | the kubelet (cAdvisor built in) | Resource usage of each container |
| `kube_…` | kube-state-metrics | The declared state of Kubernetes objects: limits, replicas, restarts, phases |
| `node_…` | node-exporter | The server itself |
| `traefik_…` | Traefik, once scraped in section 3 | Requests, errors, latency per router and service |

So "how much memory is the pod using" is a `container_` question, "what limit did I give it" is a `kube_` question, and "is the disk filling" is a `node_` question.

## Background: how K3s lets you change its bundled Traefik

Traefik on K3s wasn't installed with `helm install`; K3s installed it from a chart it carries, through its own **Helm controller**. That controller watches `/var/lib/rancher/k3s/server/manifests/` on the server and applies anything placed there, and it understands a special object, **`HelmChartConfig`**, as "here are extra values for the chart with this name".

```text
/var/lib/rancher/k3s/server/manifests/
  traefik.yaml          ← K3s's own HelmChart object, managed by K3s, don't edit
  traefik-config.yaml   ← your HelmChartConfig: valuesContent merged on top     ◄── you add this
                                     │
                                     ▼
                      K3s Helm controller re-renders the Traefik chart with your values
                                     │
                                     ▼
                      helm-install-traefik Job runs, Traefik Deployment rolls
```

The reason to do it this way rather than editing Traefik's Deployment directly is that K3s re-applies its bundled chart on every K3s upgrade, which would wipe a hand edit. A `HelmChartConfig` survives upgrades because it's an input to that process, not a change to its output. The same mechanism would let you change Traefik's ports, enable its dashboard, or add arguments.

The values in section 3 do two things: `service.enabled: true` makes Traefik expose its `/metrics` endpoint as a Service called `traefik-metrics`, and `serviceMonitor.enabled: true` creates the `ServiceMonitor` that tells the Prometheus Operator to scrape it. That second part only works because step 18's values file set `serviceMonitorSelectorNilUsesHelmValues: false`, allowing ServiceMonitors from any namespace.

## Background: sizing requests and limits from data

Step 13 explained what requests and limits are. The method for choosing them, once you have data:

| Setting | Set it to | Why |
|---|---|---|
| `requests.memory` | Roughly the **typical** usage | What the scheduler reserves. Too high wastes the node; too low lets the scheduler overcommit |
| `limits.memory` | About **1.5× the peak** | The OOM ceiling. Judged at the peak, which for many apps is startup, as Grafana proved in step 18 |
| `requests.cpu` | Roughly the typical usage | Same role as memory requests, for scheduling |
| `limits.cpu` | **Usually none** | CPU is compressible: with no limit a busy container just shares the cores. A limit *throttles* it to the ceiling even when cores are idle, which makes things slower and protects nothing on a single-tenant box |

"Typical" and "peak" come from the 7-day view in Namespace (Pods). Seven days matters because weekly patterns exist and because a single evening's data hides the startup spike you only get once.

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

The chart ships about thirty dashboards. These three answer most questions; the rest are specialised views (per-node, per-workload, networking, the Prometheus server's own health) that become useful once you know what you're looking for.

### 2. PromQL you'll actually use

Open **`http://prometheus.home.arpa/graph`** or Grafana → **Explore**. Each query below is one line; replace `portfolio` with any namespace.

**Memory in use per pod (what the limit is compared against):**

```promql
sum(container_memory_working_set_bytes{namespace="portfolio", container!=""}) by (pod)
```

A gauge, summed per pod. This is the number `kubectl top pods` prints and the number the OOM killer compares against the limit.

**CPU per pod, in cores, averaged over 5 minutes:**

```promql
sum(rate(container_cpu_usage_seconds_total{namespace="portfolio", container!=""}[5m])) by (pod)
```

A counter, so `rate()` first, then summed per pod. `0.01` is ten millicores, which is what an idle nginx looks like.

**How close each container is to its memory limit (1.0 = about to be OOM-killed):**

```promql
sum(container_memory_working_set_bytes{namespace="portfolio", container!=""}) by (pod, container)
  /
sum(kube_pod_container_resource_limits{namespace="portfolio", resource="memory"}) by (pod, container)
```

Usage from the kubelet divided by the declared limit from kube-state-metrics, matched on `pod` and `container`. Anything above `0.8` deserves a bigger limit before it becomes a Challenges entry.

**Restarts in the last hour, cluster-wide:**

```promql
sum(increase(kube_pod_container_status_restarts_total[1h])) by (namespace, pod) > 0
```

`increase` over an hour on the restart counter, filtered to non-zero. An empty result is the good result.

**Free memory on the server as a fraction:**

```promql
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes
```

Two gauges from node-exporter. `MemAvailable` is the honest figure, including cache the kernel would give back; `MemFree` would understate it.

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

The three `add…Labels` lines make Traefik attach an `entrypoint`, `router` and `service` label to each metric, which is what lets you break requests down per site. (If you paste this and the shell sits at a `>` prompt, the closing `EOF` got indented; see the Challenges in [step 21](../10-dns/21-local-dns-adguard.md).)

K3s notices the file and re-runs the Traefik chart. Back on the **laptop**, watch it roll:

```bash
kubectl -n kube-system rollout status deploy/traefik
kubectl -n kube-system get svc traefik-metrics
kubectl -n kube-system get servicemonitor traefik
```

The first waits for the re-rendered Traefik to finish its rolling update. The second confirms the new metrics Service exists. The third confirms the ServiceMonitor the Operator will act on.

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

The `code=~"5.."` regex selects HTTP 5xx responses. Errors divided by everything gives the fraction, per backend service.

**p95 latency per service, in seconds:**

```promql
histogram_quantile(0.95, sum(rate(traefik_service_request_duration_seconds_bucket[5m])) by (le, service))
```

"95% of requests to each service completed within this many seconds." The `le` label must survive the `by (…)` or the function has no buckets to work with.

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

The early `kubectl top` reading from step 17 (portfolio pods at about 5Mi each against a 64Mi limit) suggests the first row will show a limit that could come down, not up. The table waits for the 7-day view all the same.

### 5. Alerts without Alertmanager

The chart installed a set of alert rules. **`http://prometheus.home.arpa/alerts`** shows them. `Watchdog` is **always firing** on purpose: it exists so a notification channel can prove it's alive. Anything else firing, especially `KubePodCrashLooping`, `KubeContainerWaiting`, or `NodeFilesystemSpaceFillingUp`, is worth reading. Wiring these to a phone is a later step, once Alertmanager comes back.

How to read that page:

| State | Meaning |
|---|---|
| **Inactive** | The rule's condition is false. The normal state for almost everything |
| **Pending** | The condition became true but hasn't held for the rule's `for:` duration yet. Most rules wait 5 to 15 minutes to avoid alerting on blips |
| **Firing** | The condition has held long enough. With Alertmanager this is when a notification would go out; without it, this page is the notification |

Each rule is a PromQL expression you can click through and run yourself. `KubePodCrashLooping`, for instance, is built on the same restart counter as the query in section 2. Alertmanager, when it comes back, adds only the routing: grouping, silencing, and sending to Slack, email or a phone.

## Challenges and solutions

| Challenge | What was going on | Solution |
|---|---|---|
| Gaps in every graph, and the 7-day view had less than a day of data | I had switched the server off overnight. Prometheus only records while it runs; there's no catching up. | Nothing to fix, but it changes the plan: the limits table in section 4 needs a few days of **continuous** uptime. Leave the box on. It idles at about 10W. |
| Not much else went wrong | Dashboards, PromQL queries, and the Traefik scrape all worked first time. | The Traefik `HelmChartConfig` keys were right for the chart K3s v1.36 bundles. |

*Limits table in section 4 still to be filled after a few days of uptime.*

Traps to expect when writing your own queries:

| Symptom | Usual cause |
|---|---|
| A graph that climbs forever | A counter plotted without `rate()` |
| Empty result from a division | The two sides have different label sets; aggregate both `by (…)` the same labels |
| `histogram_quantile` returns nothing | `le` was dropped from the `by (…)` |
| Numbers that don't match `kubectl top` | Querying `container_memory_usage_bytes` (includes cache) instead of `working_set_bytes` |
| Several lines per pod where one was expected | Missing `container!=""`, so the pod-level aggregate rows are included |

## Lessons

> 💡 **What I learned:**
> - **Three dashboards cover most questions.** Cluster for headroom, Namespace (Pods) for one app, Node Exporter for the box.
> - **`working_set_bytes` is the number that matters for memory.** It's what the OOM killer compares against the limit.
> - **Counters need `rate()`.** A raw `requests_total` graph only goes up. The rate is the thing you care about.
> - **Changing K3s's bundled Traefik is a file on the server, not a `helm upgrade`.** `HelmChartConfig` is K3s's hook for that, and it survives K3s upgrades.
> - **Limits are measurements, not guesses.** The first week of data usually shows the original numbers were off, in both directions.
> - **Every built-in panel is a PromQL lesson.** Open it in Explore and read the query before writing your own.
> - **Requests commitment is the scheduler's view, utilisation is reality.** Both have to fit for the next app to run well.

## Glossary

| Term | Meaning |
|---|---|
| Panel / variable / time range | A dashboard's building block / a dropdown substituted into every query / the window shown |
| Explore | Grafana's query editor; opens any panel's query for reading and editing |
| Instant vector / range vector | The latest value per series / the last N minutes of samples per series (`[5m]`) |
| `rate()` / `increase()` | Per-second change of a counter over a window / total change over a window |
| `sum by (…)` | Add up series that share the named labels, discard the rest |
| `histogram_quantile` | Compute a percentile from a histogram's `_bucket` series; needs `le` in the grouping |
| `le` | "less than or equal", the bucket boundary label on histogram series |
| Working set | Memory actively in use, excluding reclaimable cache; the OOM figure |
| Requests commitment / limits commitment | Sum of all pods' requests / limits as a share of the node |
| Compressible resource | One that can be shared by slowing everyone (CPU), as opposed to one that runs out (memory) |
| HelmChartConfig | K3s's object for overriding values of a bundled chart, placed in the server's manifests folder |
| ServiceMonitor | Tells the Prometheus Operator which Service's pods to scrape |
| Router / service (Traefik) | One router per Ingress host rule / one service per backend Service it forwards to |
| Alert states | Inactive, Pending (true but not yet for the `for:` duration), Firing |
| Watchdog | An alert that always fires so a notification pipeline can prove it works |
| Alertmanager | The component that routes firing alerts to channels; disabled in step 18 |

---

[◀ Previous: Monitoring: Prometheus and Grafana](18-prometheus-grafana.md) · [🏠 Home](../../README.md) · [Next: HTTPS Everywhere With cert-manager ▶](../09-tls/20-https-with-cert-manager.md)
