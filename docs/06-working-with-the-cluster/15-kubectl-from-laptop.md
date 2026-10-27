# 15. Running kubectl From the Laptop

## What

Until now every `kubectl` command meant SSHing into the server first. This step copies the cluster credentials (the **kubeconfig**) to the laptop and points them at the server's fixed IP, so `kubectl get pods` works straight from a Mac terminal.

The kubeconfig is one YAML file with three things in it: the cluster's address and CA certificate, a user (a client certificate with full admin rights), and a **context** that ties the two together.

## Why

| Reason | Detail |
|---|---|
| **Faster loop** | Edit a manifest on the laptop, `kubectl apply` it, no copying files to the server. |
| **Real tooling** | VS Code's Kubernetes extension, `k9s`, Lens, and Helm all read the same kubeconfig. |
| **CKAD habit** | The exam gives you a terminal with contexts already configured. Switching between contexts should feel normal. |
| **Headless for real** | The server becomes a box you almost never log in to, which is what a server should be. |

This only became safe after [step 14](../05-networking/14-dhcp-reservation.md). The kubeconfig hard-codes the server address; if the IP drifted, the file would silently break.

## When

Right after the fixed IP, before you start writing real manifests. Everything from here on assumes commands run on the laptop unless the page says otherwise.

## How

### 1. Install kubectl on the Mac

```bash
brew install kubectl
kubectl version --client
```

If you already have minikube installed, you already have kubectl. Check with `which kubectl`.

### 2. Copy the kubeconfig from the server

K3s writes its config to `/etc/rancher/k3s/k3s.yaml`, and in [step 12](../04-kubernetes/12-installing-k3s.md) we copied it to `~/.kube/config` on the server. Pull that copy down to the laptop under a **separate name**, so it doesn't overwrite a minikube config you may already have:

```bash
mkdir -p ~/.kube
scp bernado@192.168.100.22:~/.kube/config ~/.kube/homelab.yaml
chmod 600 ~/.kube/homelab.yaml
```

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

K3s issues its API certificate with the node's IP and hostname included, so the TLS check passes against `192.168.100.22` (and against `quantum`, if your `/etc/hosts` maps it) without any extra flags.

### 4. Give the context a real name

K3s names the cluster, user and context all `default`. Rename the context so it's obvious which cluster you're talking to:

```bash
kubectl --kubeconfig ~/.kube/homelab.yaml config rename-context default homelab
```

### 5. Make kubectl find it

Two options:

**Option A, one file per cluster (what I did).** Tell kubectl to read several files. Add to `~/.zshrc`:

```bash
export KUBECONFIG="$HOME/.kube/config:$HOME/.kube/homelab.yaml"
alias k=kubectl
```

Then `source ~/.zshrc`. If `~/.kube/config` doesn't exist, that's fine; kubectl skips missing files.

**Option B, merge into one file.**

```bash
KUBECONFIG=~/.kube/config:~/.kube/homelab.yaml kubectl config view --flatten > ~/.kube/merged
mv ~/.kube/merged ~/.kube/config
chmod 600 ~/.kube/config
```

**Option C, per session (what I ended up doing).** Don't persist anything. Run the `export KUBECONFIG=…` line by hand in each new terminal, then `kubectl config use-context homelab`. It costs ten seconds a day and keeps the CKAD context-switching routine fresh. The [Daily Checklist](../daily-checklist.md) starts with exactly this.

### 6. Select the context and test

```bash
kubectl config get-contexts
kubectl config use-context homelab
kubectl get nodes
```

```text
NAME      STATUS   ROLES           AGE   VERSION
quantum   Ready    control-plane   2d    v1.36.5+k3s1
```

If that works, you can close the SSH session. From here on, `kubectl` on the laptop is the main way in.

### 7. Only if step 6 hangs: check the port

**Skip this if `kubectl get nodes` worked.**

On the **laptop**, check it can reach the API port at all:

```bash
nc -zv 192.168.100.22 6443
```

If that fails, SSH to the **server** and check the firewall:

```bash
sudo ufw status
```

Ubuntu Server ships with `ufw` **inactive**, so port 6443 is open by default and you'll see `Status: inactive`. Only if it says `active` do you need a rule, still on the server:

```bash
sudo ufw allow from 192.168.100.0/24 to any port 6443 proto tcp
```

`ufw` does not exist on macOS. If you see `sudo: ufw: command not found`, you ran it on the laptop.

## Keep this file private

The kubeconfig contains a client certificate that is **cluster-admin**. Anyone with the file controls the cluster.

- It lives in `~/.kube/`, never inside this repo.
- `chmod 600` so only your user can read it.
- If you ever paste it anywhere by mistake, rotate it: on the server, `sudo k3s certificate rotate` and re-copy.

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

---

[◀ Previous: A Fixed IP for the Server](../05-networking/14-dhcp-reservation.md) · [🏠 Home](../../README.md) · [Next: Exposing an App With an Ingress ▶](16-nginx-ingress.md)
