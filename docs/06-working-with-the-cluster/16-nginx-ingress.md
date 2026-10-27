# 16. Exposing an App With an Ingress

## What

In [step 13](../04-kubernetes/13-first-app.md) the whoami app was reachable at `http://192.168.100.22/` through an Ingress. Then, while testing, I ran nginx the quick way:

```bash
kubectl run nginx --image=nginx
kubectl expose pod nginx --port 5000 --target-port 80 --name nginx-svc
kubectl port-forward --address 0.0.0.0 svc/nginx-svc 8000:5000
```

That worked, but only while the `port-forward` command was running. This step replaces it with the permanent shape: a **Deployment**, a **Service**, and an **Ingress** rule, written as a manifest in `apps/nginx/`, routed by the Traefik ingress controller that K3s already runs.

## Why

| `port-forward` | Ingress |
|---|---|
| Dies when you close the terminal | Always on |
| One app per port, you have to remember which | One entry point (port 80) for every app, routed by host name or path |
| A debugging tool | How real clusters expose HTTP |
| Not a CKAD objective | Core CKAD objective |

There's also a cleanliness reason. `kubectl run` and `kubectl expose` are **imperative**: they create things with no record of how. A manifest in git is **declarative**: the repo *is* the description of what's running, and `kubectl apply -f` makes the cluster match.

### Host-based or path-based?

Two ways to tell Traefik which app a request belongs to:

| Style | Example | Catch |
|---|---|---|
| **Path** | `http://192.168.100.22/nginx` | nginx then receives a request for `/nginx`, which it doesn't have, and returns **404**. You'd need Traefik's strip-prefix middleware to rewrite the path. |
| **Host** | `http://nginx.home.arpa/` | Needs a name that resolves to the server. One `/etc/hosts` line on the laptop does it. |

I went with **host-based**. It keeps the app's own URLs untouched, scales to many apps without rewriting rules, and is how you'll run things once you have a domain and HTTPS. The whoami app from step 13 keeps its catch-all `/` rule, so anything that doesn't match a host still lands there.

**Why `.home.arpa`?** It's the domain reserved for home networks ([RFC 8375](https://www.rfc-editor.org/rfc/rfc8375)). It will never resolve on the public internet, so there's no risk of a typo sending you somewhere else. Avoid `.local`: macOS uses it for mDNS and lookups behave strangely.

## When

After [step 15](15-kubectl-from-laptop.md), so you can apply the manifest from the laptop. Also the moment you catch yourself leaving a `port-forward` running "for now".

## How

### 1. Remove the test pod and service

```bash
kubectl delete pod nginx
kubectl delete svc nginx-svc
```

### 2. The manifest

[`apps/nginx/nginx.yaml`](../../apps/nginx/nginx.yaml):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.27-alpine
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
  name: nginx
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx
spec:
  rules:
  - host: nginx.home.arpa
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx
            port:
              number: 80
```

Things worth noticing compared to the quick version:

- **Pinned image tag** (`nginx:1.27-alpine`) instead of `nginx`, which means "whatever latest is today".
- **Service port 80**, not 5000. There's no reason for the Service port to differ from the container port unless you have one.
- **Resource requests and limits**, because this is an 8GB machine and because CKAD will ask.
- **`host:` on the Ingress rule.** That one line is what makes it host-based.

### 3. Apply it from the laptop

```bash
kubectl apply -f apps/nginx/nginx.yaml
kubectl get deploy,svc,ingress nginx
```

```text
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   1/1     1            1           20s

NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/nginx   ClusterIP   10.43.x.x      <none>        80/TCP    20s

NAME                              CLASS     HOSTS             ADDRESS          PORTS   AGE
ingress.networking.k8s.io/nginx   traefik   nginx.home.arpa   192.168.100.22   80      20s
```

The `ADDRESS` column filling in with the server IP means Traefik has picked the rule up.

### 4. Test without touching DNS

You can prove routing works before adding any hosts entry, by sending the `Host` header by hand:

```bash
curl -s -H "Host: nginx.home.arpa" http://192.168.100.22 | head -5
```

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

And confirm the catch-all still goes to whoami:

```bash
curl -s http://192.168.100.22 | head -3
```

### 5. Make the name resolve on the laptop

```bash
sudo sh -c 'echo "192.168.100.22  nginx.home.arpa" >> /etc/hosts'
```

Now `http://nginx.home.arpa` in the browser shows the nginx welcome page. No port, no tunnel, no `port-forward`.

Each new app gets its own line in `/etc/hosts` for now. When that gets annoying, that's the signal to run a local DNS server (Pi-hole or AdGuard Home), which is on the long-term list.

### 6. Verify

```bash
kubectl describe ingress nginx          # rule, backend, and any events
kubectl logs deploy/nginx --tail=5      # the curl requests should appear here
```

## Challenges and solutions

| Challenge | What was going on | Solution |
|---|---|---|
| `kubectl get deploy,svc,ingress nginx` said all three were `NotFound` | I ran the check before the `apply`. | Order matters. Apply first, then get. |
| `kubectl get deploy, svc, ingress nginx` errored with *arguments in resource/name form must have a single resource and name* | Spaces after the commas. kubectl saw `deploy,` as one argument and `svc,` as another. | No spaces in the resource list: `deploy,svc,ingress`. |
| nginx log shows `GET /favicon.ico … 404` | The browser asks every site for a favicon; the default nginx image doesn't ship one. | Harmless. Ignore it, or ship a favicon when you deploy a real site. |
| nginx log shows the client as `10.42.0.43`, not my laptop's IP | That's the Traefik pod. Traffic goes laptop → Traefik → nginx, so nginx sees Traefik as the client. The last field in the log line, `10.42.0.1`, is the `X-Forwarded-For` header, and it's the node's internal bridge address, not the laptop either. | Expected on K3s defaults. Getting the real client IP through to the app needs `externalTrafficPolicy: Local` on the Traefik service, a topic for later. |

What went smoothly: the Ingress got its `ADDRESS` within seconds, the `Host` header curl proved routing before any DNS change, and the catch-all `/` rule from step 13 kept serving whoami.

## Lessons

> 💡 **What I learned:**
> - **`port-forward` is for looking, Ingress is for running.** If you'd be sad when the terminal closes, it should be an Ingress.
> - **Imperative to get it working, declarative to keep it.** `kubectl run` and `expose` are fine for a two-minute test, and `kubectl run … --dry-run=client -o yaml` turns that test into the manifest you commit.
> - **Host-based routing avoids the path-rewrite problem.** Apps assume they're at `/`. Give each one its own name instead of fighting that.
> - **`curl -H "Host: …"` tests the Ingress without DNS.** Separate "is routing right?" from "does the name resolve?" and each one is easy to debug.
> - **Use `.home.arpa` for home names.** Reserved for exactly this, never leaks to the internet, and doesn't collide with mDNS like `.local` does.

---

[◀ Previous: Running kubectl From the Laptop](15-kubectl-from-laptop.md) · [🏠 Home](../../README.md) · [Next: First Real Workload: The Portfolio Site ▶](../07-workloads/17-portfolio.md)
