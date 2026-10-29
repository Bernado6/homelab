# 2. Raspberry Pi vs Mini PC

"Should I buy a Raspberry Pi?" was the question I started the whole project with, before [step 0](00-before-you-start.md) taught me to ask about goals first. It is still a real question, because a Pi and a used mini PC cost about the same and look like competitors. They are not. They are two different tools, and this page is about telling them apart.

## What

Decide which **kind** of machine your goals from [step 1](01-goals-decide-specs.md) need: a Raspberry Pi (a single-board ARM computer) or a used office mini PC (a small x86 desktop). Which specific model comes in [step 4](04-budget-and-software-choices.md).

## Background: what each one actually is

### Raspberry Pi: a single-board computer

A Raspberry Pi is a **single-board computer (SBC)**: the processor, memory and all the connectors sit on one bare circuit board the size of a credit card. The current model is the **Raspberry Pi 5**. Some things about it that matter for this decision:

| Part | On a Pi |
|---|---|
| Processor | An **ARM** system-on-chip, the same family of chips as phones use |
| Memory | **Soldered** to the board. You choose the amount when you buy, and it never changes. |
| Storage | A **microSD** card by default. The Pi 5 exposes a PCIe connector so an NVMe SSD can be added, but only through an add-on board (a "HAT"). |
| GPIO | A 40-pin header you can wire sensors, relays, LEDs and displays to. No PC has this. |
| Power | A USB-C supply, and the Pi 5 is fussy about getting a strong enough one |
| Cooling | None on the board. The Pi 5 runs hot enough to want an active cooler (a small fan). |
| Case | None. The board ships bare. |
| Video | Two micro-HDMI ports, so you need a micro-HDMI cable or adapter |
| Network | Gigabit Ethernet and Wi-Fi built in |

A Pi is cheap as a board. It becomes a computer only after you add the parts in the lower rows.

### Mini PC: a one-litre office desktop

"Mini PC" here means a specific family of machines: the **Dell OptiPlex Micro**, **Lenovo ThinkCentre Tiny** and **HP ProDesk / EliteDesk Mini**. They are about one litre in volume, which is why you will also see them called "1L PCs" or "tiny/mini/micro". Companies buy them by the thousand for office desks and replace them on three to five year cycles, so refurbished units are plentiful and cheap.

Inside, they are built from laptop-class parts in a desktop-shaped box:

| Part | In a 1L mini PC |
|---|---|
| Processor | An **x86** (Intel or AMD) desktop CPU, usually the low-power **T** variant |
| Memory | Two **SO-DIMM** slots (the small laptop-style RAM sticks). Replaceable and upgradeable. |
| Storage | An **M.2 slot for an NVMe SSD** built in. Many also have a **2.5-inch bay** for a second SATA drive. |
| Power | An external power brick, like a laptop's |
| Cooling | A proper heatsink and fan, already fitted |
| Case | It is a case |
| Video | DisplayPort outputs (sometimes HDMI too) |
| Network | Gigabit Ethernet built in. **Wi-Fi often not fitted**, since office desks are wired. |

Everything the Pi needs you to add, the mini PC already has, which is the whole "real cost" argument below.

## Background: ARM vs x86, and why it matters for containers

A CPU understands one **instruction set**, the vocabulary of operations it can run. The two that matter here:

- **x86-64** (also called `amd64`): Intel and AMD. Every desktop, laptop and server PC for the last twenty years.
- **ARM64** (also called `aarch64`): phones, Apple Silicon Macs, Raspberry Pis, some cloud servers.

A program compiled for one does not run on the other. A **container image** is just a packaged program with its files, so images are built **per architecture**. On a registry you will see tags like `linux/amd64` and `linux/arm64`. Many well-maintained projects publish a **multi-arch** image, where the registry hands each machine the right build automatically. Many smaller or older projects publish only `amd64`, because that is what the author and almost every server runs.

What this means in practice:

| On | Pulling a random image from Docker Hub or GitHub |
|---|---|
| x86 mini PC | Works. Every image you will ever meet has an amd64 build. |
| Raspberry Pi | Works **if** the project built for arm64. Otherwise the container starts and dies with `exec format error`, and you are looking for a fork or building it yourself. |

For learning Kubernetes, where you follow tutorials that pull dozens of different images, "every image works" removes a whole category of problems that have nothing to do with what you are trying to learn. It also matters later in this guide: in [step 17](../07-workloads/17-portfolio.md) I build my own image on GitHub Actions for amd64, because that is what the server is.

## Background: why RAM and storage are the real difference

**Soldered versus slotted RAM.** On the Pi, memory is a chip on the board. On a mini PC it is a stick in a SO-DIMM slot. Step 1 made the case that RAM is the resource a homelab runs out of first; the Pi's RAM is fixed the day you buy it, and the mini PC's can be doubled or quadrupled with a screwdriver.

**microSD versus NVMe.** A microSD card was designed for cameras: write a photo, read it back occasionally. A server running databases and logs writes constantly, in small pieces, all day. microSD cards are slow at that and **wear out** under it, which is the most common way a Pi server dies. A mini PC boots from an NVMe SSD that is built for exactly that workload. The Pi 5 can be given an NVMe drive, but through an add-on board that is one more thing to buy and mount.

## Side by side

