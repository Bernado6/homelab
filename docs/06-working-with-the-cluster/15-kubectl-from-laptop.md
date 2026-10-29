# 15. Running kubectl From the Laptop

A short step in commands, a long one in understanding. The file you copy here is the key to the whole cluster, and once you know what is inside it, most of `kubectl config` stops feeling like incantation. This page spends more time on the file than on the copying.

## What

Until now every `kubectl` command meant SSHing into the server first. This step copies the cluster credentials (the **kubeconfig**) to the laptop and points them at the server's fixed IP, so `kubectl get pods` works straight from a Mac terminal.

The kubeconfig is one YAML file with three things in it: the cluster's address and CA certificate, a user (a client certificate with full admin rights), and a **context** that ties the two together.

After this step:

- `kubectl` on the Mac talks to the homelab directly over the LAN.
- The homelab sits beside `minikube` and `ckad` as one more **context**, and switching between them is a one-line habit.
- The SSH session becomes something you open rarely, for the server itself, not for the cluster.

## Background: what a kubeconfig contains

### How kubectl talks to a cluster

`kubectl` is a plain HTTPS client. Every command it runs becomes a request to the **API server**, which on K3s listens on port **6443**. `kubectl get nodes` is `GET /api/v1/nodes` with a certificate attached. Nothing else about the cluster matters to `kubectl`: not where the pods run, not which node you're on, only "where is the API server and how do I prove who I am".

```text
 laptop                                            server 192.168.100.22
 ──────                                            ─────────────────────
 kubectl get nodes
   │  reads ~/.kube/homelab.yaml
   │  ├─ server:  https://192.168.100.22:6443   ─────────► kube-apiserver :6443
   │  ├─ CA cert: "is this really my API server?"           │ presents its serving certificate
   │  └─ client cert + key: "this is who I am"              │ checks the client cert against its own CA
   │                                                        │ authorizes: this cert is cluster-admin
   ▼                                                        ▼
 NAME      STATUS   ROLES  ◄──────────────────────────── JSON list of nodes
```

Two TLS checks happen, in both directions. The laptop checks the server's certificate against the **CA** in the kubeconfig, so it can't be tricked into sending its credentials to the wrong machine. The server checks the laptop's **client certificate** against the same CA, so it knows who is calling. Both certificates were issued by the CA that K3s generated at install time.

### The three sections

A kubeconfig is YAML with three lists and one pointer. Stripped to its skeleton, the file K3s writes looks like this:

```yaml
apiVersion: v1
kind: Config
clusters:                              # WHERE: one entry per cluster
- name: default
  cluster:
    server: https://127.0.0.1:6443     # the API server address
    certificate-authority-data: LS0t…  # base64 of the CA cert, used to verify the server
users:                                 # WHO: one entry per identity
- name: default
  user:
    client-certificate-data: LS0t…     # base64 of the client cert (this one is cluster-admin)
    client-key-data: LS0t…             # base64 of its private key. This is the secret.
contexts:                              # WHICH PAIR: a cluster + a user (+ optional namespace)
- name: default
  context:
    cluster: default
    user: default
current-context: default               # the pointer: which context kubectl uses by default
```

| Section | Answers | Contains |
|---|---|---|
| `clusters` | *Where* is the API server, and how do I know it's genuine? | `server` URL and the CA certificate |
| `users` | *Who* am I? | A client certificate and private key, or a token, or an exec plugin for cloud clusters |
| `contexts` | *Which* cluster, as *which* user, in *which* namespace by default? | A name plus three references |
| `current-context` | Which context applies when I don't say? | One name |

The `-data` suffix means the certificate is embedded inline as base64, which is why the file is self-contained and can be copied as a single unit. The alternative form, `certificate-authority: /path/to/ca.crt`, points at a file on disk and breaks the moment you move the config to another machine.

Lists mean one file can hold many clusters. A `minikube` cluster, a `ckad` practice cluster and the homelab can all live in one kubeconfig, each with its own entry in all three lists, and a context per combination. That is what the CKAD exam environment looks like.

### Why the copied file says `127.0.0.1`

