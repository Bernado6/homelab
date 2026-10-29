# 12. Installing Kubernetes (K3s)

One command, two minutes, and there is a Kubernetes cluster on the machine. That speed is the whole reason K3s was chosen in [step 4](../01-planning/04-budget-and-software-choices.md), and it is also a trap: it is easy to end up with a running cluster and no idea what the seven pods it started are for. This page installs K3s and then slows down to explain what just happened.

## What

Install K3s on the server over SSH, let the normal user run `kubectl` without `sudo`, and confirm that the node is `Ready` and the system pods are running. Then learn what each of those pods does, because every later step builds on one of them.

## Background: what Kubernetes is, in one page

Kubernetes is a system that runs **containers** across one or more machines and keeps them running the way you declared. You describe the desired state in YAML ("two copies of this image, with this much memory, reachable on port 80"), submit it to the cluster, and a set of controllers works continuously to make reality match the description. If a container dies, it is restarted. If a machine dies, its containers are rescheduled elsewhere (with more than one node). You never start a container by hand.

The pieces, and where they live:

```text
 CONTROL PLANE (the brain)                       NODE (where containers run)
 ─────────────────────────                       ──────────────────────────
 kube-apiserver   ── the front door; kubectl      kubelet      ── agent that starts and
                     and everything else talk                    watches containers on this
                     to this, over HTTPS :6443                   machine, as told by the API
 datastore        ── holds the cluster state      kube-proxy   ── programs the node's network
                     (etcd upstream;                               rules so Service IPs work
                      SQLite in K3s)              container    ── actually runs the containers
 kube-scheduler   ── decides which node each        runtime        (containerd)
                     new pod goes to              CNI plugin   ── gives each pod an IP and
 controller-      ── the loops that notice           (Flannel)     connects pods together
   manager           "desired ≠ actual" and act
```

On a single-node homelab the control plane and the node are the same machine, which is why `kubectl get nodes` below shows one entry with the role `control-plane`. Everything in both columns is running on the OptiPlex.

Two words that come up constantly:

| Word | Meaning |
|---|---|
| **Pod** | The smallest thing Kubernetes runs: one or more containers that share a network address and are scheduled together. Usually one container per pod. |
| **Namespace** | A named folder for objects. `kube-system` holds the cluster's own components; `default` is where your things go unless you say otherwise. |

## Background: what K3s is, and how it differs from upstream Kubernetes

K3s is a **distribution** of Kubernetes, not a fork: the API, the objects, `kubectl` and the YAML are the standard ones, which is why CKAD practice on it transfers to any cluster. What it changes is the packaging.

| | Upstream Kubernetes (kubeadm) | K3s |
|---|---|---|
| How it ships | Several binaries and container images, one per component | **One binary**, about 70MB, containing all of them |
| Datastore | **etcd**, a separate distributed database, its own cluster to run | **SQLite**, embedded, a single file at `/var/lib/rancher/k3s/server/db/`. Can use etcd when you add servers |
| Container runtime | You install one | **containerd** bundled and configured |
| Pod networking (CNI) | You choose and install one | **Flannel** built in |
| In-cluster DNS | You install CoreDNS | **CoreDNS** installed automatically |
| Ingress controller | You install one (nginx, Traefik, …) | **Traefik** installed automatically |
| `type: LoadBalancer` Services | Need a cloud provider or MetalLB | **ServiceLB** (klipper-lb) uses the node's own IP |
| Storage for volumes | You install a provisioner | **local-path** provisioner uses a folder on the node's disk |
| `kubectl top` | You install metrics-server | **metrics-server** installed automatically |
| How it is started | `systemd` units you set up | One `k3s.service` the installer creates |
| Install | An afternoon | `curl … \| sh -` |
| RAM at idle | More | Under 1GB |

Each bundled piece is a normal Kubernetes workload that K3s deploys for you at first start, from manifests it drops into `/var/lib/rancher/k3s/server/manifests/`. That is why they appear as pods in `kube-system` rather than as hidden magic, and why they can be inspected, restarted and replaced like anything else.

The name is a joke on the abbreviation: Kubernetes is "K8s" (K, eight letters, s), and something half the size would be "K3s" (K, three letters, s). There is no word the 3 stands for.

### What the install script does

`https://get.k3s.io` is a shell script. Piping a script from the internet straight into `sh` deserves a moment's thought: you are running whatever that URL serves, as root. The reasons it is acceptable here are that the URL is the project's official one, the script is short enough to read first (`curl -sfL https://get.k3s.io | less`), and the alternative, downloading the binary and writing the `systemd` unit yourself, is what the script does anyway. Read it once, then run it.

