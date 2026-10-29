# 4. Budget, Specs and Software Choices

The three pages before this settled *what kind* of machine and *how many*. This one turns that into a shortlist of actual models a seller will recognise, a set of must-haves to check against, and the two software decisions that shape the rest of the guide: which operating system, and which Kubernetes. It is also the page where I learned to double-check advice.

## What

**Budget:** KSh 40,000

**What I looked for:**
- Dell OptiPlex 7060/7070 Micro, Lenovo ThinkCentre M720q/M920q, or HP ProDesk/EliteDesk Mini G4/G5
- Intel Core i5/i7, **8th gen or newer**
- 16GB RAM ideally (8GB acceptable to start)
- NVMe SSD
- Gigabit Ethernet

Plus two software choices: **Ubuntu Server LTS** as the operating system and **K3s** as the Kubernetes distribution.

## Background: the used office PC market

Why those three names and not a random brand? Dell OptiPlex, Lenovo ThinkCentre and HP ProDesk/EliteDesk are **business** lines. Companies lease them in bulk and return them after a few years, so refurbishers get pallets of identical machines. That gives you things consumer mini PCs do not:

| What business lines give you | Why it matters for a homelab |
|---|---|
| Thousands of identical units | Prices are competitive and spares are easy to find |
| Service manuals and parts lists published by the maker | You can look up exactly how many RAM slots and which drive bays a model has before you buy |
| BIOS and firmware updates for years | Security fixes and the settings you need ([step 9](../03-ubuntu-server/09-bios-settings.md)) |
| A **service tag** or serial that the maker's site recognises | You can verify what the machine shipped with ([step 5](../02-hardware/05-buying-the-machine.md)) |
| Standard laptop-class parts | SO-DIMM RAM and M.2 SSDs are the same ones sold everywhere |

Decoding the model names, since sellers quote them and they are easy to confuse:

| Family | Models on my list | How the generation shows in the name |
|---|---|---|
| Dell OptiPlex Micro | **7060** (8th gen), **7070** (9th gen) | The last two digits move with the CPU generation; the cases look identical |
| Lenovo ThinkCentre Tiny | **M720q**, **M920q** (both launched with 8th gen CPUs; the 9-series is the higher-end line with more expansion) | The **q** means the Tiny form factor |
| HP ProDesk / EliteDesk Mini | **G4** (8th gen), **G5** (9th gen) | The **G** number is the hardware generation; EliteDesk is the premium line |

"Refurbished" should mean tested, cleaned, and sold with some warranty. "Used" means someone's old machine. Ask which one you are being offered.

## Background: reading an Intel CPU name

A seller writes "ci5 9th gen". The spec sheet says **i5-9500T**. Learn to read the second form, because the first one hides most of what matters:

```text
 i5 - 9 500 T
 │    │  │   └─ suffix: T = low power (35W), designed for small cases
 │    │  └───── model within the generation; higher is usually faster or has more cores
 │    └──────── generation: 9 = 9th gen
 └───────────── tier: i3 < i5 < i7 < i9
```

| Suffix | Means | Typical in |
|---|---|---|
| (none) | Standard desktop part, 65W | Full-size desktops |
| **T** | Low power, **35W** | Micro / Tiny / Mini PCs |
| K | Unlocked for overclocking, 95W and up | Gaming desktops |
| U | Ultra-low power laptop part | Laptops |

The watt figure is the **TDP** (thermal design power): how much heat the cooler is built to remove under sustained load, and a good proxy for electricity use. A 35W T chip gives up some peak speed in exchange for running cool and quiet in a one-litre box 24 hours a day, which is exactly the trade a homelab wants.

## Background: PCIe generations, and why Gen3 is enough

**PCIe** (PCI Express) is the bus that connects fast devices, including an NVMe SSD, to the CPU. A device uses a number of **lanes** (an NVMe drive uses four, written x4), and each **generation** of PCIe doubles the speed of a lane:

| PCIe generation | Per lane, roughly | An x4 NVMe SSD, roughly | Platforms |
|---|---|---|---|
| Gen3 | ~1 GB/s | ~3.5 GB/s | Intel 8th and 9th gen (the machines on my list) |
| Gen4 | ~2 GB/s | ~7 GB/s | Intel 11th gen and newer, AMD Ryzen 3000 and newer |
| Gen5 | ~4 GB/s | ~14 GB/s | Recent desktops |

