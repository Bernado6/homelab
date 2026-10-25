# 2. Raspberry Pi vs Mini PC

| Feature | Raspberry Pi 5 | Used mini PC (OptiPlex Micro, ThinkCentre Tiny, HP Mini) |
|---|---|---|
| Best for | GPIO, sensors, single light services | Kubernetes, many containers, file and media servers |
| CPU | ARM | x86 (every container image works) |
| RAM | Fixed, can't upgrade | Upgradeable (up to 64GB on many models) |
| Storage | microSD/USB; NVMe needs an add-on | Built-in NVMe slot (some also have a 2.5" bay) |
| Power use | Very low | Low |
| Real cost | Board + power supply + case + cooler + storage adds up | Often a better deal refurbished |

**My choice:** a used mini PC. The apps I wanted need more RAM than a Pi offers, and I wanted room to upgrade.

> 💡 **What I learned:** a Pi looks cheap until you add the official power supply, case, cooling and NVMe adapter. For running several services, a refurbished office mini PC gives more for the money. Pis are great, just for a different goal.

---

[◀ Previous: Start With Your Goal](01-goals-decide-specs.md) · [🏠 Home](../../README.md) · [Next: One Node or a Cluster? ▶](03-one-node-or-cluster.md)
