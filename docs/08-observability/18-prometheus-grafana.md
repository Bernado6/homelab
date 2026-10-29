# 18. Monitoring: Prometheus and Grafana

Until this step, "how is the server doing?" meant SSHing in and running `top`. This step installs the standard Kubernetes monitoring stack so the answer is a graph you can open from the phone. It's also the first Helm chart, the first persistent volume, and the first time a memory limit bites back, so the page spends a lot of time on what those things are before installing anything.

## What

Install the **kube-prometheus-stack** Helm chart into a `monitoring` namespace. It brings:

| Component | Job |
|---|---|
| **Prometheus** | Scrapes metrics every 30s from the node, the kubelet, every pod that exposes them, and stores 7 days of history on disk |
| **Grafana** | Dashboards on top of Prometheus. Comes with a set for Kubernetes already wired up |
| **Prometheus Operator** | Lets you add scrape targets with Kubernetes objects (`ServiceMonitor`) instead of editing a config file |
| **kube-state-metrics** | Turns Kubernetes objects into metrics: pod restarts, deployment replicas, PVC status |
| **node-exporter** | CPU, RAM, disk and network of the server itself |

Both UIs get a host-based Ingress, same pattern as steps 16 and 17: `http://grafana.home.arpa` and `http://prometheus.home.arpa`.

## Background: what "metrics" are

Observability has three kinds of signal, and they answer different questions:

| Signal | What it is | Question it answers | Tool in this homelab |
|---|---|---|---|
| **Metrics** | Numbers sampled over time: memory in use, requests per second, restarts | *Is it healthy? How much? Since when?* | Prometheus, this step |
| **Logs** | Lines of text an app writes as it runs | *What exactly happened at 02:14?* | `kubectl logs` for now |
| **Traces** | The path of one request across services | *Which hop was slow?* | Not needed on one node |

Metrics are cheap to store and easy to graph, so they're where monitoring starts. A metric is a **time series**: a name, a set of **labels**, and a value that changes over time.

```text
container_memory_working_set_bytes{namespace="portfolio", pod="portfolio-5ddbc576ff-79p8q", container="nginx"}  5242880
└──────────── name ────────────┘ └───────────────────────── labels ─────────────────────────────┘  └ value ┘
```

Labels are what make it useful: the same metric name exists for every container on the node, and the labels tell them apart. They're also the trap: every unique combination of labels is a separate series stored forever. A label with thousands of possible values (a user ID, a request path) multiplies the series count, which is called **cardinality**, and high cardinality is how Prometheus eats a machine. The defaults here are safe.

### Three kinds of metric

| Type | Behaviour | Example | How you read it |
|---|---|---|---|
| **Gauge** | Goes up and down | memory in use, temperature, number of pods | Read the value directly |
| **Counter** | Only ever increases, resets on restart | total requests served, total restarts | Never read it raw; take its **rate** over time |
| **Histogram** | Counts observations into buckets | request duration | Ask for a percentile, "95% of requests were faster than X" |

Step 19 turns this table into queries. For now, the point is that "requests: 48,201" is useless and "requests: 3 per second for the last five minutes" is the thing you want, and Prometheus does that conversion for you if you ask for `rate()`.

### Pull, not push

Most monitoring you may have met pushes: the app sends its numbers somewhere. Prometheus **pulls**. Every target exposes a plain-text page, usually `/metrics`, and Prometheus fetches it on a schedule (every 30 seconds here). This is called **scraping**.

```text
Prometheus ──GET /metrics every 30s──► kubelet            (container CPU, memory)
           ──GET /metrics every 30s──► node-exporter      (the server's CPU, RAM, disk, network)
           ──GET /metrics every 30s──► kube-state-metrics (pods, deployments, PVCs as numbers)
           ──GET /metrics every 30s──► any pod with a ServiceMonitor (Traefik in step 19)
```

