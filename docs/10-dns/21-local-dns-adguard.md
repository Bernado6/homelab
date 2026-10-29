# 21. Local DNS for the Whole House (AdGuard Home)

This is a long page on purpose. DNS is the layer everything else sits on, this step touches it in three places at once (the router, the server's own resolver, and the cluster), and the one thing that went wrong took a day to understand. If you only read one background section in this guide, make it this one.

## What

Run a DNS server on the cluster, and tell the router to hand that server to every device at home. After this:

- `*.home.arpa` resolves on **every** device, phones included. The `/etc/hosts` lines on the Mac go away.
- Any new Ingress host works the moment it exists. One wildcard rule covers all of them.
- Ads and trackers are blocked for the whole network as a side effect.
- You can see every DNS query every device makes, which teaches you more about your network than anything else on this list.

## Background: how DNS actually works

### Names to addresses

Computers talk to IP addresses. People type names. DNS, the Domain Name System, is the lookup that turns one into the other. When you type `portfolio.home.arpa`, before a single byte of HTTP is sent, something has to answer the question *"what address is that?"*

Every device has a tiny piece of code for this called the **stub resolver**. It does not know any answers. It knows the address of **one DNS server** to ask, and it asks it. That one server address usually arrives with the IP lease, from **DHCP**, which is why the router is involved in this step at all: the router's DHCP server is what tells every device "your DNS server is X".

### Who actually knows the answer

The server a device asks is almost never the one that knows. For a public name like `github.com`, the chain is:

```text
phone ──► router ──► ISP or public resolver ──► root servers ──► .com servers ──► github's servers
                     (recursive resolver: does the walking, caches the result)
```

A **recursive resolver** walks that hierarchy on your behalf and remembers the answer for as long as the record's **TTL** (time to live) allows. Home routers don't usually recurse; they **forward** to whichever resolver the ISP gave them. Public resolvers you may have heard of, `1.1.1.1` (Cloudflare), `9.9.9.9` (Quad9), `8.8.8.8` (Google), are recursive resolvers anyone can use.

An **authoritative** server is the one at the end of the chain that owns the zone and has the real records.

### The record types you'll meet

| Type | Means | Example |
|---|---|---|
| `A` | name → IPv4 address | `portfolio.home.arpa → 192.168.100.22` |
| `AAAA` | name → IPv6 address | not used in this homelab yet |
| `CNAME` | name → another name | `www.example.com → example.com` |
| `PTR` | address → name (reverse lookup) | what shows the client *name* in AdGuard's dashboard |

### What's different about `*.home.arpa`

Nobody on the internet is authoritative for `home.arpa`. [RFC 8375](https://www.rfc-editor.org/rfc/rfc8375) reserves it for exactly this: names that exist only inside a home network. Public resolvers answer `NXDOMAIN` (no such name) for it, by design. So to make `portfolio.home.arpa` resolve, **something inside the house must answer before the question ever leaves**. That something is what this step builds.

Until now that "something" was `/etc/hosts` on the Mac: a file the stub resolver checks before asking any server. It works for one machine. It does nothing for a phone.

### DNS rewrites, and the wildcard

A **rewrite** is a rule in a forwarding DNS server that says "if anyone asks for this name, answer with this address and don't ask upstream". AdGuard Home supports wildcards in rewrites, so one rule:

```text
*.home.arpa  →  192.168.100.22
```

answers `portfolio.home.arpa`, `grafana.home.arpa`, `adguard.home.arpa`, and any name you invent next year. All of them point at the server, and Traefik decides which app by the `Host` header, exactly as it has since [step 16](../06-working-with-the-cluster/16-nginx-ingress.md). DNS gets the request to the right machine; the Ingress gets it to the right pod.

### Two DNS worlds in this homelab

There is DNS **inside** the cluster and DNS **outside** it, and they are different systems.

| World | Server | Answers names like | Gets its upstream from |
|---|---|---|---|
| Inside the cluster | **CoreDNS** (installed by K3s in step 12) | `portfolio.portfolio.svc.cluster.local` | the **node's** `/etc/resolv.conf` |
| Outside, the house | **AdGuard Home** (this step) | `portfolio.home.arpa`, `github.com` | upstreams you configure |

Pods ask CoreDNS. CoreDNS answers `*.cluster.local` itself and **forwards everything else to whatever the node uses**. That last fact is why the server's own resolver configuration matters so much later in this page.

### Transport: plain DNS, DoH, DoT

Classic DNS is unencrypted UDP on port 53 (TCP 53 for large answers). Anyone between you and the resolver, your ISP included, can read every name you look up. **DNS over HTTPS (DoH)** and **DNS over TLS (DoT)** wrap the same questions in TLS. Devices at home still speak plain port 53 to AdGuard, which is fine on your own LAN; AdGuard then speaks **DoH** to its upstreams, so the queries leaving the house are encrypted.

One wrinkle: an upstream like `https://dns.cloudflare.com/dns-query` is itself a *name*, and AdGuard needs an address to reach it before it can resolve anything. **Bootstrap DNS servers** are plain IPs used only for that first lookup.

## Background: what AdGuard Home is

AdGuard Home is a **forwarding DNS server with policy**. It is not a recursive resolver; it doesn't walk the hierarchy itself. For each query it:

1. Checks its **rewrites**. If one matches, it answers immediately. This is how `*.home.arpa` works.
2. Checks its **blocklists**. If the name is on one, it answers `0.0.0.0`, so the device connects to nothing. This is how ad-blocking works without any software on the device.
3. Checks its **cache**. Recently answered names come straight back.
4. Otherwise **forwards** to an upstream resolver, over DoH if you configured one, caches the answer, and returns it.
5. **Logs** the query, with the client address, so the dashboard can show who asked what.

It is a single Go binary with a web UI. Inside the container:

| Piece | Where | What it is |
|---|---|---|
| The binary | `/opt/adguardhome/AdGuardHome` | DNS listener on 53, web UI on 80 (on 3000 before first setup) |
| Configuration | `/opt/adguardhome/conf/AdGuardHome.yaml` | Everything you set in the UI: upstreams, rewrites, users, listen ports. One file. |
| Working data | `/opt/adguardhome/work/data/` | Query log, statistics database, downloaded blocklists |

Both directories need to **persist** across pod restarts, which is why the manifest gives it a volume. Lose `conf/` and you're back at the setup wizard.

A note on privileges: the container runs as root so it can bind ports 53 and 80, which are privileged. That's the upstream image's default. It lives in its own `dns` namespace without the `restricted` Pod Security level the portfolio uses. Tightening it to non-root with `NET_BIND_SERVICE` is a possible later improvement, not a requirement.

### Why AdGuard Home and not the alternatives?

| Option | What it is | Verdict |
|---|---|---|
| **AdGuard Home** ✅ | Single Go binary, web UI, blocklists, DoH upstreams built in | **Wildcard rewrites are a first-class UI feature.** One container, one config file, one volume. Fits a pod cleanly. |
| Pi-hole | The classic home ad-blocker | Same job, more moving parts: dnsmasq, FTL, lighttpd and PHP inside one image. Wildcard local records need a dnsmasq config snippet instead of a UI field. Works, just more friction in a pod. |
| Blocky | Lightweight Go DNS proxy, config-file only | Excellent and even simpler, but no UI. For a first time, seeing the query log and the rewrite rule on screen matters. |
| dnsmasq or Unbound alone | Minimal resolvers | No UI, no blocking, hand-written config. Unbound is a good *upstream* later if you want to recurse yourself instead of trusting Cloudflare or Quad9. |
| The cluster's CoreDNS | Already running | It is the cluster's internal DNS. Mixing "resolve pods" with "resolve the house" couples two things that should fail independently. |
| The Huawei router | Already the DHCP server | Can't define local records at all. It can only point devices at another DNS server, which is exactly what we'll make it do. |

## Why

| Problem before this step | After |
|---|---|
| Each new site needs a new line in `/etc/hosts`, per device, with `sudo` | One wildcard rule, once |
| Phones and tablets can't reach anything on the homelab | Everything resolves everywhere |
| The router can't add local names | The router just points at the server |
| No idea what devices are talking to | A query log for the whole house |

And the learning reasons: this is the first `type: LoadBalancer` Service that isn't Traefik, the first UDP workload, the first time the cluster provides a service the *rest of the house* depends on, and the first time an infrastructure dependency loop bites for real.

## How the pieces fit in K3s

Read the manifest, [`apps/adguard/adguard.yaml`](../../apps/adguard/adguard.yaml), alongside this.

### The query path

```text
phone                 router (DHCP said: DNS = 192.168.100.22)
  │  UDP 53
  ▼
192.168.100.22:53  ── node IP. K3s ServiceLB owns this port ──┐
                                                              │ hostPort → DNAT
                                                              ▼
                                            adguard pod (10.42.0.x):53
                                              ├─ *.home.arpa?      → answer 192.168.100.22
                                              ├─ on a blocklist?   → answer 0.0.0.0
                                              ├─ cached?           → answer from cache
                                              └─ else DoH ─────────► dns.cloudflare.com / dns.quad9.net
```

### Each object and why it's there

| Object | What it does | Why this way |
|---|---|---|
| `Namespace dns` | Its own namespace | Isolation, and `kubectl -n dns` as a habit |
| `PersistentVolumeClaim adguard-data`, 1Gi, `local-path` | A folder on the server's SSD, mounted into the pod | `conf/` and `work/` survive restarts. Same storage class as Prometheus in step 18. |
| `Deployment`, `replicas: 1`, `strategy: Recreate` | One pod; on updates, stop the old one before starting the new one | A `ReadWriteOnce` volume can't be mounted by two pods at once. `RollingUpdate` would deadlock waiting for the old pod to release it. |
| Two `volumeMounts` with `subPath` | `work` and `conf` are two directories inside **one** volume | One PVC instead of two |
| `startupProbe` on TCP 53, 60 × 30s | Holds the other probes off for up to 30 minutes | AdGuard doesn't open 53 until the first-run wizard is done. See Challenges. |
| `readinessProbe` on TCP 53 | Pod only receives traffic once 53 answers | The DNS Service has no endpoints until then, so the router never gets a half-ready server |
| `livenessProbe` on TCP 53 | Restart the container if DNS stops answering | The whole house depends on this port |
| `Service adguard-dns`, `type: LoadBalancer`, UDP + TCP 53 | Publishes port 53 **on the node's own IP** | This is the address the router hands out. Details below. |
| `externalTrafficPolicy: Local` on that Service | Keep the real client address | Without it every device shows as `10.42.0.1`. Same issue as the Traefik log in step 16. Safe with one node. |
| `Service adguard-web`, `ClusterIP` 80 | Cluster-internal address for the UI | Traefik's backend |
| `Ingress adguard`, host `adguard.home.arpa`, TLS | The UI at `https://adguard.home.arpa` | Same cert-manager annotations as step 20 |

### ServiceLB, the piece that makes port 53 appear on the node

A `type: LoadBalancer` Service normally needs a cloud provider to hand out an IP. K3s ships a small substitute called **ServiceLB** (also klipper-lb). You met it in step 12 as the `svclb-traefik` pod without knowing it. When it sees a LoadBalancer Service it:

1. Creates a `svclb-<service>` pod in `kube-system` on each node.
2. Gives that pod a **hostPort** for each Service port, which means "this port on the node's IP belongs to this pod".
3. Reports the node's IP as the Service's `EXTERNAL-IP`.

That's why `kubectl -n dns get svc` shows `192.168.100.22` under EXTERNAL-IP. It's the same mechanism that gives Traefik ports 80 and 443.

Under the hood, hostPort is implemented by the CNI's **portmap** plugin as an iptables **DNAT** rule, which rewrites the destination of matching packets to the pod's address. The rule matches packets arriving on port 53 whose destination is **any address local to the node**. Hold that thought.

## The rule: a DNS server must not depend on itself

Three places this applies, and the first one is the one that got me.

### 1. The node's own resolver

Ubuntu runs `systemd-resolved`. It provides a **stub resolver on `127.0.0.53:53`**, and `/etc/resolv.conf` points there, so every program on the node, including **containerd when it pulls images**, sends its DNS queries to `127.0.0.53`. The stub then forwards to whatever DHCP configured, which was the router.

Now re-read the ServiceLB paragraph. The moment the `adguard-dns` Service exists, there's a DNAT rule: *port 53 on any local address → the AdGuard pod*. `127.0.0.53` is a local address. From that moment, the node's own resolver is redirected into AdGuard. If AdGuard is running, that mostly works, badly. If AdGuard is **not** running, every lookup on the node is refused, including the lookup of `registry-1.docker.io` that containerd needs to download the AdGuard image. The image that would fix it can never arrive.

The symptoms, on the server:

```text
$ dig +short registry-1.docker.io
;; communications error to 127.0.0.53#53: connection refused
$ dig @1.1.1.1 +short registry-1.docker.io
18.211.107.208 ...               # upstream fine, only the stub is dead
$ resolvectl status eno1 | grep 'DNS Servers'
       DNS Servers: 192.168.100.1   # the router was never changed; this is not the router
```

Even once AdGuard runs, leaving it like this means the node's DNS breaks every time the AdGuard pod restarts. The fix is to **take the node off the stub entirely** and give it fixed upstreams, which is section 1 of the How below. After that, the node asks `1.1.1.1` directly. The DNAT rule only matches *local* destinations, so those packets sail past it.

### 2. CoreDNS

CoreDNS forwards external names to the node's `/etc/resolv.conf`. Fixing the node fixes the cluster, but CoreDNS reads that file when its pod starts, so it needs a restart to notice.

### 3. AdGuard's own upstreams

If AdGuard were told to use "the system resolver" as upstream, it would ask the node, which after section 7 would be told by the router to ask AdGuard. Explicit upstreams (`dns.cloudflare.com`, `dns.quad9.net`) break that loop before it exists.

## When

After HTTPS ([step 20](../09-tls/20-https-with-cert-manager.md)), so the admin UI is behind TLS from day one. Before any step that adds more `*.home.arpa` names, which is every step after this.

## How

The sections are numbered in the order they **should** be done. I did section 1 last, and paid for it.

### 1. First: take the server off its stub resolver

On the server, over SSH:

```bash
# 1a. Tell systemd-resolved to stop listening on 127.0.0.53 and to use fixed upstreams
sudo mkdir -p /etc/systemd/resolved.conf.d
printf '[Resolve]\nDNSStubListener=no\nDNS=1.1.1.1 9.9.9.9\n' | sudo tee /etc/systemd/resolved.conf.d/no-stub.conf

# 1b. Stop DHCP from injecting the router (and soon AdGuard) as a per-link DNS server
F=/etc/netplan/60-dns.yaml
printf 'network:\n  version: 2\n  ethernets:\n    eno1:\n      dhcp4-overrides:\n        use-dns: false\n      dhcp6-overrides:\n        use-dns: false\n      nameservers:\n        addresses:\n        - 1.1.1.1\n        - 9.9.9.9\n' | sudo tee $F
sudo chmod 600 $F

# 1c. Point /etc/resolv.conf at the real upstream list instead of the stub
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
sudo netplan apply
sudo systemctl restart systemd-resolved

grep nameserver /etc/resolv.conf     # exactly: 1.1.1.1 and 9.9.9.9
dig +short registry-1.docker.io      # addresses, not "connection refused"
```

What each part does:

| Line | Effect |
|---|---|
| `DNSStubListener=no` | `127.0.0.53` stops existing. Nothing can be hijacked through it. |
| `DNS=1.1.1.1 9.9.9.9` | resolved's global upstreams |
| `dhcp4-overrides: use-dns: false` (and dhcp6) | Ignore the DNS servers the router advertises. Netplan insists both IPv4 and IPv6 be set the same way. |
| `nameservers: addresses:` | Per-link upstreams for `eno1`, the same two |
| symlink to `/run/systemd/resolve/resolv.conf` | That file lists the real upstreams. `/etc/resolv.conf` used to point at a different file that listed only the stub. |

Then from the laptop, make CoreDNS re-read the node's resolver config:

```bash
kubectl -n kube-system rollout restart deploy/coredns
```

Two paste traps: a heredoc's closing `EOF` must start at column one (an indented paste hangs the shell, hence `printf … | tee` above), and YAML needs a space after `addresses:`. If `netplan apply` complains, `sudo vim $F` and fix it by hand.

### 2. Deploy

From the homelab repo on the laptop:

```bash
kubectl apply -f apps/adguard/adguard.yaml
kubectl -n dns get pods,svc,ingress,pvc
```

Expect the pod at `0/1 Running` with zero restarts (the startup probe is waiting for the wizard), the PVC `Bound`, and:

```text
NAME          TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                     AGE
adguard-dns   LoadBalancer   10.43.x.x      192.168.100.22   53:3xxxx/UDP,53:3xxxx/TCP   30s
```

A new `svclb-adguard-dns-…` pod has appeared in `kube-system`. That's ServiceLB doing its job.

### 3. First-run wizard, through a port-forward

AdGuard serves a setup wizard on port 3000 until it has a config. This is a legitimate use of `port-forward`: a one-off, from the laptop, to a UI that has no Ingress yet.

```bash
kubectl -n dns port-forward deploy/adguard 3000:3000
```

Open `http://localhost:3000`. Five screens:

| Screen | Set | Why |
|---|---|---|
| 2, Admin Web Interface | **All interfaces**, port **80** | The `adguard-web` Service and the Ingress target port 80 inside the pod |
| 2, DNS server | **All interfaces**, port **53** | The `adguard-dns` Service targets 53. If it warns the port is in use, that's the pod seeing its own hostPort; continue. |
| 3, Authentication | A username and a strong password | This UI controls DNS for every device in the house |
| 4, Configure your devices | Skip | The router does this in section 7 |
| 5 | Open Dashboard | |

Clicking through to the dashboard **fails** with "localhost refused to connect". That is success: AdGuard has moved to port 80 and the forward on 3000 is dead. Confirm:

```bash
kubectl -n dns get pods      # now 1/1 Running: port 53 is open, the startup probe passed
```

### 4. Log in and add the wildcard rewrite

Nothing resolves `adguard.home.arpa` yet, so one more forward, to the real UI:

```bash
kubectl -n dns port-forward svc/adguard-web 8080:80
```

At `http://localhost:8080`, log in. **Filters → DNS rewrites → Add DNS rewrite:**

| Domain | Answer |
|---|---|
| `*.home.arpa` | `192.168.100.22` |

That rule is the whole point of the step.

### 5. Upstreams and blocking

**Settings → DNS settings:**

| Field | Value | Why |
|---|---|---|
| Upstream DNS servers | `https://dns.cloudflare.com/dns-query` and `https://dns.quad9.net/dns-query`, one per line | DoH: queries leaving the house are encrypted. Two providers so one outage doesn't take you down. |
| Bootstrap DNS servers | `1.1.1.1` and `9.9.9.9` | Plain IPs to resolve the two names above, once |
| Rate limit | `0` | It's your own network |

Click **Test upstreams**, wait for green, then **Apply**.

**Filters → DNS blocklists:** the default *AdGuard DNS filter* is already on. Enough to start.

### 6. Test the server directly, before touching the router

From the Mac. Nothing on the network changes yet; `@192.168.100.22` asks AdGuard explicitly:

```bash
dig @192.168.100.22 portfolio.home.arpa +short     # 192.168.100.22   (the rewrite)
dig @192.168.100.22 anything.home.arpa +short      # 192.168.100.22   (wildcard: a name that exists nowhere)
dig @192.168.100.22 github.com +short              # a public IP      (forwarding works)
dig @192.168.100.22 ad.doubleclick.net +short      # 0.0.0.0          (blocklist works)
```

My run returned exactly that, and the AdGuard dashboard then showed 4 queries, 1 blocked, 15 ms average. Four correct answers prove the rewrite, the wildcard, DoH upstreams, blocking, **and** the ServiceLB path on port 53, all at once.

Check **Top clients** on the dashboard. It should show your Mac's LAN address. If it shows `10.42.0.1` instead, `externalTrafficPolicy: Local` isn't on the Service yet; re-apply the manifest.

### 7. Point the router at AdGuard

In the Huawei admin page (through the SSH tunnel from [step 14](../05-networking/14-dhcp-reservation.md) if the laptop is on Wi-Fi), **Advanced → LAN → DHCP Server Configuration**, in the *Primary Address Pool* block:

| Field | Value |
|---|---|
| Primary DNS server | `192.168.100.22` |
| Secondary DNS server | `9.9.9.9` |

The HG8145V5 **rejects `1.1.1.1`** here with *Incorrect format of the IP address of the secondary DNS server*. It's a firmware validation quirk. `9.9.9.9` is accepted; blank is also valid.

**About the secondary.** Devices treat the two as equals, not as primary and backup. Some will send part of their queries to `9.9.9.9`, which answers `NXDOMAIN` for `*.home.arpa`, so a local name may occasionally fail on one device until it retries. Leaving it blank means every device depends on AdGuard being up, though the cluster itself no longer does, thanks to section 1. Pick the failure mode you prefer. I kept the secondary.

Devices only see the new DNS when their DHCP lease renews.

### 8. Clean up the laptop and verify for real

```bash
sudo sed -i '' '/home\.arpa/d;/quantum$/d' /etc/hosts     # remove the hosts-file workaround
sudo ipconfig set en0 DHCP                                 # renew the lease now, not tomorrow
scutil --dns | grep nameserver | head -3
dig portfolio.home.arpa +short                             # no @server: the Mac's normal resolver
```

My result:

```text
  nameserver[0] : 192.168.100.22
  nameserver[1] : 9.9.9.9
192.168.100.22
```

Now `https://adguard.home.arpa`, `https://portfolio.home.arpa` and the rest open in the browser with the padlock from step 20, through DNS, with no hosts file and no port-forward. Close any forwards still running.

### 9. Phones

They resolve `*.home.arpa` now. HTTPS sites will warn until the phone trusts the homelab CA from step 20. For a phone you use often, send it `~/homelab-root-ca.crt`:

- **iPhone:** AirDrop the file, install the profile under Settings → General → VPN & Device Management, then **Settings → General → About → Certificate Trust Settings** and switch it on. The second step is the one people miss.
- **Android:** Settings → Security → Encryption & credentials → Install a certificate → CA certificate.

Guests get the warning. Nothing on `*.home.arpa` is meant for guests.

### 10. Reading the Query Log

**Query Log** in the AdGuard menu is the most educational screen in the homelab. Spend ten minutes with it:

- Filter by your phone's address and watch what it looks up while sitting idle. That's telemetry.
- Click any entry: it shows the **upstream** that answered, the **response time**, and the **status** (processed, blocked, rewritten).
- `*.home.arpa` entries show status *Rewritten*. Blocked ones show which filter list matched.
- Open **Settings → Client settings** and give your devices names. The dashboard becomes readable.

## Challenges and solutions

| Challenge | What was going on | Solution |
|---|---|---|
| Pod stuck in `ImagePullBackOff` for 22 hours with `lookup registry-1.docker.io: Try again` | Creating the `type: LoadBalancer` Service on port 53 made ServiceLB's hostPort rule redirect **every** local port-53 connection, including the node's own stub resolver at `127.0.0.53`, into the AdGuard pod. The pod had no image yet, so the node could not resolve the registry to fetch it. `dig @1.1.1.1` worked the whole time; only the stub was dead. I first assumed the router had been changed; `resolvectl` showed it hadn't. | Took the node off the stub resolver entirely ([section 1](#1-first-take-the-server-off-its-stub-resolver)). The image pulled within a minute. That section is now first for a reason. |
| Image pulled, then `Running` with restarts climbing: `Readiness probe failed … :53: connection refused`, `Container adguard failed liveness probe, will be restarted` | My probes checked port 53, but AdGuard doesn't listen on 53 until the first-run wizard has finished. Liveness killed the container every two minutes, which would also have made the wizard a race. | Added a **startupProbe** on port 53 with a 30-minute window (`periodSeconds: 30`, `failureThreshold: 60`). Liveness and readiness are suspended until it passes. On later restarts the saved config opens 53 at once. Textbook startup-probe use, and a CKAD topic. |
| AdGuard's dashboard showed every client as `10.42.0.1` | The node's internal bridge address. ServiceLB rewrote the source of every incoming query, so per-client stats were meaningless. | `externalTrafficPolicy: Local` on the `adguard-dns` Service. K3s then preserves the real client address. Same root cause as the Traefik log entry in step 16. |
| Router rejected `1.1.1.1` as secondary DNS: *Incorrect format of the IP address* | HG8145V5 firmware validation quirk. | `9.9.9.9` was accepted. Blank is also fine. |
| After the router change, the Mac still showed `192.168.100.1` as its nameserver | DHCP settings only reach a device when its lease renews. | `sudo ipconfig set en0 DHCP`, or toggle Wi-Fi. `scutil --dns` then listed `192.168.100.22` first. |
| Heredoc paste hung the shell with a `>` prompt | The closing `EOF` was indented. Terminators must start at column one. | `Ctrl+C`, then `printf … \| sudo tee` instead, which has no terminator to get wrong. |
| `netplan apply` rejected the file twice | First: `use-dns` must be set for both `dhcp4-overrides` and `dhcp6-overrides`. Second: a missing space after `addresses:` kept surviving the paste. | Added the IPv6 override; switched to the `- item` list form. Ended up fixing the last one in `vim`. |

## Lessons

> 💡 **What I learned:**
> - **DNS is "ask one server"; everything else is who that server asks.** Devices know one address, from DHCP. Change what DHCP hands out and you've changed DNS for the house.
> - **`*.home.arpa` can only be answered from inside the house.** No public resolver will ever know it. A wildcard rewrite in a local forwarder replaces every hosts file at once, and Traefik does the rest by `Host` header.
> - **A DNS server must not depend on itself.** Three places: the node's stub resolver, CoreDNS, and AdGuard's own upstreams. The first one hijacked the node through a port-53 DNAT rule and stopped the image that would have fixed it from being pulled. Do section 1 before anything else.
> - **`type: LoadBalancer` on K3s means "publish on the node IP".** ServiceLB is how Traefik had 80 and 443 all along; now AdGuard has 53 the same way. It works by hostPort, which is a DNAT rule on **any local address**, which is the whole story above.
> - **Startup probes exist for exactly this.** A container that isn't ready to be judged yet gets a window; the real probes take over after. Don't bend liveness delays to fake it.
> - **`port-forward` has a real job.** Reaching a setup wizard that has no Ingress yet is exactly what it's for. Then it goes away.
> - **Secondary DNS is not a backup.** Clients spread queries across both. Decide which failure you'd rather have.
> - **Encrypt the queries that leave the house.** Plain port 53 inside the LAN, DoH to upstreams. Bootstrap IPs are the one unencrypted lookup, and they only resolve the upstream's own name.

## Glossary

| Term | Meaning |
|---|---|
| Stub resolver | The tiny client on every device that asks one configured DNS server |
| Recursive resolver | A server that walks the DNS hierarchy for you and caches answers (`1.1.1.1`, `9.9.9.9`) |
| Forwarder | A DNS server that passes queries to another server instead of recursing; routers and AdGuard are forwarders |
| Authoritative server | The server that owns a zone and holds the real records |
| TTL | How long a cached answer may be reused, in seconds |
| `A` / `AAAA` / `CNAME` / `PTR` | IPv4 / IPv6 / alias / reverse records |
| `NXDOMAIN` | "No such name" |
| Rewrite | A local rule: answer this name with this address, don't ask upstream |
| DoH / DoT | DNS over HTTPS / over TLS: encrypted transport to the upstream |
| Bootstrap DNS | Plain IP resolvers used only to find the DoH upstream's own address |
| DHCP | Hands out IP leases **and** the DNS server address to every device |
| `systemd-resolved` | Ubuntu's resolver service; its stub listens on `127.0.0.53` |
| CoreDNS | The cluster-internal DNS for `*.cluster.local`; forwards the rest to the node's resolver |
| ServiceLB / klipper-lb | K3s's built-in way to give a `LoadBalancer` Service the node's IP, via hostPort |
| hostPort / DNAT | "This node port belongs to this pod", implemented as an iptables destination rewrite |
| `externalTrafficPolicy: Local` | Deliver only to pods on this node and keep the client's source address |
| startupProbe | A probe that must pass once before liveness and readiness begin |

---

[◀ Previous: HTTPS Everywhere With cert-manager](../09-tls/20-https-with-cert-manager.md) · [🏠 Home](../../README.md) · [Next: Buying a Domain and Setting Up Cloudflare ▶](../11-public-access/22-domain-and-cloudflare.md)