K3s writes its kubeconfig for use **on the node**, so the server address is `https://127.0.0.1:6443`, the loopback address. On the server, that is the API server. On the laptop, `127.0.0.1` is the laptop itself, where nothing listens on 6443, so `kubectl` fails with a connection refused. The fix is one line: replace the loopback address with the node's LAN IP. That is why [step 14](../05-networking/14-dhcp-reservation.md) had to come first.

### Reading your own kubeconfig safely

`kubectl config view` prints the merged config with every certificate replaced by `DATA+OMITTED` and keys by `REDACTED`, so it is safe to look at on a shared screen. `cat ~/.kube/homelab.yaml` is not: it prints the private key.

## Background: TLS SANs, and why the K3s certificate accepts the node IP

When a browser or `kubectl` connects to `https://192.168.100.22:6443`, verifying the certificate means two things: it must be signed by a CA we trust (the one in the kubeconfig), **and** it must be *for* the address we typed. The second check is done against the certificate's **Subject Alternative Names** (SANs), a list of DNS names and IP addresses the certificate is valid for.

If `192.168.100.22` were not in that list, `kubectl` would refuse with an error like *certificate is valid for 127.0.0.1, not 192.168.100.22*, and the usual bad advice is to add `--insecure-skip-tls-verify`. That switches off the check entirely, which defeats the point of having a CA in the file.

K3s avoids the problem by generating its API server certificate with a generous SAN list at install time. The list it generates includes the node's own IP address and hostname, `127.0.0.1` and `localhost`, and the in-cluster names `kubernetes`, `kubernetes.default`, `kubernetes.default.svc` and `kubernetes.default.svc.cluster.local`. That is why editing the `server:` line is enough: both `192.168.100.22` and `quantum` are already names the certificate is valid for.

If you ever need another name on it, for instance a public DNS name, K3s has an install option `--tls-san <name>` that adds it. You can see the list yourself from the laptop with:

```bash
openssl s_client -connect 192.168.100.22:6443 </dev/null 2>/dev/null | openssl x509 -noout -ext subjectAltName
```

The first `openssl` opens a TLS connection and prints the certificate; the second parses out the SAN extension.

## Background: KUBECONFIG merging, contexts and namespaces

### Where kubectl looks for config

`kubectl` finds its configuration in this order; the first one that applies wins:

| Priority | Source | Example |
|---|---|---|
| 1 | `--kubeconfig` flag on the command | `kubectl --kubeconfig ~/.kube/homelab.yaml get nodes` |
| 2 | `KUBECONFIG` environment variable | `export KUBECONFIG=~/.kube/config:~/.kube/homelab.yaml` |
| 3 | The default file | `~/.kube/config` |

The `KUBECONFIG` variable can hold **several paths separated by colons**, like `PATH`. `kubectl` reads all of them and merges them in memory, every time it runs. Nothing is written to disk. Rules of the merge:

- Entries are combined by name across all files. Two files each with a cluster called `default` collide; the **first file listed wins**.
- `current-context` is taken from the first file that sets one.
- Files that don't exist are skipped silently.

This is why giving the homelab's cluster, user and context a distinctive name matters. K3s calls all three `default`, and so does nearly every other tool.

### Contexts

A context is a saved combination: this cluster, as this user, in this namespace. Switching contexts switches all three at once. The commands you'll use daily:

| Command | What it does |
|---|---|
| `kubectl config get-contexts` | List every context in the merged config; `*` marks the current one |
| `kubectl config current-context` | Print the current one |
| `kubectl config use-context homelab` | Make `homelab` the default for every following command. Writes `current-context` into the first kubeconfig file. |
| `kubectl --context=minikube get nodes` | Run one command against another context without switching |
| `kubectl config rename-context default homelab` | Rename a context. Only the context name changes; the cluster and user entries it points at keep theirs. |
| `kubectl config view --minify` | Show only the parts of the config the current context uses |

### Namespaces live in the context

Every context can carry a `namespace:` field. When it does, `kubectl get pods` means "pods in that namespace" and you can drop `-n`. When it doesn't, the default is the namespace called `default`. Setting it is a context change, not a cluster change:

```bash
kubectl config set-context --current --namespace=portfolio
```

The [Daily Checklist](../daily-checklist.md) uses this as a drill. It is also a classic CKAD time-saver: set the namespace once at the start of a question instead of typing `-n` twenty times.

## Why

