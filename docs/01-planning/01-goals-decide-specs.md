# 1. Start With Your Goal: It Decides the Specs

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

## Specs: what actually matters

| Component | Guidance |
|---|---|
| **RAM** | The real limit in a homelab. 8GB is enough to start, 16GB is comfortable, 32GB+ runs everything. |
| **CPU** | Intel 8th gen or newer (6 cores) is the sweet spot for used machines. You rarely need the newest chip. |
| **Storage** | NVMe SSD for the OS and apps. Add a bigger drive later for photos and files. |
| **Network** | Gigabit Ethernet. Check whether Wi-Fi is included (often it isn't). |
| **Upgradeability** | RAM slots and drive bays matter more than raw speed. |

## My goals

1. Practise Kubernetes for the **CKAD** exam on real hardware (I'd been using minikube on my laptop)
2. Self-host: **my portfolio site, Prometheus + Grafana, Paperless-ngx, Nextcloud, Immich**
3. Stay within **KSh 40,000**

Those goals pointed to **one x86 mini PC running K3s**.

> 💡 **What I learned:** "Kubernetes homelab" means different hardware for different people. For **CKAD**, which is about building and running apps, one node teaches nearly everything. For **CKA**, which is about running the cluster itself, you want several nodes. Knowing which exam I was preparing for saved me from buying three machines.

---

[◀ Previous: Before You Start](00-before-you-start.md) · [🏠 Home](../../README.md) · [Next: Raspberry Pi vs Mini PC ▶](02-raspberry-pi-vs-mini-pc.md)