Two consequences. First, something that doesn't expose `/metrics` can't be monitored until you give it an **exporter**, a small sidecar or service that translates for it; node-exporter is exactly that for the Linux kernel. Second, Prometheus itself must be able to reach every target, which is trivial on one node and the reason four targets below can never work.

## Background: the five components, and what each one is not

New people confuse these because the names overlap. Each one does one job.

| Component | Does | Does not |
|---|---|---|
| **Prometheus** | Scrapes, stores, and answers queries in its own language (PromQL). Has a basic UI at `/graph` and `/targets`. | Draw nice dashboards. Send notifications. |
| **Grafana** | Draws dashboards by sending queries to Prometheus. Stores dashboards and users in its own small database. | Store any metrics. If Prometheus is down, Grafana shows "No data". |
| **Prometheus Operator** | Watches for Kubernetes objects like `ServiceMonitor` and `PrometheusRule` and rewrites Prometheus's config to match. | Scrape anything itself. It's a controller, like the ones that manage Deployments. |
| **kube-state-metrics** | Asks the Kubernetes API "what objects exist and what state are they in?" and exposes the answer as metrics: `kube_pod_container_status_restarts_total`, `kube_deployment_status_replicas_available`. | Measure resource usage. It knows a pod *has* a 64Mi limit, not what it's *using*. |
| **node-exporter** | Reads `/proc` and `/sys` on the server and exposes the machine's own numbers: `node_memory_MemAvailable_bytes`, disk space, network bytes. | Know anything about Kubernetes. It would work on a laptop. |

And one that is **not** part of this stack but you already have:

| | metrics-server (bundled with K3s, step 12) | Prometheus |
|---|---|---|
| Purpose | Feeds `kubectl top` and autoscalers with *current* CPU and memory | Stores *history* and answers arbitrary queries |
| Retention | None, only the latest reading | 7 days here |
| Storage | None | A volume |

They don't talk to each other. `kubectl top` keeps working with or without Prometheus, which makes it a handy independent cross-check in section 7.

## Background: Helm, and why the values file is the install

Helm is the package manager for Kubernetes. The vocabulary:

| Term | Meaning | Here |
|---|---|---|
| **Chart** | A bundle of Kubernetes manifest *templates* plus a file of default settings | `kube-prometheus-stack`, maintained by the Prometheus community |
| **Repository** | A web index of charts | `prometheus-community`, added with `helm repo add` |
| **Values** | The settings you override. Anything you don't set keeps the chart's default | `apps/monitoring/values.yaml` |
| **Release** | One installed instance of a chart, with a name, in a namespace | `monitoring` in namespace `monitoring` |
| **Revision** | Each `helm upgrade` creates a new numbered revision; `helm rollback` goes back | Revision 3 on my box after the Grafana fix and the TLS change |

When you run `helm install`, Helm takes the templates, fills them in with the defaults plus your values, and applies the result. The same chart installs very differently depending on the values, which is why **the values file is the install**. Everything specific to your cluster lives there: the storage class, the resource limits, the Ingress hosts, which components are off. Commit it; it's the one file that reproduces the stack.

```bash
helm install <release> <repo>/<chart> -n <namespace> -f values.yaml    # first time
helm upgrade <release> <repo>/<chart> -n <namespace> -f values.yaml    # every change after
```

This chart has hundreds of settings. You'll never read them all; `helm show values prometheus-community/kube-prometheus-stack` prints them if you ever need to find one.

## Background: persistent storage

Everything so far has been stateless: a pod dies, a new one starts, nothing is lost because nothing was kept. Prometheus keeps seven days of data, and Grafana keeps your saved dashboards and the admin password. If those lived inside the container's filesystem, every restart would erase them.

Kubernetes separates "I need a disk" from "here is a disk":