PCIe is backward compatible: a Gen4 drive in a Gen3 slot works, at Gen3 speed. So on an 8th or 9th-gen machine, a Gen4 SSD costs more and delivers exactly what a Gen3 one does. Buy the cheaper drive. For a homelab the difference between 3.5 and 7 GB/s would be invisible anyway; what matters is that it is an SSD and not a spinning disk ([step 1](01-goals-decide-specs.md)).

## Background: the RAM you are shopping for

These machines take **DDR4 SO-DIMM** sticks, the laptop size, in **two slots**. Two things to know before talking to a seller:

- "8GB" can be **one 8GB stick** (one slot free: add a second stick and you have 16GB) or **two 4GB sticks** (both slots full: upgrading means throwing both away). Ask which. [Step 6](../02-hardware/06-checking-what-i-got.md) shows how to check it yourself.
- RAM is priced per stick and prices move. The time to get RAM cheaply is often **as part of the purchase**, because the shop has pulled sticks from other machines and will fit one for less than retail.

## Background: operating systems for a server

An operating system choice for a server is a choice about *how you will fix it at 11pm*. The three I considered:

| OS | What it is | How you work with it |
|---|---|---|
| **Ubuntu Server LTS** | Debian-based Linux from Canonical, server edition with no desktop. **LTS** (long-term support) releases get five years of updates. | A normal shell over SSH. `apt` for packages. The biggest community and the most tutorials of any server OS. |
| **Debian** | The project Ubuntu is built on. Leaner, more conservative, slower to change. | The same shell and `apt`. Slightly older packages, fewer beginner guides. |
| **Talos Linux** | An **immutable** OS built only to run Kubernetes. There is no shell, no SSH and no package manager at all; you configure it entirely through an API with `talosctl`. | Excellent in production because there is nothing to drift. Hard for a first-timer because when something is wrong you cannot log in and look. |

The question from [step 0](00-before-you-start.md), "how much Linux do I know?", is answered here. I wanted to *learn* by having a shell, and I wanted to be able to troubleshoot by having a shell.

## Background: Kubernetes distributions

A Kubernetes **distribution** packages the same Kubernetes API and components in a different way, exactly like Linux distributions package the same kernel. Whatever you pick, `kubectl` and the YAML are identical, which is why CKAD practice on K3s transfers to any cluster.

| Distribution | What it is | What you get out of the box | RAM at idle, roughly |
|---|---|---|---|
| **K3s** | A single binary from Rancher (now SUSE), built for edge and small machines | containerd, Flannel networking, CoreDNS, **Traefik** ingress, **ServiceLB**, the **local-path** storage class, metrics-server. One curl-to-shell command and it runs. | under 1GB |
| **kubeadm** | The upstream bootstrapping tool | A bare control plane. You install the container runtime, networking, ingress, storage and metrics yourself. Closest to what the CKA exam uses. | more, and several afternoons of setup |
| **MicroK8s** | Canonical's distribution, installed as a snap | A bare cluster plus `microk8s enable` for add-ons (dns, ingress, storage) | about the same as K3s |
| **Talos** | The OS and Kubernetes together | A production-style cluster with no shell | needs its own thinking, see the note below |

K3s bundling Traefik, ServiceLB and local-path is what lets [steps 12 to 18](../04-kubernetes/12-installing-k3s.md) go from nothing to Ingress and persistent volumes without installing anything extra. You will meet each of those bundled pieces later and the names will make sense then.

## Software choices

| Layer | Options I considered | What I chose and why |
|---|---|---|
| OS | Ubuntu Server, Debian, Talos Linux | **Ubuntu Server LTS**: normal shell for troubleshooting, huge community, 5 years of updates |
| Kubernetes | K3s, kubeadm, MicroK8s, Talos | **K3s**: one-command install, light on RAM, includes ingress (Traefik), storage (local-path) and metrics |

> ⚠️ **Note:** Talos Linux *is* a Kubernetes distribution; you don't install K3s on top of it. Pick one approach.

### Why not the others

| Option | Why not, for a first single-node homelab |
|---|---|
| Debian | Nothing wrong with it. Ubuntu Server has more first-timer documentation and the installer sets up SSH in one checkbox. A coin flip I resolved toward the bigger community. |
| Talos Linux | No shell. Every lesson in this guide that starts with "SSH in and look" would be impossible. Worth revisiting when the goal is "run it like production" rather than "learn it". |
| kubeadm | The right tool for CKA practice, where installing the cluster *is* the lesson. For CKAD it is an afternoon of setup that teaches nothing on the syllabus. |
| MicroK8s | A fine alternative. It runs as a snap, which I preferred not to depend on, and its add-on model is a little less transparent than K3s's "it's all in one binary". |

