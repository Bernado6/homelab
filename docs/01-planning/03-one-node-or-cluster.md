# 3. One Node or a Cluster?

| | One powerful node | 3-node cluster |
|---|---|---|
| ✅ Pros | Simple; all RAM in one place; one machine to manage | High availability; feels like production; teaches scheduling and node failures |
| ❌ Cons | If it's off, everything is off | Needs shared storage (Longhorn/NFS); more networking; RAM split across machines |
| 💰 Cost | One machine | Three machines |
| 🎯 Best for | CKAD, self-hosting, first homelab | CKA, high-availability practice |

**My choice:** one node, with the option to add more later. K3s makes joining extra machines easy.

> 💡 **What I learned:** a cluster is more interesting, but one node gets you learning on day one. You can always grow.

---

[◀ Previous: Raspberry Pi vs Mini PC](02-raspberry-pi-vs-mini-pc.md) · [🏠 Home](../../README.md) · [Next: Budget, Specs and Software Choices ▶](04-budget-and-software-choices.md)
