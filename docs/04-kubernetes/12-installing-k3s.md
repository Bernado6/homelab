# 12. Installing Kubernetes (K3s)

```bash
curl -sfL https://get.k3s.io | sh -
```

Let your normal user run `kubectl`:

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
source ~/.bashrc
```

Check:

```bash
kubectl get nodes
kubectl get pods -A
```

My result:

```text
NAME      STATUS   ROLES           AGE   VERSION
quantum   Ready    control-plane   71s   v1.36.5+k3s1
```

| Pod (kube-system) | What it does |
|---|---|
| `coredns` | DNS inside the cluster, so apps find each other by name |
| `traefik` + `svclb-traefik` | Ingress controller: receives web traffic on ports 80/443 |
| `local-path-provisioner` | Creates storage on the SSD when an app asks for a volume |
| `metrics-server` | Makes `kubectl top` work |
| `helm-install-traefik(-crd)` | One-off install jobs, so **Completed** is correct |

> 💡 **What I learned:** K3s gives you a working cluster in about two minutes, with ingress and storage included. A couple of restarts on the `helm-install-traefik` job at first boot are normal; it retries until the cluster is ready.

---

[◀ Previous: First Login and SSH](../03-ubuntu-server/11-first-login-and-ssh.md) · [🏠 Home](../../README.md) · [Next: My First App ▶](13-first-app.md)
