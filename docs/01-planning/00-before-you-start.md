# 0. Before You Start: What to Know

A "homelab" is just a computer at home that you use to learn and to run your own services. Before spending anything, answer these questions. They matter more than which brand you buy.

This page has no commands. That is deliberate. Every expensive mistake I could have made in this project (buying three machines, buying a Raspberry Pi for the wrong job, putting family photos on a single disk) would have been made at this stage, before I touched a keyboard. Ten minutes with a notebook here saves weeks later.

## What

Answer eight questions, in writing, before you open a single listing. Each later step in Phase 1 takes one of the answers and turns it into a decision:

| Answer | Becomes a decision in |
|---|---|
| What I want to learn or run | [Step 1: goals decide specs](01-goals-decide-specs.md) |
| Which kind of machine fits | [Step 2: Raspberry Pi vs mini PC](02-raspberry-pi-vs-mini-pc.md) |
| How many machines | [Step 3: one node or a cluster](03-one-node-or-cluster.md) |
| Budget, models, OS and Kubernetes flavour | [Step 4: budget and software](04-budget-and-software-choices.md) |
| Where it lives, what you need on install day | [Step 7: display, keyboard and network](../02-hardware/07-display-keyboard-and-network.md) |

## Background: what a homelab actually is

### A server is a role, not a kind of machine

Any computer becomes a server the moment other machines rely on it. The hardware can be identical to a desktop. What changes is how it is used:

| A desktop | A server |
|---|---|
| On when you sit at it | On all the time, including when you are asleep |
| Has a screen, keyboard and mouse | Usually has none after setup. You reach it over the network. |
| Runs programs you click on and close | Runs **services**: programs that start at boot, wait for requests, and never exit |
| If it crashes, you notice and restart it | If it crashes, things silently stop working for everyone at home |
| Lives wherever is comfortable for you | Lives wherever the network cable and power are |

Every question in the table below comes from one of those differences.

### What a "service" costs

A service is a long-running process. While it runs it holds some **RAM** all the time, uses **CPU** only when it has work, and keeps its data on **disk**. Ten small services idle most of the day, so the CPU is rarely busy, but all ten keep their RAM. That is why, as you will see in [step 1](01-goals-decide-specs.md), RAM is the number that decides how much a homelab can run, not CPU speed.

### The layers you will build

It helps to see the whole stack before buying the bottom layer of it. This guide builds it from the ground up, one phase at a time:

```text
 your apps (portfolio, Grafana, AdGuard ...)        ← Phases 7 to 11
 Kubernetes (K3s): decides which app runs where     ← Phase 4
 container runtime: runs each app in its own box    ← installed with K3s
 operating system (Ubuntu Server)                   ← Phase 3
 firmware (BIOS / UEFI): the first code that runs   ← step 9
 hardware (CPU, RAM, SSD, network port)             ← Phase 2, decided in Phase 1
```

Each layer only works if the one under it does. When something breaks later, the first useful question is always "which layer?", and knowing the stack is what makes that question answerable.

### Remote management: why you will not need a screen

After the operating system is installed, you manage a server through **SSH** (Secure Shell): a text session from your laptop to the server, over the network. You type a command on the laptop, it runs on the server, the output comes back. That is why the monitor and keyboard are only needed once, on install day. A machine run this way is called **headless**.

## Background: the costs that are not the box

The listing price is the start of the budget, not the end of it. Here is what can appear on the bill of a project like this one, and when:

| Item | Why you need it | When it came up for me |
|---|---|---|
| The machine | Obviously | [Step 5](../02-hardware/05-buying-the-machine.md) |
| USB flash drive (4GB or more) | To carry the Ubuntu installer | [Step 8](../03-ubuntu-server/08-making-the-install-usb.md) |
| Ethernet cable, long enough to reach the router | Servers should be wired | [Step 7](../02-hardware/07-display-keyboard-and-network.md) |
| Monitor cable or adapter that fits the machine's video output | Install day only | [Step 7](../02-hardware/07-display-keyboard-and-network.md) |
| More RAM | The first upgrade almost everyone makes | Parked, see the roadmap in the [README](../../README.md) |
| A bigger SSD | For photos and files | Parked |
| Backup storage (USB drive, then a cloud bucket) | Because a homelab is not a backup | Parked, and must come before the storage apps |
| UPS (battery backup) | Power cuts | Parked |
| A domain name | Only if you want anything reachable from the internet | [Phase 11](../11-public-access/22-domain-and-cloudflare.md), optional |

Some of these can be borrowed or skipped. Write them down anyway, so the number you call "my budget" is honest.

## The questions

| Question | Why it matters |
|---|---|
| **What do I want to *learn* or *run*?** | This decides CPU, RAM, storage and how many machines you need (see [Start With Your Goal](01-goals-decide-specs.md)). |
| **What's my budget, including upgrades?** | The box is only part of the cost. RAM, SSDs, cables, a flash drive and backup storage add up. |
| **Where will it live?** | It needs power, ventilation and ideally an **Ethernet cable to the router**. Wi-Fi is unreliable for servers. |
| **What about power cuts?** | Without planning, a power cut means the server stays off until you press the button. A BIOS setting fixes this (see [BIOS Settings](../03-ubuntu-server/09-bios-settings.md)); a small UPS is a later upgrade. |
| **Do I have a monitor and keyboard?** | You need them **only for the install**. After that you manage everything remotely over SSH. |
| **What happens to my data if the disk dies?** | If you'll store photos or documents, plan backups from day one. A homelab is not a backup. |
| **How much Linux do I know?** | Pick an OS you can troubleshoot. Learning is the point, but being stuck with no shell is no fun. |
| **Will anything be open to the internet?** | Keep everything **local-only** at first. Exposing services safely (VPN, tunnels, HTTPS) is a later step. |

