# 6. Checking What I Got

Before wiping Windows, I checked the specs: `Win + R` → `msinfo32`, then Disk Management.

| Component | Result |
|---|---|
| Model | Dell OptiPlex 7070 Micro |
| CPU | Intel Core **i5-9500T** (6 cores, up to 4.4GHz) |
| RAM | **8GB** DDR4 |
| Storage | **256GB** NVMe SSD (Samsung) |
| Network | Gigabit Ethernet, **no Wi-Fi card** |

I decided to start with 8GB/256GB and upgrade later, adding apps in stages:

| Stage | What runs |
|---|---|
| **Now (8GB)** | Ubuntu + K3s, portfolio, Prometheus + Grafana (with limits), CKAD practice |
| **After RAM upgrade (24GB+)** | Paperless-ngx, Nextcloud, Immich |
| **After a 1TB SSD** (the 7070 Micro has a 2.5" bay) | Photo and file storage |

> 💡 **What I learned:** check the specs while Windows is still there; it takes two minutes. Also check **Task Manager → Performance → Memory → Slots used**: with 1 of 2 used, one extra stick upgrades you.

---

[◀ Previous: Buying the Machine](05-buying-the-machine.md) · [🏠 Home](../../README.md) · [Next: Display, Keyboard and Network ▶](07-display-keyboard-and-network.md)
