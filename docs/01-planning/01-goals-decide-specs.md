# 1. Start With Your Goal: It Decides the Specs

Hardware shopping should start with a list of things you want to *do*, not a list of parts. This page turns the first answer from [step 0](00-before-you-start.md) ("what do I want to learn or run?") into four numbers: how many CPU cores, how much RAM, what kind of storage, and how many machines. Everything after this in Phase 1 is choosing a box that matches those numbers.

## What

Pick your goal (or goals) from the table further down, read off what each one needs, and take the biggest requirement across them. That is your spec. For me the result was one x86 mini PC running K3s; your result may be different, and that is the point.

## Background: the four specs and what each one actually does

Before the tables make sense, you need to know what the words in them mean. If you already know what a core, a gigabyte and NVMe are, skip to the next Background section.

### CPU: cores, threads and generation

A **core** is an independent unit inside the processor that runs one stream of instructions at a time. Six cores can run six things at once. A **thread** (Hyper-Threading on Intel) lets one core juggle two streams; it helps a little, not twice as much. Kubernetes measures CPU in **millicores** (`500m` = half a core), and every container on the node shares the same pool.

The **generation** matters more than the clock speed. Intel numbers them: an i5-**8**500T is 8th gen, an i5-**9**500T is 9th gen. The 8th generation is the usual cutoff in used-PC advice because that is when the i5 went from four cores to six, so an 8th-gen i5 does more parallel work than a 7th-gen i7. Clock speed (the GHz number) decides how fast one core finishes one task, and for a homelab full of idle services it is rarely the limit.

The **T** at the end of a model like i5-9500T means a low-power version of the chip, designed for small cases. It runs cooler and quieter and draws less electricity, at the cost of a lower sustained clock. For a machine that sits on 24 hours a day, that is a good trade. More on this in [step 4](04-budget-and-software-choices.md).

### RAM: the one that runs out

**RAM** (memory) is the fast working space every running program lives in. It is measured in gigabytes (GB), and in a homelab it is the resource that runs out first. The reason is an asymmetry:

| Resource | When it is scarce | Result |
|---|---|---|
| CPU | Processes take turns (time-slicing) | Everything gets **slower**, nothing dies |
| RAM | There are no turns. A process either has its memory or it does not. | The kernel's **OOM killer** terminates a process to free memory. In Kubernetes the pod shows `OOMKilled` and restarts, or new pods sit in `Pending` because the scheduler cannot find room. |

Services hold their RAM all day whether or not they are busy, so ten idle services cost ten lots of RAM and almost no CPU. On top of the apps, the operating system and Kubernetes itself take a share before you run anything. Rough shape of an 8GB box:

```text
 8GB total
 ├─ Ubuntu Server itself                  a few hundred MB
 ├─ K3s (control plane, kubelet, CoreDNS, Traefik ...)  roughly 1GB
 ├─ monitoring (Prometheus + Grafana, with limits)       1 to 2GB
 └─ what is left for your apps            the rest, and it goes fast
```

Those are orders of magnitude, not measurements; [step 19](../08-observability/19-using-the-metrics.md) is where I measured the real numbers. The lesson is the same: buy a box whose RAM can grow.

### Storage: HDD, SATA SSD, NVMe

| Type | How it works | Speed, roughly | Good for |
|---|---|---|---|
| HDD (hard disk) | Spinning magnetic platters, a moving read head | ~150 MB/s, slow when many small reads happen at once | Bulk storage: photos, media, backups |
| SATA SSD | Flash memory over the old disk interface | up to ~550 MB/s, the interface is the cap | Cheap general use |
| NVMe SSD | Flash memory plugged straight into the CPU's PCIe lanes, M.2 stick form | several GB/s | The OS, databases, container images, anything that does many small reads and writes |