| Reason | Detail |
|---|---|
| **Faster loop** | Edit a manifest on the laptop, `kubectl apply` it, no copying files to the server. |
| **Real tooling** | VS Code's Kubernetes extension, `k9s`, Lens, and Helm all read the same kubeconfig. |
| **CKAD habit** | The exam gives you a terminal with contexts already configured. Switching between contexts should feel normal. |
| **Headless for real** | The server becomes a box you almost never log in to, which is what a server should be. |

This only became safe after [step 14](../05-networking/14-dhcp-reservation.md). The kubeconfig hard-codes the server address; if the IP drifted, the file would silently break.

### Why not just keep using SSH?

| Approach | What it costs |
|---|---|
| SSH in, run `kubectl` there | Every manifest has to be copied to the server first, or edited there in `vim`. No local editor, no local tooling, and the habit of treating the server as a workstation. |
| `kubectl` on the laptop, kubeconfig copied ✅ | One file to protect. Everything else gets easier. |
| `ssh -L 6443:localhost:6443` and keep `127.0.0.1` in the file | Works, but the tunnel has to be up for every command, and it hides the SAN lesson above. Useful only if the API port were firewalled. |

## When

Right after the fixed IP, before you start writing real manifests. Everything from here on assumes commands run on the laptop unless the page says otherwise.

## How

### 1. Install kubectl on the Mac

```bash
brew install kubectl
kubectl version --client
```

`brew install kubectl` fetches the client binary. `kubectl version --client` prints only the client's version and skips contacting any cluster, so it works before any kubeconfig exists. A version close to the server's (`v1.36.x` here) is ideal; `kubectl` officially supports one minor version of skew in either direction.

If you already have minikube installed, you already have kubectl. Check with `which kubectl`.

### 2. Copy the kubeconfig from the server

K3s writes its config to `/etc/rancher/k3s/k3s.yaml`, and in [step 12](../04-kubernetes/12-installing-k3s.md) we copied it to `~/.kube/config` on the server. Pull that copy down to the laptop under a **separate name**, so it doesn't overwrite a minikube config you may already have:

```bash
mkdir -p ~/.kube
scp bernado@192.168.100.22:~/.kube/config ~/.kube/homelab.yaml
chmod 600 ~/.kube/homelab.yaml
```

| Command | What it does |
|---|---|
| `mkdir -p ~/.kube` | Create the directory kubectl expects; `-p` makes it a no-op if it already exists |
| `scp bernado@…:~/.kube/config ~/.kube/homelab.yaml` | Secure copy over SSH: remote path on the left, local destination on the right. It prompts for the same password or key as `ssh`. |
| `chmod 600 ~/.kube/homelab.yaml` | Owner may read and write, nobody else may even read. `kubectl` warns loudly if a kubeconfig is readable by others. |

Expect `scp` to print a one-line progress bar ending in `100%`. If it says `No such file or directory`, the server-side copy from step 12 was skipped; `sudo cat /etc/rancher/k3s/k3s.yaml` on the server shows the original, which is root-only.

### 3. Point it at the server

Open the file and look at the `server:` line:

```bash
grep server: ~/.kube/homelab.yaml
```

```text
    server: https://127.0.0.1:6443
```

That address is correct **on the server** and wrong everywhere else. Change it to the fixed IP:

```bash
sed -i '' 's#https://127.0.0.1:6443#https://192.168.100.22:6443#' ~/.kube/homelab.yaml
```

Reading the `sed`: `-i ''` edits the file in place (the empty quotes are macOS `sed` syntax for "no backup file"; on Linux it would be just `-i`). The `s#old#new#` is a substitution using `#` as the separator, because the URL itself contains slashes. Run the `grep` again afterwards; it should now show `192.168.100.22`.

K3s issues its API certificate with the node's IP and hostname included, so the TLS check passes against `192.168.100.22` (and against `quantum`, if your `/etc/hosts` maps it) without any extra flags.

### 4. Give the context a real name

K3s names the cluster, user and context all `default`. Rename the context so it's obvious which cluster you're talking to:

```bash
kubectl --kubeconfig ~/.kube/homelab.yaml config rename-context default homelab
```

`--kubeconfig` points this one command at the new file only, so the rename cannot touch a `default` context in some other file. The output is `Context "default" renamed to "homelab".` The cluster and user entries inside the file are still called `default`; that's fine, because the context name is the one you type.

