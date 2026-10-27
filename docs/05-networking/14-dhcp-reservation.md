# 14. A Fixed IP for the Server (DHCP Reservation)

## What

The router hands the server an IP address (`192.168.100.22` in my case) from its DHCP pool. That lease can change after a reboot, a power cut, or whenever the router feels like reshuffling. A **DHCP reservation** (Huawei calls it a *static IP binding*) tells the router: *"the device with this MAC address always gets this IP."*

The server keeps using DHCP as normal. Nothing changes on Ubuntu. The router just always answers with the same address.

## Why

Everything you build from here on points at that IP:

| Depends on the IP | What breaks if it changes |
|---|---|
| `ssh bernado@192.168.100.22` | You have to find the new IP with a monitor plugged in |
| `/etc/hosts` entry on the laptop (`quantum`) | Name resolves to the wrong machine |
| Browser bookmarks, `http://<IP>` for apps | Dead links |
| A kubeconfig copied to the laptop | `kubectl` can't reach the API server |
| Future HTTPS / Ingress host rules | Certificates and DNS point at the wrong place |

I hit this in [step 13](../04-kubernetes/13-first-app.md): `http://quantum:8000` failed from the laptop with `DNS_PROBE_FINISHED_NXDOMAIN` because the laptop has no idea what "quantum" is. The fix is a hosts-file entry, and that entry is only safe once the IP is pinned.

### Why a reservation and not a static IP on the server?

| | DHCP reservation (router) | Static IP (netplan on the server) |
|---|---|---|
| Where it's configured | Router admin page | `/etc/netplan/*.yaml` on Ubuntu |
| Risk of two devices getting the same IP | None, the router owns the pool | Possible if you pick an address inside the DHCP pool |
| If you move the box to another network | Just works, gets a new lease | Won't come up until you edit netplan |
| Survives a router reset | ❌ You redo the reservation | ✅ |

**My choice:** reservation first. It's one screen in the router, zero changes on the server, and no risk of IP conflicts. If I ever lose the router config, I can fall back to netplan.

## When

Do this **right after first SSH** ([step 11](../03-ubuntu-server/11-first-login-and-ssh.md)) and before anything else learns the IP. I did it later than I should have, and the first thing that bit me was the hostname not resolving from the laptop.

## How

My router is a **Huawei EchoLife HG8145V5**, the standard Safaricom Home Fibre box. The menu names below are from that firmware; other routers call the same thing *Address Reservation*, *DHCP Binding* or *Static Lease*.

### 1. Get the server's MAC address and gateway

On the server:

```bash
ip -br link | grep eno1          # MAC of the wired interface
ip route | grep default          # gateway = the router
```

```text
eno1  UP  xx:xx:xx:xx:xx:xx <BROADCAST,MULTICAST,UP,LOWER_UP>
default via 192.168.100.1 dev eno1 proto dhcp src 192.168.100.22 metric 100
```

Copy the MAC. The gateway is the router's address, `192.168.100.1`.

### 2. Reach the router admin page

This is where it got interesting. `http://192.168.100.1` from my laptop **timed out**, even though the laptop could ping the router. From the server, `curl -I http://192.168.100.1` returned `200 OK`. The difference: the server is on a **wired** port, the laptop on **Wi-Fi**, and this firmware only serves the admin page to wired clients.

Rather than find a cable, I tunnelled through the server with SSH. The first attempt forwarded only port 80, and the router's login page immediately bounced the browser to `https://localhost`, dropping the port. So the tunnel has to carry **both** 80 and 443, and 443 has to land on the laptop's real port 443 so the redirect works:

```bash
# on the laptop; sudo because port 443 is privileged on macOS
sudo ssh -L 443:192.168.100.1:443 -L 8080:192.168.100.1:80 bernado@192.168.100.22
```

