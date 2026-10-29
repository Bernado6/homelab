# 14. A Fixed IP for the Server (DHCP Reservation)

This is a small change in the router, but it is the foundation that every later step stands on. It also turned into my first real lesson about how a home network hands out addresses, why a hostname that works on the server means nothing to the laptop, and how an SSH tunnel can get you into a page the router refuses to show you. If you have never looked inside a router's admin page before, read the background sections first; the How will make much more sense.

## What

The router hands the server an IP address (`192.168.100.22` in my case) from its DHCP pool. That lease can change after a reboot, a power cut, or whenever the router feels like reshuffling. A **DHCP reservation** (Huawei calls it a *static IP binding*) tells the router: *"the device with this MAC address always gets this IP."*

The server keeps using DHCP as normal. Nothing changes on Ubuntu. The router just always answers with the same address.

After this step:

- The server is at `192.168.100.22` today, tomorrow, and after the next power cut.
- The laptop can call the server `quantum` through one line in `/etc/hosts`, safely.
- Everything built later (kubeconfig, Ingress names, certificates, DNS) can hard-code that address without fear.

## Background: how a device gets an IP address

### Every network card has a MAC address

Before a device has an IP address it already has a **MAC address** (Media Access Control). It is a 48-bit identifier burned into the network interface at the factory, written as six pairs of hex digits, like `xx:xx:xx:xx:xx:xx`. The first three pairs identify the manufacturer, the last three are unique to that card.

| | MAC address | IP address |
|---|---|---|
| Set by | The hardware maker | The network you plug into |
| Changes when | Never (unless you deliberately override it) | You join a different network, or the lease changes |
| Scope | Only meaningful on the local network segment | Routable between networks |
| Looks like | `xx:xx:xx:xx:xx:xx` | `192.168.100.22` |

The MAC is what the router uses to recognise "the same device as last time". That is why a DHCP reservation is keyed on it: the MAC is the only identity a device has before it is given an address.

A wired port and a Wi-Fi card are two different interfaces with two different MACs. The OptiPlex only has the wired one, `eno1`, which keeps things simple.

### Private IP ranges

