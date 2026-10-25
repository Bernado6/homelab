# Troubleshooting

Problems I hit along the way, and what fixed them. Ordered roughly by when they came up.

| Problem | Cause / fix | Step |
|---|---|---|
| No Wi-Fi in Windows | OptiPlex Micros usually have no Wi-Fi card. Use Ethernet. | [7](02-hardware/07-display-keyboard-and-network.md) |
| Chrome blocks Rufus | "Uncommon download". `Ctrl + J` → Keep. Check the signature is from Akeo Consulting. | [8](03-ubuntu-server/08-making-the-install-usb.md) |
| F12 → USB goes back to Windows | 1) Check the flash drive has Ubuntu files (`boot`, `casper`, `EFI`); if not, re-flash, trying **DD Image mode**. 2) Use a **rear** USB port. 3) Make sure you selected the USB entry. 4) Last resort: temporarily disable Secure Boot. | [10](03-ubuntu-server/10-installing-ubuntu-server.md) |
| Installer can't see the SSD | BIOS → SATA Operation → **AHCI**. | [9](03-ubuntu-server/09-bios-settings.md) |
| Ubuntu only uses ~100GB | The LVM default. Untick LVM during install, or extend the volume later. | [10](03-ubuntu-server/10-installing-ubuntu-server.md) |
| Installer asks for a proxy | Leave it blank at home. | [10](03-ubuntu-server/10-installing-ubuntu-server.md) |
| `hostname -i` shows `127.0.1.1` | Use `hostname -I` or `ip -4 addr`. | [11](03-ubuntu-server/11-first-login-and-ssh.md) |
| Server IP changed | Set a DHCP reservation in the router. | [11](03-ubuntu-server/11-first-login-and-ssh.md) |
| `helm-install-traefik` pod restarts after K3s install | Normal at first boot. It retries until the cluster is ready, then shows **Completed**. | [12](04-kubernetes/12-installing-k3s.md) |

---

[◀ Previous: My First App](04-kubernetes/13-first-app.md) · [🏠 Home](../README.md) · [Next: Lessons Learned ▶](lessons-learned.md)