Keep that session open and browse to **`https://localhost`**. Accept the certificate warning (the router's cert is self-signed) and the Huawei login page appears.

If your laptop is wired, or your router doesn't have this restriction, just open `http://192.168.100.1` and skip the tunnel.

### 3. Log in

The HG8145V5 asks for three things, all printed on the label on the back of the router:

| Field | Where on the label |
|---|---|
| User Name | `username:` line |
| Password | `password:` line |
| SN | `SN:` line, a 16-character code starting `4857` |

The SN must match exactly. Mine was hard to read from a photo; `z` vs `2` cost me one attempt.

> ⚠️ That label also shows the Wi-Fi key. Don't commit a photo of it to the repo.

### 4. Bind the MAC to the IP

**Advanced → LAN → DHCP Static IP Configuration → New.** Enter the MAC from step 1 and `192.168.100.22`, then **Apply**.

I reserved the address the server **already had**, so nothing changed immediately and there was no chance of a conflict with another device.

### 5. Renew the lease and verify

On the server, renew without rebooting:

```bash
sudo networkctl renew eno1
hostname -I
```

Then reboot once and check again:

```bash
sudo reboot
# wait, then from the laptop:
ssh bernado@192.168.100.22 hostname -I
```

```text
192.168.100.22 10.42.0.0 10.42.0.1
```

The first address is the one that matters. The two `10.42.x.x` addresses are the K3s pod network interfaces (`flannel.1` and `cni0`) that appeared in [step 12](../04-kubernetes/12-installing-k3s.md); they live only inside the server.

### 6. Give the server a name on the laptop

With the IP pinned, a hosts entry on the Mac is safe:

```bash
sudo sh -c 'echo "192.168.100.22  quantum" >> /etc/hosts'
```

From the laptop, `ssh bernado@quantum` and `http://quantum` now work. This is laptop-only; other devices on the network still need the IP.

## Challenges and solutions

| Challenge | What was going on | Solution |
|---|---|---|
| `http://192.168.100.1` timed out from the laptop | Router only serves its admin page to **wired** clients; laptop was on Wi-Fi. Ping worked, HTTP didn't. | SSH tunnel through the wired server (or plug the laptop in). |
| Tunnel on port 80 only: browser jumped to `https://localhost` and refused | Login page redirects to HTTPS using the host name without the port. | Forward 443 too, onto the laptop's own port 443: `sudo ssh -L 443:192.168.100.1:443 …` |
| Certificate warning on `https://localhost` | Router cert is self-signed and issued for its own IP. | Accept it. It's your router on your LAN. |
| Login form wants a serial number | HG8145V5 firmware requires the SN as a third factor. | It's on the back label, 16 hex characters starting `4857`. |
| Misread a character on the label | Photo was blurry; `z` and `2` look alike. | Try the obvious substitutions; fall back to `admin` as the password. |
| `hostname -I` shows three addresses | Two are K3s internal interfaces. | Only the `192.168.100.x` one is the LAN address. |

## Lessons

> 💡 **What I learned:**
> - **Pin the IP before anything depends on it.** SSH, hosts files, bookmarks and kubeconfigs all hard-code it. Do this on day one, right after the first SSH login.
> - **The server's hostname means nothing to the rest of the network.** `quantum` works on the server because Ubuntu puts it in its own `/etc/hosts`. Your laptop and router have never heard of it. Either add the name to each client, or run your own DNS later (Pi-hole / AdGuard is a classic homelab step).
> - **An always-on wired box is a useful tool in itself.** The router refused the laptop but trusted the server. SSH port forwarding turned the server into a doorway. This is the same `-L` flag we used for port-forward in step 13, just pointed somewhere else.
> - **"Can ping but can't connect" means a service-level block, not a network one.** Ping proves the path; the timeout on port 80 proved the router was choosing not to answer.
> - **Reservation beats static when you don't control the DHCP pool.** The router already tracks who has which address; let it keep doing that.
> - **Reserve the address the box already has.** Zero disruption, nothing to reconnect, no conflicts.
> - **Write the MAC address down** somewhere private. You'll need it again if the router is ever reset.

---

[◀ Previous: My First App](../04-kubernetes/13-first-app.md) · [🏠 Home](../../README.md) · [Next: Running kubectl From the Laptop ▶](../06-working-with-the-cluster/15-kubectl-from-laptop.md)