## How: turning this into a shopping message

1. **Write the shortlist** of model names from the "What I looked for" list. Sellers search their stock by those names.
2. **Write the must-haves underneath:** i5 or i7, 8th gen or newer, NVMe SSD, Gigabit Ethernet, RAM amount, power adapter included.
3. **Decide your fallbacks before you ask.** Mine were: 8GB acceptable to start; i5 acceptable if it is 6 cores; 256GB acceptable because a bigger drive is a later purchase.
4. **Send that list, without the budget number**, to a seller (or several). I messaged one local seller; what came back, and the checklist I used to compare the offers, is [step 5](../02-hardware/05-buying-the-machine.md).
5. **Budget the extras** from [step 0](00-before-you-start.md) against what is left after the machine.

## When

After the node count ([step 3](03-one-node-or-cluster.md)), because one machine versus three changes what KSh 40,000 can buy. Before you message anyone.

## Challenges and solutions

| Challenge | What was going on | Solution |
|---|---|---|
| Advice that said "run K3s on Talos" | Talos Linux is a Kubernetes distribution with no shell to install anything on. The advice mixed two options from the same column as if they stacked. | Read what each tool actually is before combining them. Pick one Kubernetes, pick one OS, and check that the pairing exists. |
| Prices in online advice did not match the shops | RAM and SSD prices had risen since the advice was written | Treat prices from any article or chat as a starting point and price locally on the day |
| Listings that only said "i5 9th gen" | The tier and generation are there; the core count and T suffix are not. An i5-9400T and an i5-9500T are both "i5 9th gen". | Ask for the exact model string ([step 5](../02-hardware/05-buying-the-machine.md)) |
| Being tempted by a Gen4 SSD "for future-proofing" | The machine's PCIe is Gen3; the drive would run at Gen3 speed | Buy Gen3 for an 8th/9th-gen platform |
| "8GB" that turns out to be two 4GB sticks | Both slots full; the 16GB upgrade now costs two sticks instead of one | Ask how many slots are used, and verify in [step 6](../02-hardware/06-checking-what-i-got.md) |

## Lessons

> 💡 **What I learned:**
> - **RAM and SSD prices have gone up a lot.** It's usually cheaper to buy a unit that already has the RAM you want, or get the shop to fit it as part of the deal, than to buy sticks separately.
> - An 8th/9th-gen PC only supports **PCIe Gen3** SSDs, so a cheaper Gen3 drive is fine; Gen4 gains nothing.
> - Online AI answers gave me useful starting points but also some wrong details (like running K3s on Talos). Double-check before you buy or install.

## Glossary

| Term | Meaning |
|---|---|
| Business line | PC ranges sold to companies (OptiPlex, ThinkCentre, ProDesk/EliteDesk); plentiful and well documented second-hand |
| Refurbished | Used, tested, cleaned and usually sold with a short warranty |
| Service tag | Dell's per-machine identifier; look it up on Dell's support site to see the original configuration |
| i5-9500T | Tier i5, 9th generation, model 500, low-power T variant |
| TDP | Thermal design power: the sustained heat (and roughly the power) a CPU is designed for; 35W for T chips |
| DDR4 SO-DIMM | The laptop-sized RAM stick these machines use |
| PCIe / lane / generation | The bus fast devices use; width in lanes; each generation doubles per-lane speed |
| Gen3 / Gen4 | PCIe 3.0 (~3.5 GB/s for an x4 SSD) / PCIe 4.0 (~7 GB/s). 8th/9th-gen Intel is Gen3. |
| LTS | Long-term support: an Ubuntu release maintained for five years |
| Immutable OS | An OS whose system files cannot be changed at runtime (Talos); configured through an API, not a shell |
| Kubernetes distribution | A packaging of the same Kubernetes API with different bundled components and installers |
| K3s | Lightweight single-binary Kubernetes that bundles networking, ingress, storage and metrics |
| kubeadm | The upstream tool that bootstraps a bare cluster; what CKA practises |
| MicroK8s | Canonical's snap-packaged Kubernetes with enable-able add-ons |
| Traefik / ServiceLB / local-path | The ingress controller, load-balancer substitute and storage class that K3s ships with |
| snap | Canonical's self-contained package format |

---

[◀ Previous: One Node or a Cluster?](03-one-node-or-cluster.md) · [🏠 Home](../../README.md) · [Next: Buying the Machine ▶](../02-hardware/05-buying-the-machine.md)
