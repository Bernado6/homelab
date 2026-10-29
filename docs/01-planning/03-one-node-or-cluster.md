# 3. One Node or a Cluster?

Every Kubernetes diagram shows several machines. Every "homelab" photo on the internet shows a shelf of them. So the natural assumption is that learning Kubernetes means buying at least three computers. This page is about why that assumption is usually wrong for a first homelab, and when it is right.

## What

Decide how many machines to buy: one node that does everything, or three (or more) that form a cluster. The answer follows from which exam or job you are preparing for ([step 1](01-goals-decide-specs.md)) and how much you want to spend ([step 4](04-budget-and-software-choices.md)).

## Background: what a node is, and what a cluster adds

A **node** is one machine that runs containers for Kubernetes. It runs the **kubelet** (the agent that takes orders) and a container runtime (the thing that actually starts containers). A **cluster** is a **control plane** plus one or more nodes. The control plane holds the desired state ("three copies of the portfolio, one of Grafana") and the **scheduler** picks a node for each pod.

```text
 ONE NODE                              THREE NODES
 ┌──────────────────────────┐          ┌────────────┐ ┌────────────┐ ┌────────────┐
 │ control plane            │          │ control pl.│ │ control pl.│ │ control pl.│
 │ + kubelet + runtime      │          │ kubelet    │ │ kubelet    │ │ kubelet    │
 │ all pods run here        │          │ some pods  │ │ some pods  │ │ some pods  │
 └──────────────────────────┘          └────────────┘ └────────────┘ └────────────┘
 the machine is the cluster            the scheduler spreads pods across machines
```

On one node, the scheduler's choice is always the same, so you never see scheduling do anything interesting. What you *do* see is everything else: how a Deployment keeps pods alive, how a Service finds them, how an Ingress routes to them, how a volume persists, how probes decide readiness. That list is the CKAD syllabus.

With three nodes, the scheduler has choices, and failure becomes visible. Pull the power on one machine and watch its pods get recreated on the other two. That is what "high availability" looks like, and it is what the CKA syllabus is about.

## Background: high availability and why the number is three

**High availability (HA)** means the service keeps working when a component fails. For a Kubernetes cluster, the component that matters most is the control plane's database, which stores the entire state of the cluster. In upstream Kubernetes that database is **etcd**; a single-server K3s uses an embedded **SQLite** file instead, which is lighter and one reason K3s fits in so little RAM.

To survive the loss of a machine, the database has to run on several machines that agree with each other, and agreement uses a **quorum**: a strict majority must be reachable.

| Database copies | Majority needed | Machines you can lose |
|---|---|---|
| 1 | 1 | 0 |
| 2 | 2 | 0 (losing one leaves no majority, which is worse than one copy) |
| 3 | 2 | 1 |
| 5 | 3 | 2 |

That table is why HA clusters come in threes and fives, never twos. A "2-node cluster" has two workers' worth of capacity and one node's worth of availability.

## Background: why a cluster needs shared storage