What it does, in order:

1. Detects the architecture (amd64 here) and downloads the matching `k3s` binary to `/usr/local/bin/k3s`.
2. Creates symlinks so that `kubectl`, `crictl` and `ctr` all point at that same binary. There is no separate `kubectl` install.
3. Writes `/etc/systemd/system/k3s.service` and enables it, so K3s starts at every boot.
4. Starts the service. On first start K3s generates its certificates and CA, initialises the SQLite datastore, starts the control plane and the kubelet, and applies the bundled manifests.
5. Writes the admin **kubeconfig** to `/etc/rancher/k3s/k3s.yaml`.

The `curl` flags: `-s` silent, `-f` fail on an HTTP error instead of piping an error page into `sh`, `-L` follow redirects.

## Background: what a kubeconfig is, and why we copy `k3s.yaml`

`kubectl` is an HTTPS client. Every command becomes a request to the API server on port 6443, and to make that request it needs three things: *where* the API server is, *how to verify* it is the real one, and *who* it is acting as. A **kubeconfig** is the YAML file that holds those three things: the server URL, the cluster's CA certificate, and a client certificate and key that identify the user.

K3s writes that file to `/etc/rancher/k3s/k3s.yaml`, owned by root with permissions `0600`. The client certificate in it is **cluster-admin**: whoever holds the file can do anything to the cluster. That is why it is locked down, and why `kubectl get nodes` as your normal user fails straight after the install: `kubectl` cannot read the file, falls back to looking for `~/.kube/config`, finds nothing, and tries `localhost:8080`, where nothing listens.

The ways to let a normal user run `kubectl`:

| Option | How | Trade-off |
|---|---|---|
| **Copy the file to `~/.kube/config` and own it** ✅ | The commands in the How | The standard location `kubectl` looks in. Your copy, your permissions, root's original untouched. Setting `KUBECONFIG` as well makes the choice explicit. What I did. |
| `sudo kubectl …` every time | Nothing to set up | Habit-forming in the wrong direction; every command runs as root |
| Install with `--write-kubeconfig-mode 644` | K3s makes the original world-readable | Any user on the machine becomes cluster-admin |
| `export KUBECONFIG=/etc/rancher/k3s/k3s.yaml` | Point at root's file | Still unreadable without `sudo`; same problem |

The copied file says the server is at `https://127.0.0.1:6443`, which is correct on the node. [Step 15](../06-working-with-the-cluster/15-kubectl-from-laptop.md) copies it again, to the laptop, and that is where the loopback address has to change.

## When

After [step 11](../03-ubuntu-server/11-first-login-and-ssh.md): you need an SSH session and an up-to-date system. Before anything else in the guide, because every later step is a `kubectl` command.

## How

All of this runs on the server, in the SSH session from step 11.

### 1. Install K3s

```bash
curl -sfL https://get.k3s.io | sh -
```

This downloads the install script and runs it (see the Background for exactly what it does). Expect a handful of `[INFO]` lines: finding the release, downloading the binary and its hash, verifying it, creating the symlinks and the `systemd` unit, and finally `[INFO]  systemd: Starting k3s`. The prompt returns in well under a minute; the cluster carries on starting in the background for another minute or two.

### 2. Let your normal user run `kubectl`

