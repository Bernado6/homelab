# 🏠 Homelab: A Single-Node Kubernetes Server at Home

This is how I went from *"should I buy a Raspberry Pi?"* to running apps on my own Kubernetes cluster at home in Nairobi, on a refurbished **Dell OptiPlex 7070 Micro** running Ubuntu Server and K3s.

It's written two ways at once:
- **My story:** what I chose, why, and what went wrong along the way
- **A guide for you:** what to think about before you buy, how goals turn into specs, and the exact steps I followed

Rather than documenting only the final setup, this repo follows the journey step by step, from hardware decisions to a running cluster. Each step ends with **💡 What I learned**, and the general lessons are collected in [Lessons Learned](docs/lessons-learned.md).

## How to read this

Read the steps in order if you're starting from scratch. Every page has **Previous / Home / Next** links at the bottom, so you can follow the whole guide like a course. If you already own hardware, jump to [Phase 3](#phase-3-ubuntu-server). Once the cluster exists (after step 15), start each session with the [Daily Checklist](docs/daily-checklist.md).

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

### Phase 5: Networking
*Make the server reliably reachable from the rest of your network.*

| # | Step | What you'll do |
|---|---|---|
| 14 | [A Fixed IP for the Server (DHCP Reservation)](docs/05-networking/14-dhcp-reservation.md) | Pin the IP in a Huawei HG8145V5, reach a wired-only admin page over an SSH tunnel, verify the lease |

### Phase 6: Working with the cluster
*Manage it from the laptop and expose apps the permanent way.*

| # | Step | What you'll do |
|---|---|---|
| 15 | [Running kubectl From the Laptop](docs/06-working-with-the-cluster/15-kubectl-from-laptop.md) | Copy the kubeconfig, fix the server address, name the context, keep it private |
| 16 | [Exposing an App With an Ingress](docs/06-working-with-the-cluster/16-nginx-ingress.md) | Replace `port-forward` with Deployment + Service + host-based Ingress from a manifest |

### Phase 7: Real workloads
*Your own apps, built on CI and pulled from a registry.*

| # | Step | What you'll do |
|---|---|---|
| 17 | [First Real Workload: The Portfolio Site](docs/07-workloads/17-portfolio.md) | Build on GitHub Actions for amd64, publish to GHCR, deploy with Kustomize into a restricted namespace |

### Phase 8: Observability
*See what the box and the apps are actually doing.*

| # | Step | What you'll do |
|---|---|---|
| 18 | [Monitoring: Prometheus and Grafana](docs/08-observability/18-prometheus-grafana.md) | First Helm chart, first persistent volume, kube-prometheus-stack tuned for 8GB, both UIs behind Ingresses |
| 19 | [Using the Metrics](docs/08-observability/19-using-the-metrics.md) | The three dashboards that matter, PromQL for your pods, Traefik requests per site, limits from evidence |

### Phase 9: TLS
*A padlock on every site, without a public domain.*

| # | Step | What you'll do |
|---|---|---|
| 20 | [HTTPS Everywhere With cert-manager](docs/09-tls/20-https-with-cert-manager.md) | Install cert-manager, run a private CA in the cluster, trust it once on the laptop, three lines per Ingress |

### Reference

- [Daily Checklist](docs/daily-checklist.md): run this before any step. Point kubectl at the homelab, check health, check the sites, sync the repo
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
| Network | Gigabit Ethernet (wired to router), fixed IP via DHCP reservation |
| Router | Huawei EchoLife HG8145V5 (Safaricom Home Fibre) |
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
- [x] DHCP reservation for a fixed IP
- [x] kubectl from the laptop
- [x] Host-based Ingress for apps (`*.home.arpa`)
- [x] Portfolio site (`portfolio.home.arpa`, image from GHCR)
- [x] Prometheus + Grafana (lightweight, with limits)
- [x] HTTPS with cert-manager (private CA for `*.home.arpa`)
- [ ] Backups (external drive / cloud)
- [ ] RAM upgrade → 24GB+
- [ ] 1TB SSD for app data
- [ ] Paperless-ngx
- [ ] Nextcloud
- [ ] Immich
- [ ] UPS for power cuts

This is an evolving project. New steps get added to `docs/` as new capabilities are introduced, and the roadmap above tracks what's done.

## 🔒 What is deliberately not in this repo

This repo is public on purpose, so the line between "useful detail" and "secret" matters.

| In the repo | Kept out |
|---|---|
| Private LAN addresses (`192.168.100.x`). Unreachable from the internet and shared by millions of home networks. | Router admin password, Wi-Fi key and serial number. They live on the router's label and nowhere else. |
| Hostnames and `.home.arpa` names. They only resolve on my own machines. | The kubeconfig. It is cluster-admin and lives in `~/.kube/` on the laptop only. |
| Helm values that reference Secrets **by name**. | The Secrets themselves. Grafana's admin password was created with `kubectl create secret` and never written to a file. |
| The CA's public certificate could be here; the private key could not. | Any private key, including the CA's, which never leaves the cluster. |
| Server username in command prompts. It is not a credential on its own. | SSH keys, and any photo of a device label. |

If you fork this, do the same: anything you would not paste into a public chat does not go in a manifest, a values file, or a screenshot. The [`.gitignore`](.gitignore) backs this up by excluding key and certificate files, kubeconfigs, `.env` files and images, so the common slips can't be committed by accident.

## 📁 Repo layout

```text
README.md                  ← you are here: index, setup summary, roadmap
docs/
  01-planning/             ← steps 0–4: goals, Pi vs mini PC, node count, budget
  02-hardware/             ← steps 5–7: buying, checking, install-day kit
  03-ubuntu-server/        ← steps 8–11: USB, BIOS, install, SSH
  04-kubernetes/           ← steps 12–13: K3s, first app
  05-networking/           ← step 14: fixed IP via DHCP reservation
  06-working-with-the-cluster/  ← steps 15–16: kubectl from the laptop, Ingress
  07-workloads/            ← step 17: portfolio site from GHCR (manifests live in the app repo)
  08-observability/        ← steps 18–19: Prometheus + Grafana, using the metrics
  09-tls/                  ← step 20: cert-manager and a private CA
  daily-checklist.md
  troubleshooting.md
  lessons-learned.md
apps/                      ← manifests and values referenced in the guide
  hello/hello.yaml         ← step 13: whoami behind a catch-all Ingress
  nginx/nginx.yaml         ← step 16: nginx behind a host-based Ingress (TLS added in 20)
  monitoring/values.yaml   ← step 18: kube-prometheus-stack values for 8GB (TLS added in 20)
  cert-manager/            ← step 20: CA issuers and the HTTPS redirect middleware
```

---

*Built in Nairobi 🇰🇪. Notes, issues and suggestions welcome.*
