# Daily Checklist

Run this at the start of every homelab session, **before** following any step. It takes about two minutes.

It is deliberately **not** automated. The kubeconfig for the homelab is not wired into `~/.zshrc`, so every new terminal starts without it. That is a choice: the CKAD exam begins every task with "switch to context X", and `kubectl config` should be muscle memory, not a line in a dotfile. If you'd rather make it permanent, [step 15, section 5](06-working-with-the-cluster/15-kubectl-from-laptop.md) shows how.

## 1. Point kubectl at the homelab (this shell only)

```bash
export KUBECONFIG="$HOME/.kube/config:$HOME/.kube/homelab.yaml"
kubectl config get-contexts
kubectl config use-context homelab
kubectl config current-context
```

`get-contexts` should now list `ckad`, `minikube` **and** `homelab`. If `homelab` is missing, the file isn't where the export says; check with `ls ~/.kube/`.

Things to practise while you're here, one per day:

| Drill | Command |
|---|---|
| Set a default namespace for the current context | `kubectl config set-context --current --namespace=portfolio` then `kubectl get pods` with no `-n` |
| Put it back | `kubectl config set-context --current --namespace=default` |
| See only the active context's config | `kubectl config view --minify` |
| Run one command against another cluster without switching | `kubectl --context=minikube get nodes` |
| Use a kubeconfig file directly, no export | `kubectl --kubeconfig ~/.kube/homelab.yaml get nodes` |

## 2. Is the cluster healthy?

```bash
kubectl get nodes
kubectl get pods -A | grep -v -E 'Running|Completed'
kubectl top nodes
```

Expected: `quantum Ready`, the grep prints **only the header line**, and memory on the node is somewhere around 35% with the current apps. Anything else is the first thing to look at, with the four-command sequence from [step 18's Challenges](08-observability/18-prometheus-grafana.md#challenges-and-solutions).

Optional, if something looks off:

```bash
kubectl get events -A --sort-by=.lastTimestamp | tail -15
```

## 3. Are the sites up?

```bash
curl -s -o /dev/null -w 'portfolio  %{http_code}\n' http://portfolio.home.arpa/healthz
curl -s -o /dev/null -w 'grafana    %{http_code}\n' http://grafana.home.arpa/api/health
curl -s -o /dev/null -w 'prometheus %{http_code}\n' http://prometheus.home.arpa/-/ready
```

Three `200`s. A connection error on all three usually means the laptop's `/etc/hosts` is fine but the server is down or has changed IP; `ping 192.168.100.22` tells you which.

## 4. Is the repo in sync?

```bash
cd ~/Desktop/MyProjects/mle/homelab
git status --short
git pull --ff-only
```

Uncommitted work from yesterday shows up here. Commit it before starting something new, so each step is its own commit.

## 5. Once a week: the server itself

```bash
ssh bernado@quantum 'uptime; df -h / | tail -1; apt list --upgradable 2>/dev/null | wc -l'
```

Uptime confirms no unexpected reboots, disk should be well under 50% with Prometheus retention capped, and the last number is pending updates. Apply them with `sudo apt update && sudo apt upgrade -y` and reboot if the kernel changed. Everything comes back on its own: AC Recovery from [step 9](03-ubuntu-server/09-bios-settings.md), K3s as a systemd service, the pods via their Deployments.

## End of session

```bash
git add -A && git commit -m "<what you did>" && git push
unset KUBECONFIG      # optional: the next terminal starts clean anyway
```

---

> 💡 **Why this page exists:** the day after setting up kubectl on the laptop, a new terminal said `error: no context exists with the name: "homelab"`. The export had only lived in the previous shell. Rather than fix it permanently, I kept it manual and turned it into a drill.

[🏠 Home](../README.md)
