# 11. First Login and SSH

Three commands, and then the monitor goes back to its owner. This is the step where the OptiPlex stops being a computer you sit at and becomes a server you reach over the network. It is short to do and worth understanding properly, because every later page begins with "SSH in".

## What

Log in at the server's own keyboard one last time, bring the freshly installed system up to date, find the IP address the router gave it, and open an **SSH** session to it from the laptop. From then on the server runs **headless**.

## Background: what `apt update` and `apt upgrade` do

Ubuntu installs software from **repositories**: online archives of packages signed by Canonical, served from the mirror the installer picked in [step 10](10-installing-ubuntu-server.md). `apt` is the tool that talks to them, and the two commands do two different things:

| Command | What it does | Changes installed software? |
|---|---|---|
| `apt update` | Downloads the **list** of available packages and versions from each repository | No. It only refreshes the catalogue |
| `apt upgrade` | Compares the catalogue with what is installed and **installs newer versions** of everything that has one | Yes |

They are chained with `&&`, which means "run the second only if the first succeeded", so a failed refresh does not lead to upgrading from stale information. `-y` answers "yes" to the "Do you want to continue?" prompt in advance. `sudo` runs both as the administrator, because installing software changes system files.

The installer already pulled security updates during the install, so this first upgrade is usually small. The habit matters more than the size: a server that gets this pair of commands regularly is a server that has its security fixes.

## Background: what an IP address is to the server, and the loopback trap

A machine on a network can have several addresses at once, one per **network interface**. After a fresh install the OptiPlex has two:

| Interface | Address | What it is |
|---|---|---|
| `lo` (loopback) | `127.0.0.1` | The machine talking to itself. Every computer has it; it never leaves the box |
| `eno1` (Ethernet) | `192.168.100.x` | The lease the router gave it. The one the laptop can reach |

Later, after [step 12](../04-kubernetes/12-installing-k3s.md), K3s adds more interfaces (`cni0`, `flannel.1`) with `10.42.x.x` addresses for the pod network. They show up in `hostname -I` too, and they are only reachable from inside the cluster.

### Why `hostname -i` lies

`hostname` has two similar-looking flags that work in completely different ways:

| Flag | How it finds an answer | On this server |
|---|---|---|
| `hostname -i` (lowercase) | Looks up **the hostname in `/etc/hosts`** and prints what it finds | `127.0.1.1` |
| `hostname -I` (capital) | Asks the kernel for **the address of every non-loopback interface** | `192.168.100.x` (and later the `10.42.x.x` ones) |

The lowercase version returns `127.0.1.1` because Debian and Ubuntu put a line in `/etc/hosts` mapping the machine's own hostname to that address. It is a second loopback address, chosen so that programs that resolve the local hostname get *something* even when the network is down. It is correct for the machine's own purposes and useless for anyone else: `127.0.1.1` from the laptop means the laptop itself.

```text
 hostname -i                              hostname -I
 ───────────                              ───────────
 /etc/hosts                               kernel interfaces
   127.0.0.1   localhost                    lo     127.0.0.1   (skipped: loopback)
   127.0.1.1   quantum   ◄── answer          eno1   192.168.100.x  ◄── answer
```

`ip -4 addr` is the long form of the same question: it lists every interface with its IPv4 address and leaves you to read the `inet` line under the Ethernet one. It is the command to reach for when `hostname -I` prints several addresses and you want to know which belongs to which interface.

## Background: what SSH is

**SSH** (Secure Shell) is a protocol for opening an encrypted terminal session on another machine over the network. The server runs `sshd`, the **OpenSSH server** installed by the tick-box in step 10, listening on TCP port **22**. The laptop runs the `ssh` client, which macOS and Linux include and Windows has had since Windows 10.

```text
 laptop                                           server 192.168.100.x
 ──────                                           ───────────────────
 ssh user@192.168.100.x
   │  TCP 22
   ├─► server presents its HOST KEY  ─────────────  sshd
   │   "is this the machine I think it is?"          │
   │   first time: fingerprint prompt, saved to       │
   │   ~/.ssh/known_hosts                             │
   │                                                  │
   ├─► encrypted channel established                 │
   │                                                  │
   ├─► authenticate: password (or key)  ───────────►  checks against the user account
   │                                                  │
   ▼                                                  ▼
 a shell prompt on the server, running in the laptop's terminal window
```