Three blocks of IPv4 addresses are reserved for private networks ([RFC 1918](https://www.rfc-editor.org/rfc/rfc1918)). They are never routed on the public internet, which is why millions of homes can all use the same addresses without colliding:

| Range | Shorthand | Where you meet it |
|---|---|---|
| `10.0.0.0` to `10.255.255.255` | `10.0.0.0/8` | Large offices, and the K3s pod and Service networks (`10.42.x.x`, `10.43.x.x`) |
| `172.16.0.0` to `172.31.255.255` | `172.16.0.0/12` | Docker's default bridge network |
| `192.168.0.0` to `192.168.255.255` | `192.168.0.0/16` | Almost every home router, including mine: `192.168.100.0/24` |

The `/24` after `192.168.100.0` means the first 24 bits are fixed and the last 8 vary, so this network has addresses `192.168.100.1` through `192.168.100.254`. The router takes `.1`. The rest is available for the DHCP pool and for reservations. This is also why a private LAN address can appear in a public repo: it tells nobody anything about where I am.

### The DHCP conversation

**DHCP** (Dynamic Host Configuration Protocol) is how a device with no address gets one. The router runs the DHCP **server**; every device runs a DHCP **client**. When the server boots or plugs in, four messages go back and forth. People remember them as **DORA**:

```text
server (no IP yet)                              router (DHCP server, 192.168.100.1)
      │                                                   │
      │  1. DISCOVER  "anyone out there? my MAC is xx:xx…" │   broadcast to the whole LAN
      │──────────────────────────────────────────────────►│
      │                                                   │
      │  2. OFFER     "you can have 192.168.100.22"        │
      │◄──────────────────────────────────────────────────│
      │                                                   │
      │  3. REQUEST   "yes please, 192.168.100.22"         │
      │──────────────────────────────────────────────────►│
      │                                                   │
      │  4. ACK       "it's yours for N hours. gateway is  │
      │               .1, DNS is …, subnet mask is …"      │
      │◄──────────────────────────────────────────────────│
```

The ACK carries more than the address. It also tells the device the **gateway** (where to send traffic for other networks), the **subnet mask**, and the **DNS server** to use. That last item is why the router is involved again in [step 21](../10-dns/21-local-dns-adguard.md): changing the DNS server that DHCP hands out changes DNS for every device in the house.

### Leases, and why addresses drift

The address is a **lease**, not a gift. It is valid for a fixed time, and the client renews it halfway through. If the device is off when the lease expires, the router is free to give that address to someone else. Next time the device comes back it goes through DORA again and may get a different one.

In practice the router tends to hand back the same address if it is still free, which is why a homelab can run for weeks on a plain lease and then break after one unlucky power cut. The drift is rare enough to forget about and common enough to bite.

### Reservations: DHCP with a memory

A **reservation** is a line in the router's table: *MAC `xx:xx:xx:xx:xx:xx` always gets `192.168.100.22`*. The conversation above still happens, every message still flows, but the OFFER is no longer a free pick from the pool. The router looks up the MAC first.

Nothing on the server knows the difference. It still runs a DHCP client, still renews, still asks politely. It just always gets the same answer. That is the whole appeal: zero configuration on the box, and the router keeps full ownership of the address pool.

## Background: why the server's hostname means nothing to the laptop

During the Ubuntu install I named the server `quantum`. On the server, `ping quantum` works, `curl http://quantum` works, and it is tempting to assume the whole network knows the name. It does not.

A hostname is a label a machine gives **itself**. Ubuntu writes it to `/etc/hostname` and adds a line to the server's own `/etc/hosts` (`127.0.1.1 quantum`), so programs on that machine can resolve it. Nothing announces that name to the router or to any other device. The laptop has its own `/etc/hosts` and its own idea of DNS, and neither has ever heard of `quantum`.

This is exactly what happened in step 13: `http://quantum:8000` from the laptop failed with `DNS_PROBE_FINISHED_NXDOMAIN`. The name is not a network fact, it is a server-local fact.

The three ways to fix it, in increasing order of effort:

| Option | How | Reach |
|---|---|---|
| Use the IP | `ssh bernado@192.168.100.22` | Works everywhere, nothing to maintain, only safe once the IP is fixed |
| A hosts entry per client | `192.168.100.22  quantum` in the laptop's `/etc/hosts` | That one laptop only. Phones can't do this. |
| Run your own DNS | A DNS server on the LAN that answers the name for everyone | Every device. This is [step 21](../10-dns/21-local-dns-adguard.md). |

This step does the second one, and it is only safe **because** of the reservation. A hosts entry is a hard-coded IP; if the IP drifts, the name points at the wrong machine, or at nothing.

## Background: SSH local port forwarding

Section 2 of the How uses `ssh -L`. It is the same flag step 13 used to reach a `port-forward` running on the server, so it is worth understanding properly once.

### What `-L` does

`ssh -L` means **local forward**: "open a port on my machine, and whatever connects to it, carry through the SSH session and deliver it to some address *as seen from the server*". The key phrase is *as seen from the server*. The destination does not have to be the server itself. It can be anything the server can reach.

```text
 laptop (Wi-Fi)                        server (wired)                        router
 ─────────────                         ──────────────                        ──────
 browser                               sshd                                  admin page
   │                                     │                                      │
   │ connects to localhost:443           │                                      │
   ▼                                     │                                      │
 ssh client ═══════ encrypted SSH session (port 22) ════════►│                  │
   -L 443:192.168.100.1:443              │                                      │
                                         │ opens a plain TCP connection          │
                                         │ to 192.168.100.1:443 ───────────────►│
                                         │                                      │
                                         │   the router sees a connection from  │
                                         │   192.168.100.22, a wired client     │
```

The router never sees the laptop. It sees a connection from the server's wired address, which is exactly what it is willing to answer.

### Reading the flag

`-L [local_port]:[destination_host]:[destination_port]`, with the destination resolved **from the server's point of view**:

| Piece of `-L 443:192.168.100.1:443` | Meaning |
|---|---|
| `443` (first) | The port to open on the laptop. The browser connects here. |
| `192.168.100.1` | Where the server should connect on your behalf: the router |
| `443` (second) | The port on the router |

You can pass `-L` as many times as you like in one command. Each one opens another local port. The session stays open as long as the command runs; close the terminal and the tunnels close with it.

### Why 443 has to land on 443

My first attempt forwarded only port 80, to local port 8080. The router's login page answered with a redirect to `https://localhost`, with **no port**. The browser obeyed, tried port 443 on the laptop, found nothing listening, and gave up.

The router builds that redirect from the host name it was asked for (`localhost`) and assumes HTTPS lives on the default port, 443. The only way to make the redirect land somewhere useful is to make sure something **is** listening on the laptop's own port 443, which means the first number in the `-L` must be exactly `443`.

### Why `sudo`

On macOS (and Linux), ports below 1024 are **privileged**: only root can listen on them. Port 443 is below 1024, so the `ssh` command that opens it needs `sudo`. Port 8080 would not, which is why the HTTP half of the tunnel uses 8080 instead of 80.

A detail worth knowing: `sudo ssh` runs as root, so it reads root's SSH config and known_hosts, not yours. Expect a fresh "are you sure you want to continue connecting" prompt the first time, and a password prompt for the server if your key isn't where root looks for it.

## Background: why a router might only answer wired clients

It surprised me that `ping 192.168.100.1` worked from the laptop while `http://192.168.100.1` timed out. The explanation is that **ping and HTTP are different layers**:

| Test | Layer | What a success proves |
|---|---|---|
| `ping 192.168.100.1` | Network (ICMP) | Packets can get to the router and back. The path exists. |
| `http://192.168.100.1` | Application (TCP port 80) | A service on the router is willing to talk to you on that port. |

The router's network stack happily answers pings from anyone on the LAN. Its web server, a separate program, has a policy: serve the admin page only to clients on the wired (LAN) ports, not to clients associated over Wi-Fi. ISP-supplied routers often ship like this as a cheap safety measure, on the theory that anyone who can plug a cable in is already trusted. From the server, which is wired, `curl -I http://192.168.100.1` returned `200 OK` immediately.

So "can ping but can't connect" is a diagnostic in itself. The network is fine. Something is choosing not to answer. That narrows the problem to a firewall rule, a service that isn't running, or, as here, a service policy.

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

There are two places you can pin an address: in the router (a reservation) or on the server itself (a static address in netplan, with DHCP switched off for that interface). Both work. They fail differently.

| | DHCP reservation (router) | Static IP (netplan on the server) |
|---|---|---|
| Where it's configured | Router admin page | `/etc/netplan/*.yaml` on Ubuntu |
| Risk of two devices getting the same IP | None, the router owns the pool | Possible if you pick an address inside the DHCP pool |
| If you move the box to another network | Just works, gets a new lease | Won't come up until you edit netplan |
| Survives a router reset | ❌ You redo the reservation | ✅ |

**My choice:** reservation first. It's one screen in the router, zero changes on the server, and no risk of IP conflicts. If I ever lose the router config, I can fall back to netplan.

The conflict row is the one that matters most for a first-timer. If you set a static `192.168.100.22` on the server but the router's pool still includes `.22`, the router may one day lease `.22` to a phone, and two devices will fight over one address. Symptoms are intermittent and miserable. A reservation can't do that because the router is the only one making decisions.

## How the pieces fit

```text
 laptop (Wi-Fi)                                   server (wired, eno1)
 ──────────────                                   ────────────────────
 /etc/hosts: 192.168.100.22  quantum              DHCP client: "my MAC is xx:xx…"
 ssh bernado@quantum ───────────────────────────► sshd
 sudo ssh -L 443:192.168.100.1:443 … ═══════════► forwards to the router as a wired client
                                                          │
                                                          │ DORA on every boot
                                                          ▼
                                                  router 192.168.100.1
                                                  ────────────────────
                                                  DHCP pool: 192.168.100.x
                                                  reservation: xx:xx… → .22   ◄── this step
                                                  admin page: wired clients only
```

The reservation lives in the router. The hosts entry lives on the laptop. The server is untouched, which is the point.

## When

Do this **right after first SSH** ([step 11](../03-ubuntu-server/11-first-login-and-ssh.md)) and before anything else learns the IP. I did it later than I should have, and the first thing that bit me was the hostname not resolving from the laptop.

Doing it late is not fatal, because you can reserve the address the server already has (see section 4). It just means that anything you set up in between was quietly relying on luck.

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

What each command does:

| Command | What it shows |
|---|---|
| `ip -br link` | Every network interface in brief form: name, state, MAC. `grep eno1` keeps the wired one. The third column is the MAC. |
| `ip route \| grep default` | The default route, meaning "where to send anything not on my own network". `via 192.168.100.1` is the router. `proto dhcp` confirms the address came from DHCP, and `src 192.168.100.22` is the server's current address. |

Copy the MAC. The gateway is the router's address, `192.168.100.1`.

The flags in angle brackets are worth a glance: `UP` means the interface is enabled, `LOWER_UP` means a cable is plugged in and the link is live. If you ever see `UP` without `LOWER_UP`, the cable is the problem.

### 2. Reach the router admin page

This is where it got interesting. `http://192.168.100.1` from my laptop **timed out**, even though the laptop could ping the router. From the server, `curl -I http://192.168.100.1` returned `200 OK`. The difference: the server is on a **wired** port, the laptop on **Wi-Fi**, and this firmware only serves the admin page to wired clients.

Rather than find a cable, I tunnelled through the server with SSH. The first attempt forwarded only port 80, and the router's login page immediately bounced the browser to `https://localhost`, dropping the port. So the tunnel has to carry **both** 80 and 443, and 443 has to land on the laptop's real port 443 so the redirect works:

```bash
# on the laptop; sudo because port 443 is privileged on macOS
sudo ssh -L 443:192.168.100.1:443 -L 8080:192.168.100.1:80 bernado@192.168.100.22
```

Reading it piece by piece:

| Piece | Meaning |
|---|---|
| `sudo` | Needed to open local port 443 (below 1024) |
| `-L 443:192.168.100.1:443` | Laptop port 443 → router port 443, via the server. This is the one the redirect needs. |
| `-L 8080:192.168.100.1:80` | Laptop port 8080 → router port 80. Plain HTTP, for the initial page. |
| `bernado@192.168.100.22` | The SSH session that carries both forwards |

The command gives you an ordinary shell on the server, with the tunnels alive in the background. Keep that session open and browse to **`https://localhost`**. Accept the certificate warning (the router's cert is self-signed) and the Huawei login page appears.

If `https://localhost` shows nothing, check the terminal: a `bind: Address already in use` message on 443 means something else on the Mac already owns that port.

If your laptop is wired, or your router doesn't have this restriction, just open `http://192.168.100.1` and skip the tunnel.

### 3. Log in

The HG8145V5 asks for three things, all printed on the label on the back of the router:

| Field | Where on the label |
|---|---|
| User Name | `username:` line |
| Password | `password:` line |
| SN | `SN:` line, a 16-character code starting `4857` |

The SN must match exactly. Mine was hard to read from a photo; `z` vs `2` cost me one attempt.

The serial-number field is unusual. Most routers ask for a username and password only. Treat it as a third password: it proves you have physical access to the box.

> ⚠️ That label also shows the Wi-Fi key. Don't commit a photo of it to the repo.

### 4. Bind the MAC to the IP

**Advanced → LAN → DHCP Static IP Configuration → New.** Enter the MAC from step 1 and `192.168.100.22`, then **Apply**.

I reserved the address the server **already had**, so nothing changed immediately and there was no chance of a conflict with another device.

Two things to check on that screen before you click Apply:

| Check | Why |
|---|---|
| The MAC matches `eno1` exactly, every pair | A single wrong digit means a reservation for a device that doesn't exist, and the server keeps getting pool addresses as before |
| The IP is inside the router's LAN subnet (`192.168.100.x`) | Some firmware will accept an address outside the subnet and the device will never be able to use it |

### 5. Renew the lease and verify

On the server, renew without rebooting:

```bash
sudo networkctl renew eno1
hostname -I
```

| Command | What it does |
|---|---|
| `sudo networkctl renew eno1` | Tells systemd-networkd to run the DHCP conversation again on `eno1` right now, instead of waiting for the lease to reach its renewal point. The router answers from its reservation table. |
| `hostname -I` | Prints every IP address the machine has, space-separated. Capital `I`. Lowercase `i` gives `127.0.1.1`, the server-local hosts entry, which is useless here. |

Then reboot once and check again:

```bash
sudo reboot
# wait, then from the laptop:
ssh bernado@192.168.100.22 hostname -I
```

The reboot is the real test. A renew reuses state the server already has; a reboot starts DORA from a cold DISCOVER, which is what will happen after a power cut. Running `hostname -I` through `ssh` like this, with the command after the address, runs it on the server and prints the output on the laptop without opening an interactive shell.

```text
192.168.100.22 10.42.0.0 10.42.0.1
```

The first address is the one that matters. The two `10.42.x.x` addresses are the K3s pod network interfaces (`flannel.1` and `cni0`) that appeared in [step 12](../04-kubernetes/12-installing-k3s.md); they live only inside the server.

If the first address is **not** `192.168.100.22` after the reboot, the reservation did not take. Go back to the router page and compare the MAC digit by digit.

### 6. Give the server a name on the laptop

With the IP pinned, a hosts entry on the Mac is safe:

```bash
sudo sh -c 'echo "192.168.100.22  quantum" >> /etc/hosts'
```

Why the odd shape of that command: `sudo echo … >> /etc/hosts` does **not** work, because the shell opens the file for the redirect before `sudo` runs, as your normal user, and gets permission denied. Wrapping the whole thing in `sh -c '…'` makes the redirect happen inside the root shell. `>>` appends; a single `>` would wipe the file, so be careful with the number of arrows.

From the laptop, `ssh bernado@quantum` and `http://quantum` now work. This is laptop-only; other devices on the network still need the IP.

To confirm the entry is being used, `ping -c 1 quantum` on the Mac should print `PING quantum (192.168.100.22)`. If it still fails, check for a typo in the file with `grep quantum /etc/hosts`.

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

## Glossary

| Term | Meaning |
|---|---|
| MAC address | The factory-set hardware identifier of a network interface, six hex pairs. What DHCP reservations are keyed on. |
| IP address | The address a device uses on a network. On a home LAN it comes from DHCP. |
| Private IP ranges | `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`. Never routed on the internet, safe to share. |
| `/24` | A subnet where the first 24 bits are fixed: `192.168.100.1` to `.254` |
| DHCP | The protocol that hands out IP leases, plus the gateway, subnet mask and DNS server |
| DORA | Discover, Offer, Request, Acknowledge: the four DHCP messages |
| Lease | An address granted for a fixed time; renewed halfway through, reclaimable if the device is away |
| Reservation / static binding | A router rule: this MAC always gets this IP. The device still uses DHCP. |
| Static IP | An address configured on the device itself, DHCP switched off. The alternative I did not choose. |
| Gateway | The router's address, where traffic for other networks is sent. `192.168.100.1` here. |
| Hostname | The name a machine gives itself (`quantum`). Known only to that machine unless DNS or a hosts file says otherwise. |
| `/etc/hosts` | A local file checked before DNS. One line per name, per machine. |
| `ssh -L` | Local port forward: open a port here, deliver connections to an address as seen from the SSH server |
| Privileged port | Ports below 1024; opening one needs root, hence `sudo` for 443 |
| Self-signed certificate | A certificate not issued by a trusted CA. Browsers warn; fine for your own router on your own LAN. |
| ICMP / ping | A network-layer echo test. Proves the path, says nothing about services. |
| `eno1` | The server's wired Ethernet interface |
| `flannel.1`, `cni0` | K3s pod-network interfaces on the server; their `10.42.x.x` addresses are internal only |

---

[◀ Previous: My First App](../04-kubernetes/13-first-app.md) · [🏠 Home](../../README.md) · [Next: Running kubectl From the Laptop ▶](../06-working-with-the-cluster/15-kubectl-from-laptop.md)