This is the cost people miss. On one node, a pod's data lives in a folder on that node's disk (K3s's **local-path** storage class, which the Prometheus volume uses in [step 18](../08-observability/18-prometheus-grafana.md)). On three nodes, the scheduler may restart that pod on a *different* machine, and the folder is not there. So a real cluster needs storage that every node can reach:

| Option | How it works | What it costs you |
|---|---|---|
| **NFS** | One machine shares a folder over the network; every node mounts it | Simple, but that one machine is now a single point of failure, which defeats the purpose |
| **Longhorn** | Replicates each volume's blocks across several nodes' disks | Proper HA storage, but another system to learn, and it wants RAM and CPU on every node |
| **Ceph (via Rook)** | Distributed storage designed for data centres | Powerful and heavy; overkill below five nodes |

Add to that the networking: pods on different machines talk through an **overlay network** (K3s uses Flannel), you need a switch with enough ports, a cable and a DHCP reservation per machine, and three machines' worth of OS updates. None of it is hard. All of it is time and RAM spent on *operating* the cluster rather than *using* it.

## Background: the middle path, virtual machines

Step 1 mentioned "one big machine running VMs". A **hypervisor** such as **Proxmox** turns one physical machine into several virtual ones, each of which can be a Kubernetes node. You get real scheduling, real node failures (stop a VM) and real multi-node storage practice, on one power cable. The costs are RAM, because each VM needs its own operating system's share before Kubernetes gets any (hence "64GB" in step 1), and the fact that a VM failing is not quite the same as a cable being pulled. It is the cheapest honest way to practise CKA, and it is a later option for this box if it ever gets enough memory.

## Side by side

| | One powerful node | 3-node cluster |
|---|---|---|
| ✅ Pros | Simple; all RAM in one place; one machine to manage | High availability; feels like production; teaches scheduling and node failures |
| ❌ Cons | If it's off, everything is off | Needs shared storage (Longhorn/NFS); more networking; RAM split across machines |
| 💰 Cost | One machine | Three machines |
| 🎯 Best for | CKAD, self-hosting, first homelab | CKA, high-availability practice |

Two of those cells deserve a closer look.

**"All RAM in one place."** 24GB in one machine runs one app that needs 20GB. 3 × 8GB does not, because no single node can hold it. For self-hosting, where one app (Immich, Nextcloud) can be the heavy one, concentration beats distribution.

**"If it's off, everything is off."** True, and worth accepting with open eyes. In this homelab, once [step 21](../10-dns/21-local-dns-adguard.md) makes the server the house's DNS, a reboot briefly affects every phone in the building. On one node you learn to make reboots rare and fast instead of invisible, and a power-cut setting in the BIOS ([step 9](../03-ubuntu-server/09-bios-settings.md)) does most of the work.

## Why

**My choice:** one node, with the option to add more later. K3s makes joining extra machines easy.

Against my goals:

| Goal | One node | Three nodes |
|---|---|---|
| CKAD practice | Covers the whole syllabus | Covers it too, plus things not on it |
| Self-hosting apps that each want a lot of RAM | All the RAM is available to the heavy app | RAM split three ways |
| KSh 40,000 | One decent machine | Three weak ones, or over budget |
| Learning on day one | Install K3s, run a pod, done | Storage and networking to set up before the first useful app |
| Grow later | Add a node when there is a reason to | Already there |

### How K3s makes joining extra machines easy

I have not done this yet, but it is the reason "one node now" is not a dead end. K3s has two roles: a **server** (runs the control plane and, by default, also runs pods) and an **agent** (runs pods only). The machine from this guide is a server. To add a second machine later, the general procedure from the K3s documentation is:

```bash
# On the existing server: print the join token that new nodes must present
sudo cat /var/lib/rancher/k3s/server/node-token

# On the new machine: install K3s as an agent, pointing it at the server
curl -sfL https://get.k3s.io | K3S_URL=https://<server-ip>:6443 K3S_TOKEN=<token> sh -
```

The first command prints a long string; the second downloads K3s, installs it as a system service, and registers the machine with the server. A minute later `kubectl get nodes` on the laptop shows two nodes. Shared storage is still your problem at that point, but the cluster itself is a two-command upgrade. That is the escape hatch that made one node an easy decision.

## When

After step 2 has settled the kind of machine and before step 4 sets the budget, because the number of machines is the biggest multiplier on the price.

## Challenges and solutions

No cluster was built on this page, so nothing failed. The traps are in the reasoning:

| Trap | What happens | How to avoid it |
|---|---|---|
| Buying three machines to "learn Kubernetes" for CKAD | Three times the cost, weeks on storage and networking, and the syllabus still fits on one node | Match the exam to the node count ([step 1](01-goals-decide-specs.md)) |
| Building a two-node "HA" cluster | Two copies of the state cannot form a majority; it is less available than one | Three or five, or one |
| Forgetting shared storage until the first pod moves | The pod restarts on another node and its data is gone | Decide on Longhorn or NFS before the first PersistentVolumeClaim on a multi-node cluster |
| Splitting a small RAM budget across machines | No node can run the one app that needs most of it | Concentrate RAM when self-hosting; spread it only when practising failures |
| Treating "one node" as permanent | Fear of being stuck keeps you from starting | K3s agents join with one command; start, then grow when there is a reason |

## Lessons

> 💡 **What I learned:** a cluster is more interesting, but one node gets you learning on day one. You can always grow.

## Glossary

| Term | Meaning |
|---|---|
| Node | One machine that runs containers for the cluster |
| Control plane | The part of Kubernetes that stores desired state and makes decisions (API server, scheduler, controllers, database) |
| kubelet | The agent on each node that starts and stops containers as instructed |
| Scheduler | The control-plane component that picks which node a pod runs on |
| High availability (HA) | Service continues when a machine or component fails |
| etcd / SQLite | The cluster's state database: etcd in upstream Kubernetes and HA K3s; SQLite in a single-server K3s |
| Quorum | The strict majority of database copies that must agree; why clusters come in 3s and 5s |
| local-path | K3s's built-in storage class: a folder on the node's own disk |
| Shared storage | Storage reachable from every node, so a pod can move and keep its data (NFS, Longhorn, Ceph) |
| NFS | Network File System: one machine shares a folder, others mount it |
| Longhorn | Distributed block storage for Kubernetes that replicates volumes across nodes |
| Overlay network | The virtual network that lets pods on different machines talk (Flannel in K3s) |
| Hypervisor / Proxmox | Software (or an OS) that runs several virtual machines on one physical one |
| K3s server / agent | The K3s role with the control plane / the role that only runs pods |
| Node token | The secret a new machine presents to join a K3s cluster |

---

[◀ Previous: Raspberry Pi vs Mini PC](02-raspberry-pi-vs-mini-pc.md) · [🏠 Home](../../README.md) · [Next: Budget, Specs and Software Choices ▶](04-budget-and-software-choices.md)
