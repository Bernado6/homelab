# 0. Before You Start: What to Know

A "homelab" is just a computer at home that you use to learn and to run your own services. Before spending anything, answer these questions. They matter more than which brand you buy.

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

> 💡 **What I learned:** I started by asking "Pi or mini PC?", but the real first question was *"what do I want to do with it?"*. Once I wrote down my goals, the hardware choice became obvious.

---

[🏠 Home](../../README.md) · [Next: Start With Your Goal ▶](01-goals-decide-specs.md)