### Host keys: the server proving who it is

When `sshd` was installed, it generated a **host key pair**: a private key that never leaves the server and a public key it shows to anyone who connects. The first time the laptop connects it has never seen this key, so it shows a prompt like:

```text
The authenticity of host '192.168.100.x' can't be established.
ED25519 key fingerprint is SHA256:….
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Typing `yes` saves the fingerprint in `~/.ssh/known_hosts` on the laptop. From then on, `ssh` checks every connection against it silently. If the key ever changes, `ssh` refuses to connect with a loud warning, which is the protection against someone putting a different machine at the same address. On a home network the honest reasons the key changes are a reinstall of Ubuntu or the IP being reused by a different device; the fix is to delete the old line from `known_hosts`.

### Authentication: proving who *you* are

| Method | How it works | Where it stands here |
|---|---|---|
| **Password** | You type the password of the account you created in step 10 | What the installer enables by default and what this step uses |
| **Public key** | The laptop holds a private key; its public half is placed in `~/.ssh/authorized_keys` on the server. No password is sent at all | The better long-term setup: no password to type, and password login can then be turned off. Step 10's installer offered to import keys from GitHub for exactly this |

Password login over SSH is fine on a private home network. It would not be fine on a server exposed to the internet, which this one never is; [step 23](../11-public-access/23-cloudflare-tunnel.md) reaches the public internet through an outbound tunnel precisely so that port 22 never has to be opened.

### Why headless at all

| Working at the server's keyboard | Working over SSH from the laptop |
|---|---|
| A borrowed monitor and keyboard tying up a desk | The server sits by the router with nothing attached |
| Every command typed by hand; long `curl … \| sh -` lines invite typos | Copy and paste from this guide, from docs, from your own notes |
| One screen, one session | Several terminal tabs, scrollback, your own editor and tools |
| You must be in the room | Anywhere on the home network, or anywhere at all with a VPN later |

The only time the monitor comes back is when the network path itself is broken: a boot failure, a bad network change, a forgotten password. That is rare enough that borrowing is the right plan ([step 7](../02-hardware/07-display-keyboard-and-network.md)).

## When

Immediately after the first login in [step 10](10-installing-ubuntu-server.md), while the monitor is still plugged in. Before [step 12](../04-kubernetes/12-installing-k3s.md), which installs K3s over SSH.

## How

### 1. Update everything

At the server's keyboard, logged in as the user from the Profile screen:

Update everything:

```bash
sudo apt update && sudo apt upgrade -y
```

`sudo` asks for your password the first time. Expect `apt update` to print a few `Hit:` or `Get:` lines, one per repository, and finish with a count of upgradable packages. `apt upgrade -y` then lists what it will install and runs through it. A message about a pending kernel update or a suggested reboot is normal after a fresh install; a `sudo reboot` before continuing is harmless if it asks.

### 2. Find the server's IP

Find the server's IP:

```bash
hostname -I      # capital I
# or
ip -4 addr       # look for the inet line under the Ethernet entry (e.g. eno1)
```

`hostname -I` prints the address on one line, `192.168.100.x` for a home router on that range. That is the address to write down. If it prints more than one address later in the guide, the `192.168.x.x` one is the LAN address; the `10.42.x.x` ones belong to K3s.

`ip -4 addr` prints a block per interface. Find the Ethernet one (`eno1` here) and read the number after `inet`; the `/24` after it is the subnet mask, not part of the address.

It should match what the installer showed on its Network screen. If you reached for `hostname -i` by habit and saw `127.0.1.1`, re-read the Background: that is the machine's own loopback alias, not a network address.

### 3. SSH in from the laptop

From a laptop on the same network:

```bash
ssh <username>@<SERVER_IP>
```

`<username>` is the account from the Profile screen; `<SERVER_IP>` is the address from the previous step. The first connection shows the host-key fingerprint prompt described in the Background; type `yes`. Then the password prompt, which shows nothing as you type, then the same `user@quantum:~$` prompt you saw on the monitor, now inside a window on the laptop.

Run `hostname` to confirm the prompt belongs to the server and not the laptop. Then run the same `hostname -I` and get the same answer: you are on the box.

### 4. Go headless

From this point on I stopped using the monitor and did everything from my laptop, so I could copy and paste commands.

Unplug the monitor and keyboard. The server does not need them to keep running. The one setting that keeps it reachable unattended is **AC Recovery → Power On** from [step 9](09-bios-settings.md); the one thing still missing is a fixed address, which [step 14](../05-networking/14-dhcp-reservation.md) handles once the cluster exists.

Two habits worth starting now:

- Keep a note of the server's IP and username somewhere outside the server.
- When you leave an SSH session, `exit` (or `Ctrl+D`) rather than closing the window, so the shell ends cleanly.

## Challenges and solutions

Nothing failed for me here beyond the lowercase `-i` surprise. The traps for this step:

| Challenge | What was going on | Solution |
|---|---|---|
| `hostname -i` shows `127.0.1.1` | Lowercase `-i` reads the hostname's entry in `/etc/hosts`, which Ubuntu points at a loopback alias | Use `hostname -I` or `ip -4 addr`. See [Troubleshooting](../troubleshooting.md) |
| `ssh: connect to host … port 22: Connection refused` | OpenSSH server not installed (the step 10 tick-box), or wrong IP | On the server: `systemctl status ssh`. If missing: `sudo apt install openssh-server`. Re-check the IP |
| `ssh` hangs, then times out | Laptop and server on different networks (guest Wi-Fi, a second router), or a firewall between them | Put the laptop on the same LAN as the router the server is wired to |
| `Permission denied (publickey,password)` | Wrong username or password; the username is the one from the Profile screen, not your name | Check `whoami` on the server's console; passwords are case-sensitive |
| `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!` | The server's host key differs from the saved one, usually after a reinstall, or the IP now belongs to a different device | If you know why, remove the old line: `ssh-keygen -R <SERVER_IP>` on the laptop, then connect again |
| Server IP changed after a reboot or power cut | A DHCP lease is not permanent | Set a DHCP reservation in the router, [step 14](../05-networking/14-dhcp-reservation.md) |
| `hostname -I` shows `10.42.0.0 10.42.0.1` as well as the LAN IP | K3s pod-network interfaces (`flannel.1`, `cni0`), after step 12 | Only the `192.168.x.x` address is reachable from the LAN |

## Lessons

> 💡 **What I learned:**
> - `hostname -i` (lowercase) shows `127.0.1.1`, which is the machine talking to itself. Use **capital `-I`**.
> - **Reserve the server's IP in your router** (DHCP reservation) so it doesn't change after a reboot.
> - Typing long commands by hand on the server invites typos. SSH in and paste.

> 💡 **Also:** the host-key prompt on the first `ssh` is not a formality. It is the only moment the laptop learns which machine is "the server"; everything after that is checked against it.

## Glossary

| Term | Meaning |
|---|---|
| `apt` | Ubuntu's package manager; `update` refreshes the catalogue, `upgrade` installs newer versions |
| Repository / mirror | A signed online archive of packages; the mirror is the nearby copy chosen in step 10 |
| `sudo` | Run one command as the administrator (root) |
| `&&` | Run the next command only if the previous one succeeded |
| Network interface | A connection to a network; `lo`, `eno1`, and later `cni0` and `flannel.1` |
| Loopback | The address space a machine uses to talk to itself: `127.0.0.1`, and on Ubuntu the hostname alias `127.0.1.1` |
| `/etc/hosts` | A local file mapping names to addresses, checked before DNS; the source of `hostname -i`'s answer |
| `hostname -I` | Prints the address of every non-loopback interface; the right command |
| `ip -4 addr` | Lists every interface with its IPv4 address; the detailed form |
| DHCP lease | The address the router lends; it can change until [step 14](../05-networking/14-dhcp-reservation.md) |
| SSH | Secure Shell: an encrypted terminal session to another machine over TCP port 22 |
| `sshd` / OpenSSH server | The service on the server that answers SSH connections |
| Host key | The server's own key pair; its fingerprint is what the first-connection prompt shows |
| `known_hosts` | The laptop's file of host keys it has accepted; a changed key triggers a warning |
| Fingerprint | A short hash of a public key, used to recognise it |
| Password / public-key authentication | Proving who you are with the account password, or with a key pair whose public half is in `authorized_keys` |
| Headless | Running with no monitor or keyboard; managed over SSH |

---

[◀ Previous: Installing Ubuntu Server](10-installing-ubuntu-server.md) · [🏠 Home](../../README.md) · [Next: Installing Kubernetes (K3s) ▶](../04-kubernetes/12-installing-k3s.md)