| Object | Who creates it | What it is |
|---|---|---|
| **PersistentVolumeClaim (PVC)** | The app, or the chart on its behalf | A request: "I need 10Gi, read-write by one pod at a time, from storage class X" |
| **StorageClass** | The cluster admin, or K3s on install | A recipe for making volumes. `local-path` means "a directory on the node's disk" |
| **PersistentVolume (PV)** | The provisioner, automatically | The actual volume, created to satisfy the claim and **bound** to it |

```text
Prometheus pod ──mounts──► PVC "prometheus-db-…" ──bound to──► PV "pvc-5215…" ──is──► /var/lib/rancher/k3s/storage/pvc-5215…/ on the SSD
```

**Access mode.** `ReadWriteOnce` means one node can mount the volume read-write at a time. On a single node that's never a problem for running, but it is for *updating*: a rolling update starts the new pod before stopping the old one, and the new pod can't mount a volume the old one still holds. That's why the values file sets Grafana's `deploymentStrategy` to `Recreate`, which stops the old pod first.

**StatefulSet vs Deployment.** Prometheus shows up as `prometheus-…-0` rather than with a random suffix because the Operator runs it as a **StatefulSet**: pods get stable names and each name owns a specific PVC. A Deployment's pods are interchangeable; a StatefulSet's pods are not. Databases and anything that owns data use StatefulSets.

**What local-path means for you.** The volume is a folder on the one SSD. It survives pod restarts and reboots. It does not survive the disk dying, and it can't follow a pod to another node, which is fine with one node. Metrics are disposable; the apps parked on the roadmap that hold documents and photos are the ones that will need a backup plan before they get a volume.

## Background: memory limits, and what OOMKilled means

Every container here gets a memory **limit**. The kernel enforces it through **cgroups**: a container's processes are placed in a group with a ceiling, and when the group's usage reaches the ceiling, the kernel's OOM killer terminates the biggest process in it. Kubernetes reports that as `Reason: OOMKilled, Exit Code: 137` (128 plus signal 9, SIGKILL) and restarts the container, with an exponential back-off that produces `CrashLoopBackOff` if it keeps happening.

The number compared against the limit is the **working set**: memory the process is actively using, excluding cache the kernel could reclaim. Prometheus records it as `container_memory_working_set_bytes`, and `kubectl top` shows the same figure. A limit that's lower than an app's startup peak, not its steady state, produces a loop that looks like a broken app and is actually a budget. That's exactly what happened in Challenges.

## Why

| Reason | Detail |
|---|---|
| **You can't tune what you can't see** | Every manifest so far has `requests` and `limits` that were guesses. Step 19 replaces them with numbers. |
| **8GB is tight** | Knowing what's actually using memory decides when the RAM upgrade is really needed. |
| **It's on the goals list** | Prometheus + Grafana was one of the original self-hosting goals in [step 1](../01-planning/01-goals-decide-specs.md). |
| **First stateful workload** | Prometheus needs a disk. This is the first use of the `local-path` storage class K3s bundled in [step 12](../04-kubernetes/12-installing-k3s.md), and the first PersistentVolumeClaim. |
| **First Helm chart** | Helm is how most third-party software is installed on Kubernetes. Worth learning on something you'll keep. |

### Why this chart and not plain Prometheus + Grafana?

Installing the two separately means writing scrape configs and dashboards by hand. kube-prometheus-stack ships both pre-wired for Kubernetes, which is exactly the thing you want to look at. The cost is that it's a large chart with many knobs, so the values file matters.

| Option | What you get | What you do yourself | Verdict |
|---|---|---|---|
| **kube-prometheus-stack** ✅ | Prometheus, Grafana, Operator, exporters, thirty dashboards, alert rules, all wired | Write a values file | One install, a working view of the cluster in minutes |
| `prometheus` + `grafana` charts separately | The two apps | Scrape configs, dashboards, datasource, exporters | Lighter, but the wiring is the learning-heavy part and you'd redo what the stack chart already did |
| Prometheus Operator alone | The controller | Everything else | For people building their own stack |
| A hosted service (Grafana Cloud, Datadog) | Dashboards in someone else's browser | Install an agent | Free tiers exist; defeats the point of a homelab |