Let your normal user run `kubectl`:

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
source ~/.bashrc
```

Line by line:

| Command | What it does |
|---|---|
| `mkdir -p ~/.kube` | Creates the folder `kubectl` looks in. `-p` means no error if it already exists |
| `sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config` | Copies root's kubeconfig to the standard location. `sudo` because the source is unreadable otherwise |
| `sudo chown $USER:$USER ~/.kube/config` | The copy is still owned by root; this hands it to you. `$USER` expands to your username |
| `echo 'export KUBECONFIG=…' >> ~/.bashrc` | Appends one line to your shell's startup file so every future session knows where the config is. `>>` appends; a single `>` would overwrite the file |
| `source ~/.bashrc` | Re-reads the startup file now, so this session picks up the variable without logging out |

Only run the `echo` line once; running it twice adds a duplicate line, harmless but untidy.

### 3. Check

Check:

```bash
kubectl get nodes
kubectl get pods -A
```

`kubectl get nodes` lists the machines in the cluster: one. `-A` is short for `--all-namespaces`; without it you would see only the `default` namespace, which is empty, and wonder where everything is.

My result:

```text
NAME      STATUS   ROLES           AGE   VERSION
quantum   Ready    control-plane   71s   v1.36.5+k3s1
```

Reading it: the node is named after the server's hostname from step 10, `Ready` means the kubelet is reporting in and can run pods, `control-plane` is the role (and on one node, also the worker), and the version is upstream Kubernetes 1.36.5 packaged as K3s build 1. If `STATUS` says `NotReady` in the first minute, wait and run it again; it flips once networking is up.

`kubectl get pods -A` will show the `kube-system` namespace with roughly this shape (your hash suffixes and ages will differ):

```text
NAMESPACE     NAME                                      READY   STATUS      RESTARTS   AGE
kube-system   coredns-…                                 1/1     Running     0          …
kube-system   helm-install-traefik-crd-…                0/1     Completed   0          …
kube-system   helm-install-traefik-…                    0/1     Completed   1          …
kube-system   local-path-provisioner-…                  1/1     Running     0          …
kube-system   metrics-server-…                          1/1     Running     0          …
kube-system   svclb-traefik-…                           2/2     Running     0          …
kube-system   traefik-…                                 1/1     Running     0          …
```

Two statuses are correct: `Running` for the long-lived components and `Completed` for the two install jobs. `ContainerCreating` or `Pending` in the first couple of minutes is the images being pulled; re-run and it clears.

### 4. What each pod does

| Pod (kube-system) | What it does |
|---|---|
| `coredns` | DNS inside the cluster, so apps find each other by name |
| `traefik` + `svclb-traefik` | Ingress controller: receives web traffic on ports 80/443 |
| `local-path-provisioner` | Creates storage on the SSD when an app asks for a volume |
| `metrics-server` | Makes `kubectl top` work |
| `helm-install-traefik(-crd)` | One-off install jobs, so **Completed** is correct |

In more detail, with the step where each one first matters:

| Pod | What it really is | Where you meet it |
|---|---|---|
| `coredns` | The cluster's DNS server. Every Service gets a name like `hello.default.svc.cluster.local`, and pods resolve it through CoreDNS. Anything that is *not* a cluster name is forwarded to the node's own resolver. | [Step 13](13-first-app.md) uses Service names; [step 21](../10-dns/21-local-dns-adguard.md) is where the forwarding detail bites |
| `traefik` | The **Ingress controller**: a reverse proxy that watches Ingress objects and routes incoming HTTP by host and path to the right Service. | Every app from step 13 onwards |
| `svclb-traefik` | A **ServiceLB** pod. Traefik's Service is `type: LoadBalancer`, and K3s has no cloud to ask for an IP, so ServiceLB binds ports 80 and 443 on the node's own IP and forwards them to Traefik. Two containers, one per port, hence `2/2`. | This is why `http://<SERVER_IP>` reaches Traefik in step 13; the same mechanism publishes DNS in step 21 |
| `local-path-provisioner` | Watches for PersistentVolumeClaims and satisfies them by creating a directory under `/var/lib/rancher/k3s/storage/` on the node and mounting it into the pod. The `local-path` **StorageClass**. | [Step 18](../08-observability/18-prometheus-grafana.md) (Prometheus data), step 21 (AdGuard config) |
| `metrics-server` | Scrapes CPU and memory usage from the kubelet and serves it through the API, so `kubectl top nodes` and `kubectl top pods` work. | [Step 19](../08-observability/19-using-the-metrics.md) |
| `helm-install-traefik-crd`, `helm-install-traefik` | **Jobs**, not services. K3s installs Traefik as a Helm chart, and these two pods ran `helm install` once (CRDs first, then the chart) and exited. `Completed` means they succeeded. | Only here. They may restart once or twice while the API is still coming up; that is the job retrying, not an error |

Two more things run but do not appear as pods, because they are part of the K3s binary itself: **Flannel**, which gives every pod an IP in `10.42.0.0/16` and creates the `cni0` and `flannel.1` interfaces you will see in `hostname -I`, and **kube-proxy**, which programs the node so that Service IPs in `10.43.0.0/16` reach their pods. Both ranges come back in [step 13](13-first-app.md).

### 5. Useful things to know about the service

Not required, but worth trying once so they are familiar when something goes wrong:

```bash
sudo systemctl status k3s          # is the service running? recent log lines
sudo journalctl -u k3s -f          # follow the K3s log live; Ctrl+C to stop
kubectl get all -A                 # everything in every namespace, not just pods
```

A reboot of the server brings K3s and every pod back on its own; the `systemd` unit is enabled. If you ever want it gone, the installer also left `/usr/local/bin/k3s-uninstall.sh`, which removes the binary, the service and all cluster data.