| Feature | Raspberry Pi 5 | Used mini PC (OptiPlex Micro, ThinkCentre Tiny, HP Mini) |
|---|---|---|
| Best for | GPIO, sensors, single light services | Kubernetes, many containers, file and media servers |
| CPU | ARM | x86 (every container image works) |
| RAM | Fixed, can't upgrade | Upgradeable (up to 64GB on many models) |
| Storage | microSD/USB; NVMe needs an add-on | Built-in NVMe slot (some also have a 2.5" bay) |
| Power use | Very low | Low |
| Real cost | Board + power supply + case + cooler + storage adds up | Often a better deal refurbished |

### What "real cost" means

Compare what you must buy to end up with a working, cased, cooled computer with a fast disk:

| Raspberry Pi 5 | Used mini PC |
|---|---|
| The board | The machine, with CPU, RAM, SSD, case, fan and network already inside |
| Official power supply | Power brick (confirm it is included, see [step 5](../02-hardware/05-buying-the-machine.md)) |
| Case | |
| Active cooler | |
| microSD card, or an NVMe HAT plus an NVMe SSD | |
| micro-HDMI cable or adapter for install day | DisplayPort cable or adapter for install day |

The Pi's board price is the headline; the column underneath it is what you actually pay. The mini PC's price is close to the whole bill.

Power use is the one row where the Pi genuinely wins. A Pi idles at a few watts; a T-series mini PC idles at roughly ten to fifteen and draws more under load. Over a year that is a real but small difference on an electricity bill, and it is the price of having a machine that can run everything.

## Why

**My choice:** a used mini PC. The apps I wanted need more RAM than a Pi offers, and I wanted room to upgrade.

Against my goals from step 1, one by one:

| My goal | Pi | Mini PC |
|---|---|---|
| CKAD practice: follow any tutorial, pull any image | Some images will not exist for arm64 | Every image works |
| Prometheus + Grafana now | Fits | Fits |
| Paperless-ngx, Nextcloud, Immich later | Needs more RAM than the board has, and cannot be upgraded | 8GB now, add a stick later |
| Photo storage later | NVMe only via an add-on board; USB otherwise | 2.5-inch bay inside the case |
| KSh 40,000 budget | Board plus every accessory, imported | One refurbished unit, locally available |

### When a Pi is the right answer

I want to be fair to the Pi, because it is a great machine for the job it was built for:

| Goal | Why a Pi fits |
|---|---|
| Pi-hole or AdGuard as the only service | One small process; a Pi runs it on a fraction of its power |
| A WireGuard VPN endpoint | Same |
| Home Assistant with sensors | GPIO, low power, always on, and the community is built around it |
| Electronics, robotics, a weather station | Nothing else has the pins |
| A second, cheap node to learn multi-node scheduling | A Pi can join a K3s cluster as an agent later |

If every row in your step 1 table landed on "Raspberry Pi", buy one and enjoy it. This guide is for the other rows.

## When

Right after step 1 and before you look at prices. The kind of machine has to be settled before the model, because the shopping lists are completely different.

## Challenges and solutions

No purchase happened on this page, so nothing went wrong, but these are the traps on each side:

| Trap | What happens | How to avoid it |
|---|---|---|
| Comparing the Pi's board price to a mini PC's full price | The Pi looks half the cost | Add up the Pi's accessory column first |
| Buying a Pi for a RAM-hungry app | Immich or Nextcloud crawls or gets OOM-killed, and there is no upgrade path | Fixed RAM means buy for the final goal or do not buy |
| Running a Pi server from a microSD card | It gets slow, then corrupts after months of constant writes | NVMe via a HAT, or a USB SSD, from day one |
| Expecting every container image to run on ARM | `exec format error` in the middle of a tutorial | Check the image's tags for `arm64` before you depend on it, or use x86 |
| Buying a mini PC for an electronics project | No GPIO, no way to add it cleanly | That row of step 1's table says Pi for a reason |

## Lessons

> 💡 **What I learned:** a Pi looks cheap until you add the official power supply, case, cooling and NVMe adapter. For running several services, a refurbished office mini PC gives more for the money. Pis are great, just for a different goal.

## Glossary

| Term | Meaning |
|---|---|
| SBC | Single-board computer: CPU, RAM and ports on one bare board, like a Raspberry Pi |
| 1L PC / tiny-mini-micro | One-litre office desktops: Dell OptiPlex Micro, Lenovo ThinkCentre Tiny, HP ProDesk/EliteDesk Mini |
| ARM / ARM64 / `aarch64` | The CPU family in phones, Macs and Pis |
| x86 / x86-64 / `amd64` | The CPU family in Intel and AMD PCs and most servers |
| Instruction set | The vocabulary of operations a CPU understands; programs built for one do not run on another |
| Multi-arch image | A container image published for several architectures; the registry serves the right one |
| `exec format error` | What you see when a container built for one architecture is run on another |
| SO-DIMM | The small RAM stick used in laptops and mini PCs; slotted, so replaceable |
| Soldered RAM | Memory fixed to the board at the factory; cannot be changed |
| microSD | Flash card designed for cameras; slow at and worn out by constant small writes |
| HAT | An add-on board that stacks on a Pi's header, for example to add an NVMe slot |
| M.2 / NVMe | The slot and the protocol for fast SSDs that plug straight into PCIe |
| 2.5-inch bay | Space inside a case for a laptop-sized SATA drive |
| GPIO | The 40 pins on a Pi for wiring sensors and electronics |
| K3s agent | A machine that joins an existing K3s cluster as an extra node ([step 3](03-one-node-or-cluster.md)) |

---

[◀ Previous: Start With Your Goal](01-goals-decide-specs.md) · [🏠 Home](../../README.md) · [Next: One Node or a Cluster? ▶](03-one-node-or-cluster.md)
