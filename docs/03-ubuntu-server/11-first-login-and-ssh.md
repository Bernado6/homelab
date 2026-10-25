# 11. First Login and SSH

Update everything:

```bash
sudo apt update && sudo apt upgrade -y
```

Find the server's IP:

```bash
hostname -I      # capital I
# or
ip -4 addr       # look for the inet line under the Ethernet entry (e.g. eno1)
```

From a laptop on the same network:

```bash
ssh <username>@<SERVER_IP>
```

From this point on I stopped using the monitor and did everything from my laptop, so I could copy and paste commands.

> 💡 **What I learned:**
> - `hostname -i` (lowercase) shows `127.0.1.1`, which is the machine talking to itself. Use **capital `-I`**.
> - **Reserve the server's IP in your router** (DHCP reservation) so it doesn't change after a reboot.
> - Typing long commands by hand on the server invites typos. SSH in and paste.

---

[◀ Previous: Installing Ubuntu Server](10-installing-ubuntu-server.md) · [🏠 Home](../../README.md) · [Next: Installing Kubernetes (K3s) ▶](../04-kubernetes/12-installing-k3s.md)