## Why these eight and not a spec sheet

A spec sheet answers "how fast". These questions answer "for what, where, and what happens when it goes wrong". Two people can buy the same machine and have completely different outcomes because one of them put it next to the router with AC Recovery turned on and a backup plan, and the other put it on Wi-Fi in a cupboard with the only copy of their photos on it.

The last three questions deserve a sentence each, because they are the ones people skip:

- **Disk failure** is not a risk, it is a schedule. Every drive dies eventually. The question is only whether you have a second copy when it does.
- **Linux knowledge** decides which operating system you can rescue at 11pm. An OS with no shell is elegant until you need one.
- **Internet exposure** is the difference between a hobby and a liability. Anything reachable from outside needs HTTPS, authentication and updates, so it waits until those exist.

## How: answering them for yourself

1. **Write the list of things you want to run**, one line each, and mark each one *learn* or *use*. Be specific: "Nextcloud for documents", not "self-hosting".
2. **Write one budget number** that includes the extras from the table above, not just the machine.
3. **Walk the house.** Find the router. Find the nearest power socket. Measure the cable run. If there is no good spot, that is a finding, not a failure; see the options in [step 7](../02-hardware/07-display-keyboard-and-network.md).
4. **Check what you already own:** a monitor and which inputs it has (HDMI, DisplayPort, VGA), a USB keyboard, a flash drive, a spare Ethernet cable.
5. **Decide what data is precious.** If any of it will live on this machine, write "backups" into the plan before the apps that hold it.
6. **Decide "local only" for now.** Write it down so that later, when a tutorial offers to open a port on the router, you remember you chose not to.

### My answers, for the record

| Question | My answer |
|---|---|
| Learn or run | Practise for **CKAD** on real hardware; self-host my portfolio, Prometheus + Grafana, Paperless-ngx, Nextcloud, Immich |
| Budget | **KSh 40,000** |
| Where | Wired to the router with Gigabit Ethernet |
| Power cuts | BIOS **AC Recovery → Power On** on day one; a UPS later |
| Monitor and keyboard | A Lenovo ThinkVision monitor and a USB keyboard, for the install only |
| Data | Backups are parked and come **before** Paperless-ngx, Nextcloud and Immich |
| Linux | An OS with a normal shell (Ubuntu Server) so I can troubleshoot it |
| Internet | Local only first. Public access came much later, through a Cloudflare Tunnel, in Phase 11 |

## Challenges and solutions

Nothing can really go wrong on a page with no commands, but these are the traps I either fell into or watched myself nearly fall into:

| Trap | What happens | How to avoid it |
|---|---|---|
| Starting with "Pi or mini PC?" | You compare hardware without knowing what it has to do, and the comparison has no answer | Write the goals first. The hardware question answers itself in step 2. |
| Budgeting for the box only | The cable, flash drive, adapter and eventually RAM push you past the number | Use the costs table above before you set the number |
| Planning to use Wi-Fi | Dropouts, slower transfers, and a server whose address and reachability depend on signal | Plan a cable run, even a long one |
| Picking a spot with no socket or airflow | The machine ends up behind a sofa or in a closed cupboard | Walk the house first |
| Treating the homelab as the backup | One disk failure deletes the only copy | A homelab is a place to run things, not the place to keep the only copy of anything |
| Opening a port "just to test" | A service with no HTTPS or authentication is reachable by anyone who scans for it | Local only until Phase 9 and 11 |

## Lessons

> 💡 **What I learned:** I started by asking "Pi or mini PC?", but the real first question was *"what do I want to do with it?"*. Once I wrote down my goals, the hardware choice became obvious.

## Glossary

| Term | Meaning |
|---|---|
| Homelab | A computer (or several) at home used to learn and to run your own services |
| Server | Any machine other machines rely on. A role, not a type of hardware. |
| Service | A program that starts at boot, waits for requests, and never exits (a web server, a database, a DNS server) |
| Headless | Running with no monitor or keyboard attached; managed over the network |
| SSH | Secure Shell: an encrypted text session from your laptop to the server |
| BIOS / UEFI | The firmware that runs before the operating system; where settings like AC Recovery live |
| AC Recovery | A BIOS setting that turns the machine back on when power returns after a cut |
| UPS | Uninterruptible power supply: a battery that keeps the machine running through short power cuts |
| Backup | A second copy of data on a different device. Not the same as "it's on the server". |
| LAN | Local area network: the devices connected to your home router |
| Local-only | Reachable from inside the house, not from the internet |

---

[🏠 Home](../../README.md) · [Next: Start With Your Goal ▶](01-goals-decide-specs.md)
