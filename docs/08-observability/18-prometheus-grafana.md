# 18. Monitoring: Prometheus and Grafana

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

### 2. Create the namespace and the Grafana admin secret

The admin password lives in a Kubernetes Secret, created by hand, never in git. The values file references it by name.

```bash
kubectl create namespace monitoring
kubectl -n monitoring create secret generic grafana-admin \
  --from-literal=admin-user=admin \
  --from-literal=admin-password="$(openssl rand -base64 18)"
```

Read it back when you need it:

```bash
kubectl -n monitoring get secret grafana-admin -o jsonpath='{.data.admin-password}' | base64 -d; echo
```

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

### 4. Install

From the homelab repo root:

```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring -f apps/monitoring/values.yaml
```

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

This is the single-node trade-off from [step 3](../01-planning/03-one-node-or-cluster.md): the volume is a folder on this one SSD. If the disk dies, the metrics go with it, which is fine for metrics and not fine for photos.

### 6. Names on the laptop, then open the UIs

```bash
sudo sh -c 'printf "192.168.100.22  grafana.home.arpa\n192.168.100.22  prometheus.home.arpa\n" >> /etc/hosts'
kubectl -n monitoring get ingress
```

- **`http://prometheus.home.arpa/targets`**: every scrape target should be **UP**. If something is DOWN here, that's where to start debugging.
- **`http://grafana.home.arpa`**: log in as `admin` with the password from step 2. **Dashboards → Kubernetes / Compute Resources / Cluster** is the one to open first.

### 7. Verify from the CLI

```bash
kubectl top nodes
kubectl top pods -A --sort-by=memory | head -15
```

`kubectl top` uses metrics-server, which K3s bundled in step 12 and has nothing to do with Prometheus, but it's the quickest sanity check that the numbers in Grafana are in the right ballpark.

### Upgrading or changing values later

Edit `values.yaml`, then:

```bash
helm upgrade monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring -f apps/monitoring/values.yaml
```

To see what Helm actually rendered, `helm get values monitoring -n monitoring` shows your overrides and `helm get manifest monitoring -n monitoring` shows every object it created.

### Uninstalling cleanly

```bash
helm uninstall monitoring -n monitoring
kubectl -n monitoring delete pvc --all      # Helm leaves volumes behind on purpose
kubectl delete crd $(kubectl get crd -o name | grep monitoring.coreos.com)
```

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

## Lessons

> 💡 **What I learned:**
> - **A Helm values file is the install.** The chart is the same for everyone; `values.yaml` is what makes it yours. Commit it, and `helm upgrade -f` keeps the cluster matching it.
> - **Turn off what K3s doesn't expose.** The default chart assumes kubeadm. On K3s, four of its scrape targets can never be UP. Disabling them is the difference between a clean targets page and permanent red.
> - **Storage is a request, not a mount.** The pod asks for 10Gi with a storage class, and the provisioner finds somewhere to put it. On a single node that somewhere is a folder on the SSD.
> - **Secrets are created, not committed.** The Grafana password went into the cluster with `kubectl create secret` and into the values file only by name.
> - **Memory limits are a budget.** Adding them up before installing told me the stack costs under 2GB worst case, leaving room on 8GB for the apps that come next.

---

[◀ Previous: First Real Workload: The Portfolio Site](../07-workloads/17-portfolio.md) · [🏠 Home](../../README.md) · [Next: Using the Metrics ▶](19-using-the-metrics.md)