### 5. Make kubectl find it

Two options:

**Option A, one file per cluster (what I did).** Tell kubectl to read several files. Add to `~/.zshrc`:

```bash
export KUBECONFIG="$HOME/.kube/config:$HOME/.kube/homelab.yaml"
alias k=kubectl
```

Then `source ~/.zshrc`. If `~/.kube/config` doesn't exist, that's fine; kubectl skips missing files.

The `alias k=kubectl` is not decoration. You will type `kubectl` several hundred times; `k` saves a lot of keystrokes and catches a lot of typos, and it is the alias most CKAD candidates practise with.

**Option B, merge into one file.**

```bash
KUBECONFIG=~/.kube/config:~/.kube/homelab.yaml kubectl config view --flatten > ~/.kube/merged
mv ~/.kube/merged ~/.kube/config
chmod 600 ~/.kube/config
```

`config view --flatten` prints the merged result with every certificate inlined, so the output is a complete standalone file. Writing it to `merged` first and then moving it over `config` avoids truncating the file you are reading from. Back up `~/.kube/config` before trying this.

**Option C, per session (what I ended up doing).** Don't persist anything. Run the `export KUBECONFIG=…` line by hand in each new terminal, then `kubectl config use-context homelab`. It costs ten seconds a day and keeps the CKAD context-switching routine fresh. The [Daily Checklist](../daily-checklist.md) starts with exactly this.

| | A: `KUBECONFIG` in `.zshrc` | B: merge into one file | C: per session |
|---|---|---|---|
| Setup | One line, once | Three commands, once | Nothing |
| Removing a cluster later | Delete its file | Edit a shared file carefully | Delete its file |
| Risk | None | Mistyped redirect can wipe `config` | Forgetting the export (see Challenges) |
| Daily cost | None | None | One line per terminal |
| What it trains | Nothing | Nothing | Context switching |

### 6. Select the context and test

```bash
kubectl config get-contexts
kubectl config use-context homelab
kubectl get nodes
```

| Command | What to expect |
|---|---|
| `kubectl config get-contexts` | A table with `homelab` in it, alongside any `minikube` or `ckad` context. The `*` column marks the current one. |
| `kubectl config use-context homelab` | `Switched to context "homelab".` |
| `kubectl get nodes` | The first command that actually crosses the network. If TLS, the address or the port are wrong, this is where it shows. |

```text
NAME      STATUS   ROLES           AGE   VERSION
quantum   Ready    control-plane   2d    v1.36.5+k3s1
```

One line, one node, `Ready`. That single row proves the IP is right, port 6443 is reachable, the CA verified the server, and the client certificate was accepted as an admin.

If that works, you can close the SSH session. From here on, `kubectl` on the laptop is the main way in.

### 7. Only if step 6 hangs: check the port

**Skip this if `kubectl get nodes` worked.**

On the **laptop**, check it can reach the API port at all:

```bash
nc -zv 192.168.100.22 6443
```

`nc` (netcat) with `-z` tries to open a TCP connection without sending data, and `-v` reports the result. `Connection to 192.168.100.22 port 6443 [tcp/*] succeeded!` means the network path is fine and the problem is in the kubeconfig. A timeout means something between the laptop and the API server is blocking the port.

If that fails, SSH to the **server** and check the firewall:

```bash
sudo ufw status
```

Ubuntu Server ships with `ufw` **inactive**, so port 6443 is open by default and you'll see `Status: inactive`. Only if it says `active` do you need a rule, still on the server:

```bash
sudo ufw allow from 192.168.100.0/24 to any port 6443 proto tcp
```

That rule allows TCP 6443 from any address on the home LAN and nothing else, which is the right scope: the API server should never be reachable from outside the house.

`ufw` does not exist on macOS. If you see `sudo: ufw: command not found`, you ran it on the laptop.

## Keep this file private

The kubeconfig contains a client certificate that is **cluster-admin**. Anyone with the file controls the cluster.

- It lives in `~/.kube/`, never inside this repo.
- `chmod 600` so only your user can read it.
- If you ever paste it anywhere by mistake, rotate it: on the server, `sudo k3s certificate rotate` and re-copy.

