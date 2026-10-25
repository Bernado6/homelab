# 4. Budget, Specs and Software Choices

**Budget:** KSh 40,000

**What I looked for:**
- Dell OptiPlex 7060/7070 Micro, Lenovo ThinkCentre M720q/M920q, or HP ProDesk/EliteDesk Mini G4/G5
- Intel Core i5/i7, **8th gen or newer**
- 16GB RAM ideally (8GB acceptable to start)
- NVMe SSD
- Gigabit Ethernet

## Software choices

| Layer | Options I considered | What I chose and why |
|---|---|---|
| OS | Ubuntu Server, Debian, Talos Linux | **Ubuntu Server LTS**: normal shell for troubleshooting, huge community, 5 years of updates |
| Kubernetes | K3s, kubeadm, MicroK8s, Talos | **K3s**: one-command install, light on RAM, includes ingress (Traefik), storage (local-path) and metrics |

> ⚠️ **Note:** Talos Linux *is* a Kubernetes distribution; you don't install K3s on top of it. Pick one approach.

> 💡 **What I learned:**
> - **RAM and SSD prices have gone up a lot.** It's usually cheaper to buy a unit that already has the RAM you want, or get the shop to fit it as part of the deal, than to buy sticks separately.
> - An 8th/9th-gen PC only supports **PCIe Gen3** SSDs, so a cheaper Gen3 drive is fine; Gen4 gains nothing.
> - Online AI answers gave me useful starting points but also some wrong details (like running K3s on Talos). Double-check before you buy or install.

---

[◀ Previous: One Node or a Cluster?](03-one-node-or-cluster.md) · [🏠 Home](../../README.md) · [Next: Buying the Machine ▶](../02-hardware/05-buying-the-machine.md)