## Challenges and solutions

Nothing failed for me on this step, beyond the restart count on the Traefik install job, which is normal. The traps:

| Challenge | What was going on | Solution |
|---|---|---|
| `helm-install-traefik` pod restarts after K3s install | Normal at first boot. The job tries to run `helm install` before the API is fully ready, fails, and is retried. | Wait. It shows **Completed** within a minute or two. See [Troubleshooting](../troubleshooting.md) |
| `kubectl get nodes` says `The connection to the server localhost:8080 was refused` | `kubectl` could not read a kubeconfig, so it tried a default that does not exist. Usually the copy steps were skipped, or this is a new shell without `KUBECONFIG` | Run the five commands in section 2; check `ls -l ~/.kube/config` is owned by you |
| `permission denied` reading `/etc/rancher/k3s/k3s.yaml` | The file is root-only by design | Copy it with `sudo`, then `chown`; do not loosen the original |
| Node `NotReady` for the first minute | Flannel and CoreDNS still starting | Wait and re-run `kubectl get nodes` |
| Pods stuck in `ContainerCreating` or `ImagePullBackOff` right after install | Images are being pulled from the internet; slow or no connectivity | Check `kubectl -n kube-system describe pod <name>` for the Events; confirm the server has internet with `curl -I https://get.k3s.io` |
| Ran the `echo … >> ~/.bashrc` line twice | Duplicate line in the file | Harmless; delete one with any editor if it bothers you |

## Lessons

> 💡 **What I learned:** K3s gives you a working cluster in about two minutes, with ingress and storage included. A couple of restarts on the `helm-install-traefik` job at first boot are normal; it retries until the cluster is ready.

> 💡 **Also:** every "bundled" piece is an ordinary workload in `kube-system`. Nothing about K3s is hidden; it just did the installing for you. Learning what those seven pods are is the fastest way to understand what a cluster *is*.

## Glossary

| Term | Meaning |
|---|---|
| Kubernetes | A system that runs containers across machines and keeps them matching a declared desired state |
| Control plane | The API server, scheduler, controller manager and datastore; the cluster's brain. On one node, the same machine as the worker |
| kube-apiserver | The HTTPS front door on port 6443 that `kubectl` and every component talk to |
| kubelet | The per-node agent that starts and watches containers |
| kube-proxy | Programs the node's network rules so Service IPs reach pods |
| etcd / SQLite | The cluster datastore; upstream uses etcd, single-node K3s uses an embedded SQLite file |
| containerd | The container runtime that actually runs the containers; bundled with K3s |
| CNI / Flannel | The plugin that gives pods IPs and connects them; K3s bundles Flannel |
| Pod | The smallest unit Kubernetes runs; one or more containers sharing an IP |
| Namespace | A named folder for objects; `kube-system` for the cluster's own, `default` for yours |
| K3s | A single-binary Kubernetes distribution with networking, ingress, storage and metrics bundled |
| Distribution | Same Kubernetes API, different packaging; what you learn transfers |
| kubeconfig | The YAML file holding the API server address, its CA, and a client certificate; `kubectl`'s credentials |
| `k3s.yaml` | K3s's root-owned admin kubeconfig at `/etc/rancher/k3s/` |
| `KUBECONFIG` | The environment variable telling `kubectl` which kubeconfig file to use |
| cluster-admin | The all-powerful role; what the client certificate in `k3s.yaml` carries |
| CoreDNS | The in-cluster DNS server for `*.cluster.local` names |
| Traefik | The bundled Ingress controller: routes HTTP by host and path |
| ServiceLB / klipper-lb / `svclb-*` | K3s's way of giving a `LoadBalancer` Service the node's own IP, via hostPort pods |
| local-path provisioner / StorageClass | Creates volumes as directories on the node's disk when a claim asks for one |
| metrics-server | Serves CPU and memory usage so `kubectl top` works |
| Helm / `helm-install-*` Jobs | Helm is a package manager for Kubernetes; K3s installs Traefik as a chart via one-off Jobs that end `Completed` |
| `systemd` / `k3s.service` | Ubuntu's service manager and the unit that starts K3s at boot |
| `10.42.0.0/16` / `10.43.0.0/16` | K3s's default pod and Service address ranges |

---

[◀ Previous: First Login and SSH](../03-ubuntu-server/11-first-login-and-ssh.md) · [🏠 Home](../../README.md) · [Next: My First App ▶](13-first-app.md)
