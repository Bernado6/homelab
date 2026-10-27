# 13. My First App

A tiny test app (`traefik/whoami`) using the three most important Kubernetes objects: a **Deployment**, a **Service** and an **Ingress**.

The manifest lives in this repo at [`apps/hello/hello.yaml`](../../apps/hello/hello.yaml):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
      - name: hello
        image: traefik/whoami
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 10m
            memory: 16Mi
          limits:
            memory: 64Mi
---
apiVersion: v1
kind: Service
metadata:
  name: hello
spec:
  selector:
    app: hello
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello
            port:
              number: 80
```

Apply it from the repo root:

```bash
kubectl apply -f apps/hello/hello.yaml
kubectl get pods,svc,ingress
```

Opening `http://<SERVER_IP>` in a browser shows the whoami page. Refresh and the `Hostname` switches between the two pods.

## How a request travels through the cluster

```text
Browser ──► <SERVER_IP>:80 ──► Traefik (Ingress controller)
                                   │  Ingress rule: "/" → Service "hello"
                                   ▼
                     Service "hello" (ClusterIP 10.43.x.x)
                                   │  load-balances across pods
                                   ▼
                     Pod hello-xxxxx (Pod IP 10.42.x.x)
```

| Address range | What it is |
|---|---|
| `192.168.x.x` | Home network |
| `10.43.x.x` | Kubernetes Service IPs (K3s default) |
| `10.42.x.x` | Pod IPs (K3s default) |

> 💡 **What I learned:** the whoami page is a great debugging tool. It shows the pod name, pod IP and the headers Traefik adds, so you can *see* the path the request took. Always set **resource requests and limits**, especially on an 8GB machine. These are core CKAD topics.

---

[◀ Previous: Installing Kubernetes (K3s)](12-installing-k3s.md) · [🏠 Home](../../README.md) · [Next: A Fixed IP for the Server ▶](../05-networking/14-dhcp-reservation.md)
