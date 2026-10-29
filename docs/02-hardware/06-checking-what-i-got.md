# 6. Checking What I Got

The machine arrives with Windows on it. That Windows install is about to be erased, but for the next ten minutes it is the most convenient hardware inspection tool you will ever have: every spec is a few clicks away, with no commands to learn. This page is those clicks, what the numbers mean, and the plan I made once I had them.

## What

Confirm that what is in the box matches what was paid for in [step 5](05-buying-the-machine.md): model, CPU, RAM, storage, network. Then decide what to run now and what to run after upgrades.

## Background: why check while Windows is still there

Three reasons, in order of importance:

1. **Disputes are easier before the wipe.** If the RAM is 4GB or the SSD is SATA, you want to know while the machine is still in the state the seller sold it in.
2. **A working Windows session is a free hardware test.** The display output works, the keyboard works, the disk reads, the Ethernet port gets an address. If any of that fails later, you know it is the install and not the hardware.
3. **It takes two minutes.** The Linux equivalents (`lscpu`, `free -h`, `lsblk`) are easy too, but by then the decision to wipe has already been made.

## Background: what the numbers mean

The results table below has five rows. Here is how to read each one.

### CPU: "i5-9500T, 6 cores, up to 4.4GHz"

Six **cores** means six things at once ([step 1](../01-planning/01-goals-decide-specs.md)). This chip has no Hyper-Threading, so 6 cores is also 6 threads. The **T** means the 35W low-power variant. "Up to 4.4GHz" is the **turbo** frequency, the speed one core can briefly reach when the others are idle and the chip is cool; the **base** frequency, which is what it sustains with all cores busy, is lower. For a server with many idle services, turbo is what you use most of the time: one thing gets busy, one core speeds up.

### RAM: "8GB DDR4"

**DDR4** is the generation of memory; the machine takes DDR4 **SO-DIMM** sticks and nothing else. Windows may report something like "8.00 GB (7.8 GB usable)". The missing part is reserved by the built-in graphics (the iGPU), which borrows system memory. That is normal and not a fault. The number that matters for upgrades is not the amount but the **slots used**, which is why the callout at the end sends you to Task Manager.

### Storage: "256GB NVMe SSD (Samsung)"

**NVMe** confirms it is the fast kind of SSD on PCIe, not a SATA drive in an M.2 slot ([step 1](../01-planning/01-goals-decide-specs.md)). Windows will show the disk as about **238GB**, not 256. Drive makers count in decimal gigabytes (1,000,000,000 bytes); Windows counts in binary gibibytes (1,073,741,824 bytes) and still labels them GB. 256 decimal GB is 238 binary GB. Nothing is missing. A Samsung drive is a common OEM fitment in Dell business machines.

### Network: "Gigabit Ethernet, no Wi-Fi card"

The one Ethernet port on the back runs at 1 Gbit/s. **No Wi-Fi card** is the normal state for an office Micro: the M.2 slot a Wi-Fi card would sit in is simply empty. For a server that will be wired anyway, this costs nothing ([step 7](07-display-keyboard-and-network.md)).

### Two more things worth reading while you are there

- **BIOS mode: UEFI** and **partition style: GPT**. Modern machines boot through **UEFI** firmware (the successor to the old BIOS) from a disk laid out with a **GPT** partition table (the successor to MBR). Those are the two settings the USB-flashing tool asks for in [step 8](../03-ubuntu-server/08-making-the-install-usb.md), so confirming them now means no guessing then.
- **Secure Boot: On**. Ubuntu supports it, and [step 9](../03-ubuntu-server/09-bios-settings.md) leaves it on.

## Background: the Windows tools, and what each one shows

| Tool | How to open it | What to read there |
|---|---|---|
| **System Information** (`msinfo32`) | `Win + R`, type `msinfo32`, Enter | System Model, Processor (full model string, cores), Installed Physical Memory, BIOS Version, **BIOS Mode**, **Secure Boot State** |
| **Disk Management** | Right-click the Start button → Disk Management (or `Win + R`, `diskmgmt.msc`) | Number of disks and their sizes; right-click the disk → Properties → Volumes shows the **partition style** (GPT) |
| **Task Manager → Performance** | `Ctrl + Shift + Esc`, then Performance | **Memory:** Slots used, Form factor, Speed. **Disk:** the drive's name (shows the make and that it is an SSD). **Ethernet:** the adapter is present and connected. |
| **Device Manager** | Right-click Start → Device Manager | **Network adapters:** one Ethernet controller and no wireless entry means no Wi-Fi card. **Storage controllers / Disk drives:** NVMe shows up as such. |

## How

### 1. System Information

Before wiping Windows, I checked the specs: `Win + R` → `msinfo32`, then Disk Management.

`Win + R` opens the Run box; `msinfo32` is the System Information app. It takes a few seconds to populate. Read the **System Summary** page: System Model confirms the exact machine, Processor gives the full CPU string with the core count, Installed Physical Memory gives the RAM, and BIOS Mode should say UEFI.

### 2. Disk Management

Opens a map of every disk in the machine. You should see one disk of around 238GB with a few Windows partitions on it. Right-click the disk's label on the left → **Properties → Volumes** and read the partition style; it should be GPT. Note that this is the **same disk you will tell the Ubuntu installer to use** in [step 10](../03-ubuntu-server/10-installing-ubuntu-server.md), so knowing its name and size now helps you pick the right one then.

### 3. Task Manager, for the slots