What "controls the cluster" means in practice: with this file someone can read every Secret, run any container, mount the node's filesystem into a pod, and from there do anything on the server. It is strictly more powerful than the SSH password, because it skips SSH entirely. The repo's `.gitignore` excludes kubeconfig files as a backstop, but the real protection is never putting one near a repo in the first place.

## Challenges and solutions

| Challenge | What was going on | Solution |
|---|---|---|
| Ran the `ufw` command and got `sudo: ufw: command not found` after three password prompts | I ran it on the Mac. `ufw` is an Ubuntu tool, and the step was conditional anyway. `kubectl get nodes` had already worked. | Skip step 7 when step 6 works. When a page says *on the server*, check the prompt says `bernado@quantum` first. |
| Every pod showed `RESTARTS 4 (77m ago)` | The reboot in [step 14](../05-networking/14-dhcp-reservation.md). Kubernetes restarted the containers when the node came back. | Nothing to fix. Restart counts persist for the life of the pod and tell you how many times the node has bounced. |
| Next day, new terminal: `error: no context exists with the name: "homelab"` and `get-contexts` only listed `ckad` and `minikube` | The `export KUBECONFIG=…` had been typed in yesterday's shell and never added to `~/.zshrc`. Without it, kubectl reads only `~/.kube/config`, which is minikube's. | Chose to keep it manual on purpose. The export is now the first line of the [Daily Checklist](../daily-checklist.md). |

What went smoothly: `scp`, the `sed` on the server address, the context rename, and `KUBECONFIG` with two files. The K3s certificate accepted `192.168.100.22` with no extra flags, as expected.

## Lessons

> 💡 **What I learned:**
> - **The kubeconfig is just a file.** Cluster address, CA, client cert, context. Once you've read one, `kubectl config` commands stop being magic.
> - **`127.0.0.1` in a copied config is the classic trap.** It's right on the machine it came from and wrong on every other one.
> - **Keep one file per cluster and let `KUBECONFIG` join them.** Merging is fine too, but separate files make it obvious what to delete when a cluster goes away.
> - **Name your contexts.** `default` tells you nothing. `homelab` and `minikube` side by side is the CKAD exam experience in miniature.
> - **A kubeconfig is a credential.** Treat it like an SSH key.

## Glossary

| Term | Meaning |
|---|---|
| kubeconfig | The YAML file that tells `kubectl` where a cluster is and how to authenticate. Default location `~/.kube/config`. |
| API server | The HTTPS endpoint every `kubectl` command talks to. Port 6443 on K3s. |
| `clusters` | Kubeconfig section: server URL and CA certificate, one entry per cluster |
| `users` | Kubeconfig section: an identity, here a client certificate and private key |
| `contexts` | Kubeconfig section: a named cluster + user (+ namespace) combination |
| `current-context` | The context used when no `--context` is given |
| CA (certificate authority) | The signer both sides trust. K3s generates one at install; it signs the API server's certificate and the admin's client certificate. |
| Client certificate | A certificate presented by the client to prove identity; the K3s one is cluster-admin |
| SAN (Subject Alternative Name) | The list of names and IPs a certificate is valid for. `192.168.100.22` and `quantum` are on the K3s one. |
| `--tls-san` | K3s install option to add another name to that list |
| `127.0.0.1` / loopback | "This machine". Correct on the server, wrong in a copied file. |
| `KUBECONFIG` | Environment variable: a colon-separated list of kubeconfig files, merged in memory, first file wins on conflicts |
| `--kubeconfig` | Per-command flag that overrides `KUBECONFIG` |
| `kubectl config view --flatten` | Print the merged config with certificates inlined, as one standalone file |
| Namespace (in a context) | The default namespace for commands run in that context; set with `set-context --current --namespace=` |
| `scp` | Copy files over SSH |
| `chmod 600` | Owner read/write only |
| `nc -zv` | Netcat port check: can I open a TCP connection to this host and port? |
| `ufw` | Ubuntu's firewall front end; inactive by default on Ubuntu Server |
| `k3s certificate rotate` | Reissue the cluster's certificates; the way to invalidate a leaked kubeconfig |

---

[◀ Previous: A Fixed IP for the Server](../05-networking/14-dhcp-reservation.md) · [🏠 Home](../../README.md) · [Next: Exposing an App With an Ingress ▶](16-nginx-ingress.md)