Kubernetes and its apps do a lot of small random reads and writes (databases, logs, the cluster's own state store), which is exactly what spinning disks are worst at. So the rule is: **NVMe for the OS and apps, a big cheap drive later for the files**. Capacity for the OS, K3s and a dozen container images is modest; capacity for a photo library is not, and that is a separate purchase.

### Network

**Gigabit Ethernet** moves up to 1000 megabits per second, about 125 MB/s, in both directions at once, with no interference. Wi-Fi shares one radio channel with every other device in range, in one direction at a time, and its speed changes with distance and walls. A server is the one device at home that should always have the same address and the same reachability, so it should be wired. Most used office mini PCs have a Gigabit port built in and **no Wi-Fi card at all**, which is fine for a server.

### Upgradeability

A machine with two **RAM slots** and one of them empty can be doubled with one stick. A machine with soldered RAM is fixed forever. A spare **drive bay** means the photo drive can go inside the case instead of dangling off a USB port. These two facts about a used machine matter more than whether its CPU is 10% faster.

## Background: CKAD vs CKA, and why it changes the hardware

Both are Kubernetes certifications from the CNCF, both are hands-on exams in a terminal, and people say "a Kubernetes homelab" for either. They need different hardware.

First, the two halves of any Kubernetes cluster:

```text
 control plane  ── API server, scheduler, controllers, the cluster's database (etcd, or SQLite in K3s)
                   "what should be running, and where"
 nodes          ── kubelet + container runtime on each machine
                   "actually run the containers I was told to"
```

On a single-node K3s install, one machine plays both roles.

| | CKAD (Application Developer) | CKA (Administrator) |
|---|---|---|
| The job it tests | Building and running **apps on** a cluster someone else operates | Installing, upgrading and repairing **the cluster itself** |
| Typical tasks | Deployments, Services, Ingress, ConfigMaps and Secrets, probes, resource requests and limits, PersistentVolumeClaims, Jobs and CronJobs, debugging a pod | Bootstrapping with kubeadm, backing up and restoring etcd, upgrading nodes, draining and cordoning, fixing a broken kubelet, cluster networking |
| Scope | Mostly inside a namespace | Mostly the machines and the control plane |
| Does one node teach it? | **Yes.** Everything on that list works on one node. | **Only partly.** You cannot practise a node failure, a drain, or an upgrade rollout on a cluster with one node. |
| Hardware that fits | One box, 6+ cores, 8 to 16GB RAM | Three machines, or one big machine running three VMs |

I was preparing for CKAD. That single fact removed two machines from my shopping list.

## Common goals and what they need

| If your goal is… | Typical workloads | What you need | Good fit |
|---|---|---|---|
| **Learn Linux basics / one small service** | Pi-hole, WireGuard VPN, home automation | Low power, 2–4GB RAM | Raspberry Pi |
| **Electronics and sensors** | GPIO, IoT, sensor logging | GPIO pins | Raspberry Pi |
| **Self-host a few apps with Docker** | Nextcloud, Paperless-ngx, Jellyfin | x86 CPU, 16GB RAM, NVMe SSD | One used mini PC |
| **Learn Kubernetes as a developer (e.g. CKAD)** | Deployments, Services, Ingress, PVCs, probes | 6+ cores, 8–16GB RAM | One mini PC with K3s ✅ *(me)* |
| **Learn Kubernetes administration / high availability (e.g. CKA)** | Multiple nodes, node failures, distributed storage | 3 machines with 16GB each, or one big machine running VMs | 3 used mini PCs, or one 64GB machine with Proxmox |
| **Photos with AI features (Immich)** | Face/object recognition, thumbnails, video | Strong CPU (iGPU helps), 16–32GB RAM, lots of storage | Mini PC with RAM and storage upgrades |
| **Lots of storage (NAS)** | File server, backups, media library | Several drive bays | Small-form-factor desktop or a NAS |
| **Monitoring and observability** | Prometheus, Grafana | Fast SSD, a few GB RAM | Any mini PC |

A few notes on the rows, because the reasons are more useful than the numbers:

- **Pi-hole, WireGuard, home automation** are single small processes that idle. They are what a Raspberry Pi was made for.
- **GPIO** means the row of pins on a Pi that you can wire sensors and LEDs to. No PC has them. If this is your goal, the rest of this guide is the wrong guide.
- **Nextcloud, Paperless-ngx, Jellyfin** each bring a database, a cache and a worker process. Three of them together is where 8GB starts to hurt.
- **Immich** runs machine-learning models for face and object recognition. That is the one workload here that is CPU-hungry as well as RAM-hungry, and a built-in GPU (iGPU) helps with video transcoding.
- **Proxmox** is a hypervisor: an operating system whose only job is to run virtual machines. One 64GB machine with Proxmox can pretend to be three Kubernetes nodes, which is the cheapest way to practise CKA.

## Specs: what actually matters

| Component | Guidance |
|---|---|
| **RAM** | The real limit in a homelab. 8GB is enough to start, 16GB is comfortable, 32GB+ runs everything. |
| **CPU** | Intel 8th gen or newer (6 cores) is the sweet spot for used machines. You rarely need the newest chip. |
| **Storage** | NVMe SSD for the OS and apps. Add a bigger drive later for photos and files. |
| **Network** | Gigabit Ethernet. Check whether Wi-Fi is included (often it isn't). |
| **Upgradeability** | RAM slots and drive bays matter more than raw speed. |

### How to read the two tables together

1. Find every row in the goals table that applies to you. Most people have two or three.
2. For each component, take the **largest** requirement across your rows. One Immich row outweighs five Pi-hole rows.
3. Decide whether you need that largest requirement **on day one** or can grow into it. If you can grow, buy the upgradeable box with the smaller spec now.
4. Check the Good fit column. If every row says Raspberry Pi, buy a Pi. If any row says mini PC, you need a mini PC.

## My goals

1. Practise Kubernetes for the **CKAD** exam on real hardware (I'd been using minikube on my laptop)
2. Self-host: **my portfolio site, Prometheus + Grafana, Paperless-ngx, Nextcloud, Immich**
3. Stay within **KSh 40,000**

Those goals pointed to **one x86 mini PC running K3s**.

### How each goal became a spec

| Goal | What it implied |
|---|---|
| CKAD on real hardware | One node is enough. 6+ cores, 8 to 16GB. x86 so every image I meet in practice runs unchanged. |
| minikube was not enough | I wanted a cluster that stays up when the laptop closes, with a real network, a real Ingress and real persistent storage |
| Portfolio site | Tiny. A static site in a container costs almost nothing. |
| Prometheus + Grafana | A fast SSD and a few GB of RAM, with limits set so they cannot take the whole box |
| Paperless-ngx, Nextcloud, Immich | 16 to 32GB and a big drive. Not affordable on day one, so the box must have a free RAM slot and a drive bay to grow into. |
| KSh 40,000 | A used office machine, not a new one. Start at 8GB and upgrade. |

The common thread is **upgradeable x86 with a free slot and a bay**. That phrase is what I took to the sellers in [step 5](../02-hardware/05-buying-the-machine.md).

## When

Do this before looking at any listing. Come back to it whenever you add a goal: the day I add Immich, this table is what tells me to buy RAM first.

## Challenges and solutions

Nothing broke at this stage, but these are the ways the reasoning goes wrong:

| Trap | What happens | How to avoid it |
|---|---|---|
| Buying for the final goal on day one | The budget is blown on 32GB before you have run a single pod | Buy upgradeable, start small, add apps in stages ([step 6](../02-hardware/06-checking-what-i-got.md)) |
| Chasing GHz | A fast 4-core 7th-gen i7 loses to a slower 6-core 8th-gen i5 for many idle services | Generation and core count first, clock speed last |
| Assuming "Kubernetes" means three machines | Three times the cost for a CKAD syllabus that fits on one node | Know which exam, or which job, you are preparing for |
| Forgetting that the OS and K3s eat RAM before the apps do | "8GB" feels like 8GB for apps; it is not | Budget about a quarter of a small box for the system itself |
| Picking a Pi for Immich or Nextcloud | Fixed RAM and ARM-only images turn a weekend project into a month | Match the Good fit column, not the price |

## Lessons

> 💡 **What I learned:** "Kubernetes homelab" means different hardware for different people. For **CKAD**, which is about building and running apps, one node teaches nearly everything. For **CKA**, which is about running the cluster itself, you want several nodes. Knowing which exam I was preparing for saved me from buying three machines.

## Glossary

| Term | Meaning |
|---|---|
| Core / thread | An independent execution unit in the CPU / a second instruction stream a core can juggle |
| Millicore | Kubernetes' CPU unit: `1000m` is one core |
| Generation (8th gen, 9th gen) | Intel's yearly CPU design; the first digit of the model number (i5-**9**500T) |
| T suffix | Low-power version of an Intel desktop CPU, used in small cases |
| RAM | Working memory. Every running program lives in it. The homelab bottleneck. |
| OOM killer / `OOMKilled` | What the kernel does when RAM runs out: it terminates a process. Kubernetes reports the pod as `OOMKilled`. |
| HDD / SATA SSD / NVMe | Spinning disk / flash over the old disk bus / flash directly on PCIe lanes, the fastest |
| M.2 | The stick-shaped slot most NVMe SSDs plug into |
| Gigabit Ethernet | Wired networking at 1000 Mbit/s, about 125 MB/s |
| iGPU | A graphics unit built into the CPU; helps with video and some ML workloads |
| GPIO | General-purpose input/output pins on a Raspberry Pi for sensors and electronics |
| Proxmox | A hypervisor OS for running virtual machines; one way to fake a multi-node cluster on one box |
| Control plane / node | The part of Kubernetes that decides, and the part that runs containers. One machine can be both. |
| CKAD / CKA | Certified Kubernetes Application Developer / Administrator |
| minikube | A single-node Kubernetes that runs inside a VM or container on a laptop, for learning |

---

[◀ Previous: Before You Start](00-before-you-start.md) · [🏠 Home](../../README.md) · [Next: Raspberry Pi vs Mini PC ▶](02-raspberry-pi-vs-mini-pc.md)