`Ctrl + Shift + Esc` → **Performance → Memory**. The bottom right of that panel says **Slots used: 1 of 2** or **2 of 2**, with the form factor (SO-DIMM) and speed. This is the single most valuable number on the page for planning the first upgrade. While there, click **Disk** to see the drive's name and **Ethernet** to see the adapter.

### 4. Device Manager, for the network

Expand **Network adapters**. An Ethernet controller and nothing wireless means exactly what the seller's spec sheet should have said: wired only.

### 5. Write it down

| Component | Result |
|---|---|
| Model | Dell OptiPlex 7070 Micro |
| CPU | Intel Core **i5-9500T** (6 cores, up to 4.4GHz) |
| RAM | **8GB** DDR4 |
| Storage | **256GB** NVMe SSD (Samsung) |
| Network | Gigabit Ethernet, **no Wi-Fi card** |

Everything matched Option B from step 5. This table is also the "My setup at a glance" section of the README, so it is worth getting it right once.

## The plan: apps in stages

I decided to start with 8GB/256GB and upgrade later, adding apps in stages:

| Stage | What runs |
|---|---|
| **Now (8GB)** | Ubuntu + K3s, portfolio, Prometheus + Grafana (with limits), CKAD practice |
| **After RAM upgrade (24GB+)** | Paperless-ngx, Nextcloud, Immich |
| **After a 1TB SSD** (the 7070 Micro has a 2.5" bay) | Photo and file storage |

Why it is staged this way:

| Stage | Reasoning |
|---|---|
| Now | Everything in this row is light or can be made light. "With limits" means Prometheus and Grafana get Kubernetes resource limits so they cannot grow into the whole 8GB ([step 18](../08-observability/18-prometheus-grafana.md)). CKAD practice is small test workloads that come and go. |
| After RAM | Paperless-ngx, Nextcloud and Immich each bring a database, a cache and worker processes, and Immich runs machine-learning models. Together they want more than what is left of 8GB after the system and monitoring. One stick in the free slot is the fix, and that stick is a later purchase. |
| After a 1TB SSD | Photos and documents are big, and the 256GB NVMe is for the OS and apps. The 7070 Micro's **2.5-inch bay** takes a SATA SSD inside the case, so the data drive does not dangle off USB. **Backups come before this stage**, because this is the first stage that holds data I cannot re-download (see the roadmap in the [README](../../README.md)). |

Staging is also a hedge: if the homelab turns out to be less fun than planned, the money for RAM and the second SSD was never spent.

## When

The day the machine arrives, before the install USB from [step 8](../03-ubuntu-server/08-making-the-install-usb.md) goes anywhere near it. Ten minutes, once.

## Challenges and solutions

Everything matched, so there was nothing to send back. These are the things that look wrong and are not, and the things that are easy to forget:

| Looks like a problem | What is actually going on | What to do |
|---|---|---|
| Windows shows "7.8 GB usable" of 8GB | The integrated graphics reserve a slice of system RAM | Nothing. Ubuntu Server will see the same amount, minus a little for the same reason. |
| The disk shows 238GB, not 256 | Decimal GB on the box, binary GB in Windows | Nothing. The drive is the size you paid for. |
| "Slots used: 2 of 2" on an 8GB machine | Two 4GB sticks. The upgrade to 16GB costs two sticks, not one. | Worth knowing now, and worth raising with the seller if they said otherwise |
| The SSD is listed without "NVMe" anywhere | It may be a SATA M.2 drive, which fits the same slot but is several times slower | Check Device Manager → Storage controllers; an NVMe drive has an NVMe controller entry |
| Not writing down the BIOS version and service tag | You need the BIOS page in step 9 and may want to check for firmware updates later | Note both from `msinfo32` and the sticker, and keep them out of anything public |

## Lessons

> 💡 **What I learned:** check the specs while Windows is still there; it takes two minutes. Also check **Task Manager → Performance → Memory → Slots used**: with 1 of 2 used, one extra stick upgrades you.

## Glossary

| Term | Meaning |
|---|---|
| `msinfo32` | Windows System Information: model, CPU, RAM, BIOS mode and Secure Boot state in one window |
| Disk Management | The Windows tool that shows disks, sizes and partition style |
| Task Manager → Performance | Live hardware panels; the Memory panel shows RAM slots used |
| Device Manager | The list of every hardware device Windows can see, grouped by type |
| Base / turbo frequency | The clock a CPU sustains with all cores busy / the higher clock one core can reach briefly |
| Hyper-Threading | Two threads per core; the i5-9500T does not have it, so 6 cores is 6 threads |
| iGPU | Graphics built into the CPU; it reserves some system RAM, hence "usable" being less than installed |
| GB vs GiB | Decimal (10⁹) versus binary (2³⁰) gigabytes; 256GB on the box is 238GB in Windows |
| NVMe vs SATA M.2 | Two kinds of SSD that fit the same slot; NVMe is several times faster |
| UEFI | Modern PC firmware, successor to legacy BIOS; what the install USB must be made for |
| GPT | Modern partition table, successor to MBR; pairs with UEFI |
| Secure Boot | Firmware feature that only boots signed operating systems; Ubuntu is signed |
| 2.5-inch bay | Space inside the case for a laptop-sized SATA drive; where the 1TB data SSD will go |
| Resource limits | Kubernetes caps on how much CPU and RAM a container may use |

---

[◀ Previous: Buying the Machine](05-buying-the-machine.md) · [🏠 Home](../../README.md) · [Next: Display, Keyboard and Network ▶](07-display-keyboard-and-network.md)