### What it costs on an 8GB box

| Component | Memory limit in `values.yaml` | Typical use |
|---|---|---|
| Prometheus | 1Gi | 360Mi after an hour, grows with retention |
| Grafana | 512Mi | about 200Mi steady, **390Mi at startup** |
| Operator | 256Mi | 22Mi |
| kube-state-metrics | 128Mi | 20Mi |
| node-exporter | 64Mi | 8Mi |
| **Total** | **under 2Gi worst case** | **around 800Mi** |

The "typical" column is what `kubectl top pods -n monitoring` showed on my box an hour after install. Grafana started life with a 256Mi limit and got OOM-killed in a loop; see Challenges below.

Alertmanager is turned off to save another 100Mi. Prometheus still evaluates every alert rule and shows them in its own UI; Alertmanager only adds routing to Slack, email, and so on, which needs a channel we don't have yet.

### Why four scrape targets are switched off

The chart was written for clusters built with kubeadm, where the control-plane components run as separate pods with metrics ports. K3s compiles them into one binary and doesn't expose those ports:

| Target the chart expects | On K3s | Left enabled |
|---|---|---|
| `kube-controller-manager` on 10257 | Inside the k3s process, port not exposed | Permanently DOWN, alert `KubeControllerManagerDown` firing forever |
| `kube-scheduler` on 10259 | Same | Same |
| `kube-proxy` on 10249 | Same | Same |
| `etcd` on 2379 | **Doesn't exist.** K3s uses SQLite unless you add servers | Same |

The values file sets all four to `enabled: false`. The metrics you lose are internal scheduler and controller timings you'd never look at on one node.

## How the pieces fit

```text
 laptop browser
   │ https://grafana.home.arpa            https://prometheus.home.arpa
   ▼                                        ▼
 Traefik ──► Service monitoring-grafana ──► Grafana pod ──PromQL──► Service …-prometheus ──► Prometheus pod (StatefulSet, 10Gi PVC)
                                              │ 2Gi PVC                                        │ scrapes every 30s
                                              │ (dashboards, users)                            ├──► kubelet            container CPU/memory
                                                                                               ├──► node-exporter      server CPU/RAM/disk
                                                                                               ├──► kube-state-metrics objects as numbers
                                                                                               └──► anything with a ServiceMonitor
                                                                      Prometheus Operator ──writes config for──► Prometheus pod
```

## When

After the portfolio ([step 17](../07-workloads/17-portfolio.md)), so there's something interesting to measure. Before adding any heavier app, so you can watch what it does to the box.

## How

### 1. Install Helm on the laptop

```bash
brew install helm
helm version --short
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

`brew install helm` puts the client on the Mac; there's nothing to install in the cluster. `helm repo add` registers the community chart index under a short name, and `helm repo update` fetches its current list, the same way `apt update` refreshes a package list. Expect `"prometheus-community" has been added` and `Update Complete. ⎈Happy Helming!⎈`.

### 2. Create the namespace and the Grafana admin secret

The admin password lives in a Kubernetes Secret, created by hand, never in git. The values file references it by name.

```bash
kubectl create namespace monitoring
kubectl -n monitoring create secret generic grafana-admin \
  --from-literal=admin-user=admin \
  --from-literal=admin-password="$(openssl rand -base64 18)"
