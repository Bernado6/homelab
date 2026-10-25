# 🏠 Homelab: A Single-Node Kubernetes Server at Home

This is how I went from *"should I buy a Raspberry Pi?"* to running apps on my own Kubernetes cluster at home in Nairobi, on a refurbished **Dell OptiPlex 7070 Micro** running Ubuntu Server and K3s.

It's written two ways at once:
- **My story:** what I chose, why, and what went wrong along the way
- **A guide for you:** what to think about before you buy, how goals turn into specs, and the exact steps I followed

Rather than documenting only the final setup, this repo follows the journey step by step, from hardware decisions to a running cluster. Each step ends with **💡 What I learned**, and the general lessons are collected in [Lessons Learned](docs/lessons-learned.md).

## How to read this

Read the steps in order if you're starting from scratch. Every page has **Previous / Home / Next** links at the bottom, so you can follow the whole guide like a course. If you already own hardware, jump to [Phase 3](#phase-3-ubuntu-server).

## 📋 The guide

### Phase 1: Planning
*Decide what you want before you spend anything.*

| # | Step | What you'll decide |
|---|---|---|
| 0 | [Before You Start: What to Know](docs/01-planning/00-before-you-start.md) | The questions that matter more than the brand |
| 1 | [Start With Your Goal: It Decides the Specs](docs/01-planning/01-goals-decide-specs.md) | What CPU, RAM and storage your goals actually need |
| 2 | [Raspberry Pi vs Mini PC](docs/01-planning/02-raspberry-pi-vs-mini-pc.md) | Which type of machine fits |
| 3 | [One Node or a Cluster?](docs/01-planning/03-one-node-or-cluster.md) | How many machines |
| 4 | [Budget, Specs and Software Choices](docs/01-planning/04-budget-and-software-choices.md) | Shortlist of models, OS and Kubernetes distribution |

### Phase 2: Hardware
*Buy the machine and check it before wiping it.*

| # | Step | What you'll do |
|---|---|---|
| 5 | [Buying the Machine](docs/02-hardware/05-buying-the-machine.md) | Talk to sellers, checklist before paying |
| 6 | [Checking What I Got](docs/02-hardware/06-checking-what-i-got.md) | Verify specs in Windows, plan upgrades in stages |
| 7 | [Display, Keyboard and Network](docs/02-hardware/07-display-keyboard-and-network.md) | What you need for install day, and where the box lives |

### Phase 3: Ubuntu Server
*From Windows to a headless Linux server you reach over SSH.*

| # | Step | What you'll do |
|---|---|---|
| 8 | [Making the Ubuntu Install USB](docs/03-ubuntu-server/08-making-the-install-usb.md) | Download the ISO, flash with Rufus |
| 9 | [BIOS Settings](docs/03-ubuntu-server/09-bios-settings.md) | AHCI, AC Recovery, Secure Boot |
| 10 | [Installing Ubuntu Server](docs/03-ubuntu-server/10-installing-ubuntu-server.md) | Boot from USB, every installer screen explained |
| 11 | [First Login and SSH](docs/03-ubuntu-server/11-first-login-and-ssh.md) | Update, find the IP, go headless |

### Phase 4: Kubernetes
*Install K3s and run your first app through an Ingress.*

| # | Step | What you'll do |
|---|---|---|
| 12 | [Installing Kubernetes (K3s)](docs/04-kubernetes/12-installing-k3s.md) | One-command install, what each system pod does |
| 13 | [My First App](docs/04-kubernetes/13-first-app.md) | Deployment + Service + Ingress, and how a request flows |

### Reference

- [Troubleshooting](docs/troubleshooting.md): every problem I hit, with the fix and the step it belongs to
- [Lessons Learned](docs/lessons-learned.md): the general takeaways
- [`apps/`](apps/): Kubernetes manifests used in the guide, ready to `kubectl apply`

## 🖥️ My setup at a glance

| Item | Detail |
|---|---|
| Machine | Dell OptiPlex 7070 Micro (refurbished) |
| CPU | Intel Core i5-9500T (6 cores) |
| RAM | 8GB DDR4 (upgrade planned) |
| Storage | 256GB NVMe SSD |
| Network | Gigabit Ethernet (wired to router) |
| Display (install only) | Lenovo ThinkVision monitor + USB keyboard |
| OS | Ubuntu Server 26.04.1 LTS |
| Kubernetes | K3s v1.36.5+k3s1 |
| Ingress | Traefik (bundled with K3s) |
| Storage class | local-path (bundled with K3s) |
| Managed from | Laptop over SSH |

## 🗺️ Roadmap

- [x] Define goals and budget
- [x] Choose hardware (one mini PC)
- [x] Buy and check the OptiPlex 7070 Micro (i5-9500T / 8GB / 256GB)
- [x] Install Ubuntu Server 26.04.1 LTS
- [x] SSH from laptop
- [x] K3s single-node cluster
- [x] First app (whoami) through Ingress
- [ ] DHCP reservation for a fixed IP
- [ ] Portfolio site
- [ ] Prometheus + Grafana (lightweight, with limits)
- [ ] HTTPS with cert-manager
- [ ] Backups (external drive / cloud)
- [ ] RAM upgrade → 24GB+
- [ ] 1TB SSD for app data
- [ ] Paperless-ngx
- [ ] Nextcloud
- [ ] Immich
- [ ] UPS for power cuts

This is an evolving project. New steps get added to `docs/` as new capabilities are introduced, and the roadmap above tracks what's done.

## 📁 Repo layout

```text
README.md                  ← you are here: index, setup summary, roadmap
docs/
  01-planning/             ← steps 0–4: goals, Pi vs mini PC, node count, budget
  02-hardware/             ← steps 5–7: buying, checking, install-day kit
  03-ubuntu-server/        ← steps 8–11: USB, BIOS, install, SSH
  04-kubernetes/           ← steps 12–13: K3s, first app
  troubleshooting.md
  lessons-learned.md
apps/
  hello/hello.yaml         ← manifests referenced in the guide
```

---

*Built in Nairobi 🇰🇪. Notes, issues and suggestions welcome.*