```

`openssl rand -base64 18` generates 18 random bytes and prints them as 24 printable characters, so the password is strong and you never typed it. The `$( … )` runs that first and pastes the result into the command. The Secret has two keys, `admin-user` and `admin-password`, which is the shape the chart's `grafana.admin.existingSecret` setting expects.

Read it back when you need it:

```bash
kubectl -n monitoring get secret grafana-admin -o jsonpath='{.data.admin-password}' | base64 -d; echo
```

Secrets are stored base64-encoded, which is an encoding, not encryption, hence the `base64 -d` to read it. The trailing `echo` just adds the newline the password lacks.

### 3. Read the values file

[`apps/monitoring/values.yaml`](../../apps/monitoring/values.yaml). The things that matter:

| Block | What it does |
|---|---|
| `kubeControllerManager`, `kubeScheduler`, `kubeProxy`, `kubeEtcd` all `enabled: false` | K3s runs these inside one binary and doesn't expose their metrics ports. Left on, they show as permanently **DOWN** targets and fire alerts forever. K3s has no etcd at all by default. |
| `prometheusSpec.storageSpec` with `storageClassName: local-path`, `10Gi` | Asks K3s for a volume on the server's SSD. Without this, Prometheus uses an `emptyDir` and **loses all history on every restart**. |
| `retention: 7d`, `retentionSize: 8GB` | Whichever comes first. Keeps the 10Gi volume from filling. |
| `grafana.persistence` 2Gi + `deploymentStrategy: Recreate` | Saved dashboards survive restarts. `Recreate` because a `ReadWriteOnce` volume can't be mounted by the old and new pod at the same time during an upgrade. |
| `serviceMonitorSelectorNilUsesHelmValues: false` | Lets Prometheus pick up `ServiceMonitor` objects from any namespace. Step 19 needs this for Traefik. |
| Two `ingress:` blocks | Host-based Ingresses for both UIs, through Traefik. |
| `resources:` everywhere | The budget table above. |

Two more lines worth knowing about: `scrapeInterval: 30s` is how often every target is polled (the chart's default is also 30s; 15s doubles the storage for little gain at home), and `grafana.admin.existingSecret: grafana-admin` is the by-name reference to the Secret from section 2.

### 4. Install

From the homelab repo root:

```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring -f apps/monitoring/values.yaml
```

Helm renders the templates with your values and applies a few hundred objects: Deployments, a StatefulSet, Services, ConfigMaps holding the dashboards, the Operator's CRDs, PrometheusRules, ServiceMonitors. It prints a NOTES block ending in *Happy Helming*. The install returns before the pods are ready.

Then watch it come up. Prometheus takes the longest because it has to wait for its volume:

```bash
kubectl -n monitoring get pods -w
```

Expected within two or three minutes:

```text
NAME                                                     READY   STATUS    RESTARTS   AGE
monitoring-grafana-xxxxxxxxxx-xxxxx                      3/3     Running   0          2m
monitoring-kube-prometheus-operator-xxxxxxxxxx-xxxxx     1/1     Running   0          2m
monitoring-kube-state-metrics-xxxxxxxxxx-xxxxx           1/1     Running   0          2m
monitoring-prometheus-node-exporter-xxxxx                1/1     Running   0          2m
prometheus-monitoring-kube-prometheus-prometheus-0       2/2     Running   0          2m
```

Grafana shows `3/3` because it runs two sidecars that load dashboards and datasources from ConfigMaps. Prometheus is a **StatefulSet** (the `-0` suffix), which is what Kubernetes uses for pods that own a specific volume.

The `2/2` on Prometheus is the Prometheus container plus a `config-reloader` sidecar that the Operator uses to apply config changes without a restart. node-exporter runs as a **DaemonSet**, one pod per node, which on one node is one pod.

### 5. Check the storage

```bash
kubectl -n monitoring get pvc
```

```text
NAME                                                                                                     STATUS   VOLUME         CAPACITY   STORAGECLASS   AGE
monitoring-grafana                                                                                       Bound    pvc-...        2Gi        local-path     3m
prometheus-monitoring-kube-prometheus-prometheus-db-prometheus-monitoring-kube-prometheus-prometheus-0   Bound    pvc-...        10Gi       local-path     3m
```

`Bound` means the local-path provisioner created a directory on the server and handed it to the pod. On the server, the data is under:

```bash
sudo ls /var/lib/rancher/k3s/storage/
```

Each directory there is named after its PV and contains the raw files: Prometheus's time-series blocks in one, Grafana's SQLite database in the other. `Pending` instead of `Bound` would mean the provisioner couldn't create the directory, which on local-path almost always means a typo in the storage class name.

This is the single-node trade-off from [step 3](../01-planning/03-one-node-or-cluster.md): the volume is a folder on this one SSD. If the disk dies, the metrics go with it, which is fine for metrics and not fine for photos.

### 6. Names on the laptop, then open the UIs

```bash
sudo sh -c 'printf "192.168.100.22  grafana.home.arpa\n192.168.100.22  prometheus.home.arpa\n" >> /etc/hosts'
kubectl -n monitoring get ingress
```

Two Ingresses, both with `ADDRESS 192.168.100.22`, which means Traefik has picked up the rules. (From [step 21](../10-dns/21-local-dns-adguard.md) onwards the hosts lines aren't needed; the wildcard DNS rewrite covers these names.)

- **`http://prometheus.home.arpa/targets`**: every scrape target should be **UP**. If something is DOWN here, that's where to start debugging.
- **`http://grafana.home.arpa`**: log in as `admin` with the password from step 2. **Dashboards → Kubernetes / Compute Resources / Cluster** is the one to open first.

The targets page is the most honest page in the stack. Each row is one `/metrics` endpoint, the last scrape time, and any error. A clean page on K3s has the kubelet, node-exporter, kube-state-metrics, the Operator, Prometheus itself, CoreDNS and the API server, all UP, and none of the four disabled components listed at all.

### 7. Verify from the CLI

```bash
kubectl top nodes
kubectl top pods -A --sort-by=memory | head -15
```

`kubectl top` uses metrics-server, which K3s bundled in step 12 and has nothing to do with Prometheus, but it's the quickest sanity check that the numbers in Grafana are in the right ballpark.

If the two disagree wildly, something is misconfigured; if they roughly agree, trust Grafana for history and `kubectl top` for right now.

### Upgrading or changing values later

Edit `values.yaml`, then:

```bash
helm upgrade monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring -f apps/monitoring/values.yaml
```

Helm diffs the rendered output against what's in the cluster and changes only what differs. A new revision number appears in the output; `helm history monitoring -n monitoring` lists them and `helm rollback monitoring <n> -n monitoring` returns to one.

To see what Helm actually rendered, `helm get values monitoring -n monitoring` shows your overrides and `helm get manifest monitoring -n monitoring` shows every object it created.

### Uninstalling cleanly

```bash
helm uninstall monitoring -n monitoring
kubectl -n monitoring delete pvc --all      # Helm leaves volumes behind on purpose
kubectl delete crd $(kubectl get crd -o name | grep monitoring.coreos.com)
```

The PVCs are kept so an accidental uninstall doesn't destroy seven days of data. The CRDs (the `ServiceMonitor`, `PrometheusRule` and similar object types) are cluster-wide and Helm never deletes those either.

## Challenges and solutions

| Challenge | What was going on | Solution |
|---|---|---|
| Grafana pod `2/3 CrashLoopBackOff`, restart count climbing every two minutes. Dashboards loaded once, then flipped to **No data** and a *no available server* toast. | `kubectl describe pod` showed `Last State: Terminated, Reason: OOMKilled, Exit Code: 137` on the `grafana` container. `kubectl top pods` showed the pod at **387Mi** against my **256Mi** limit. Grafana provisions about thirty bundled dashboards at startup and spikes well above its steady-state memory while doing it. The other two containers in the pod are sidecars and were fine, hence `2/3`. | Raised `grafana.resources.limits.memory` to **512Mi** (request 192Mi) in `values.yaml` and ran `helm upgrade`. The "No data" in the UI was not a Prometheus problem: the browser tab was talking to a Grafana that kept dying under it. |

The diagnostic path is worth remembering, because it's the same for every crash-looping pod:

```bash
kubectl -n <ns> get pods                                   # which pod, how many containers ready
kubectl -n <ns> describe pod <pod> | grep -A6 "Last State" # why the last run ended
kubectl -n <ns> logs <pod> -c <container> --previous       # what it said before it died
kubectl top pods -n <ns>                                   # is it anywhere near its limit
```

`OOMKilled` + `137` means the limit, not the app. `Exit Code 1` with an error in `--previous` logs means the app.

What the incident teaches beyond the fix: a memory limit is compared against the **peak**, not the average, and the peak for many apps is startup. The budget table's "typical" column was right about steady state and wrong about the first ninety seconds. When sizing a limit, watch the first minute of a fresh pod with `kubectl top pods -w`, not an hour later.

## Lessons

> 💡 **What I learned:**
> - **A Helm values file is the install.** The chart is the same for everyone; `values.yaml` is what makes it yours. Commit it, and `helm upgrade -f` keeps the cluster matching it.
> - **Turn off what K3s doesn't expose.** The default chart assumes kubeadm. On K3s, four of its scrape targets can never be UP. Disabling them is the difference between a clean targets page and permanent red.
> - **Storage is a request, not a mount.** The pod asks for 10Gi with a storage class, and the provisioner finds somewhere to put it. On a single node that somewhere is a folder on the SSD.
> - **Secrets are created, not committed.** The Grafana password went into the cluster with `kubectl create secret` and into the values file only by name.
> - **Memory limits are a budget.** Adding them up before installing told me the stack costs under 2GB worst case, leaving room on 8GB for the apps that come next.
> - **Pull means the targets page tells the truth.** If Prometheus can't reach something, it says so there, with the error. Start every "why is there no data" question on `/targets`.
> - **Limits are judged at the peak.** Grafana's steady 200Mi was never the problem; its 390Mi startup was.

## Glossary

| Term | Meaning |
|---|---|
| Metric / time series | A named number with labels, sampled repeatedly over time |
| Label | A key-value tag that distinguishes series with the same name (`pod`, `namespace`, `container`) |
| Cardinality | The number of distinct label combinations; high cardinality is what makes Prometheus expensive |
| Gauge / counter / histogram | A value that varies / a value that only increases / a set of buckets for distributions |
| Scrape | Prometheus fetching a target's `/metrics` page on a schedule |
| Exporter | A helper that exposes `/metrics` on behalf of something that can't (node-exporter for Linux) |
| ServiceMonitor | A Kubernetes object telling the Operator "scrape the pods behind this Service" |
| Prometheus Operator | The controller that turns ServiceMonitors and PrometheusRules into Prometheus configuration |
| kube-state-metrics | Exposes the state of Kubernetes objects as metrics |
| metrics-server | K3s's bundled source for `kubectl top`; current readings only, no history |
| Helm chart / release / values / revision | A templated package / one installed instance / your overrides / each numbered upgrade |
| PVC / PV / StorageClass | A request for storage / the volume that satisfies it / the recipe used to make it |
| local-path | K3s's provisioner: a directory under `/var/lib/rancher/k3s/storage/` on the node |
| ReadWriteOnce | Mountable read-write by one node at a time; the reason Grafana uses `Recreate` |
| StatefulSet | A workload whose pods have stable names and their own volumes (Prometheus) |
| DaemonSet | One pod per node (node-exporter) |
| Retention | How long Prometheus keeps data, by time (`7d`) or size (`8GB`), whichever first |
| cgroup | The kernel mechanism that enforces a container's memory limit |
| Working set | The memory figure compared against the limit; `container_memory_working_set_bytes` |
| OOMKilled / 137 | The kernel killed the container for exceeding its memory limit |
| CrashLoopBackOff | Kubernetes restarting a container repeatedly with growing delays |

---

[◀ Previous: First Real Workload: The Portfolio Site](../07-workloads/17-portfolio.md) · [🏠 Home](../../README.md) · [Next: Using the Metrics ▶](19-using-the-metrics.md)
