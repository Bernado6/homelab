# 16. Exposing an App With an Ingress

This step is the first one where I wrote a manifest by hand instead of running commands, and the first time one entry point served more than one app. The manifest is short. The ideas behind it (declarative config, how the three objects link to each other, what the `Host` header is, why `.home.arpa`) are the ones every later step reuses, so they get the long treatment here.

## What

In [step 13](../04-kubernetes/13-first-app.md) the whoami app was reachable at `http://192.168.100.22/` through an Ingress. Then, while testing, I ran nginx the quick way:

```bash
kubectl run nginx --image=nginx
kubectl expose pod nginx --port 5000 --target-port 80 --name nginx-svc
kubectl port-forward --address 0.0.0.0 svc/nginx-svc 8000:5000
```

That worked, but only while the `port-forward` command was running. This step replaces it with the permanent shape: a **Deployment**, a **Service**, and an **Ingress** rule, written as a manifest in `apps/nginx/`, routed by the Traefik ingress controller that K3s already runs.

After this step:

- `http://nginx.home.arpa` works from the laptop with nothing running in a terminal.
- `http://192.168.100.22/` still shows whoami. Two apps, one IP, one port.
- The pattern for every app after this (portfolio, Grafana, Prometheus, AdGuard) is set: a host name per app, Traefik decides.

## Background: imperative vs declarative

There are two ways to make Kubernetes do something.

**Imperative** means telling it *what to do*: `kubectl run nginx --image=nginx`, `kubectl expose …`, `kubectl scale … --replicas=3`. Each command changes the cluster right now. Nothing records that you ran it. A week later, the only evidence is the objects themselves, and if one is deleted there is no way to know what it looked like.

**Declarative** means describing *what should exist* in a file, and asking Kubernetes to make reality match: `kubectl apply -f nginx.yaml`. The file is the source of truth. Run `apply` again and nothing changes, because reality already matches. Edit the file, `apply` again, and only the difference is changed. Delete the cluster, `apply` the whole folder, and everything comes back.

| | Imperative (`run`, `expose`, `create`, `scale`) | Declarative (`apply -f`) |
|---|---|---|
| Source of truth | The cluster | A file in git |
| Rerunning the same thing | Error: *already exists* | No-op |
| Changing one field | A different command for each field | Edit the file, `apply` |
| Rebuilding after a disaster | Remember every command | `kubectl apply -f apps/` |
| Code review | Impossible | A pull request |
| Where it shines | Two-minute tests, and CKAD speed | Anything you want to keep |

The two are not enemies. The fastest way to write a manifest is to let the imperative command generate it:

```bash
kubectl run nginx --image=nginx:1.27-alpine --port 80 --dry-run=client -o yaml > deploy.yaml
```

`--dry-run=client` means "don't send anything to the cluster", `-o yaml` means "print what you would have sent". That printed YAML, trimmed and committed, is the declarative version. CKAD rewards knowing this trick.

## Background: the three objects and how they link

A web app on Kubernetes is almost always three objects. Each one has a job, and each one points at the next through **labels** or **names**, never through IP addresses.

```text
 Ingress "nginx"                Service "nginx"                 Deployment "nginx"
 ┌────────────────────┐         ┌─────────────────────┐         ┌──────────────────────────┐
 │ host: nginx.home.arpa        │ selector: app=nginx │         │ replicas: 1              │
 │ path: /            │         │ port: 80            │         │ template:                │
 │ backend:           │──name──►│ targetPort: 80      │──label─►│   labels: app=nginx      │
 │   service: nginx   │         │ ClusterIP 10.43.x.x │         │   container nginx:1.27   │
 │   port: 80         │         └─────────────────────┘         │   containerPort: 80      │
 └────────────────────┘                                         └──────────────────────────┘
        ▲                                                                  │
        │ Traefik reads this                                               ▼
   request arrives at                                             Pod nginx-xxxx (10.42.x.x)
   192.168.100.22:80
```

### Deployment: "keep N copies of this pod running"

A Deployment owns a pod **template** and a replica count. It creates pods from the template, replaces them if they die, and when you change the template (a new image tag, say) it rolls the change out gradually. The pods it creates carry the labels from `template.metadata.labels`, and the Deployment finds its own pods through `spec.selector.matchLabels`. Those two must match, or `apply` is rejected.

### Service: "a stable address for whichever pods match this label"

Pods come and go and their IPs change every time. A Service is a fixed virtual IP (a **ClusterIP**, `10.43.x.x` on K3s) and a DNS name (`nginx.default.svc.cluster.local`) that load-balances across every pod whose labels match its `selector`. It has two ports:

| Field | Meaning | In this manifest |
|---|---|---|
| `port` | The port the Service listens on | 80 |
| `targetPort` | The port on the pod the traffic is sent to | 80 |

The quick version used `--port 5000 --target-port 80`, which is why `port-forward` needed `8000:5000` and the whole thing was confusing. There is no reason for the two to differ unless the app's port is awkward.

### Ingress: "when a request for this host and path arrives, send it to that Service"

An Ingress is a routing **rule**, nothing more. It doesn't run anything. An **ingress controller**, Traefik on K3s, watches every Ingress in the cluster and configures itself to match. Traefik is what actually listens on ports 80 and 443 of the node (through ServiceLB, explained properly in [step 21](../10-dns/21-local-dns-adguard.md)). The Ingress names the Service, and the Service finds the pods, so a request travels Ingress → Service → Pod without anyone hard-coding a pod address.

Everything here is in the `default` namespace, because no namespace is specified. The next step moves to a dedicated one.

## Background: how Traefik tells apps apart: the Host header

### What the Host header is

One IP address, one port 80, several apps. How does Traefik know which one you want? The answer is inside the HTTP request itself. Every HTTP/1.1 request starts like this:

```http
GET / HTTP/1.1
Host: nginx.home.arpa
User-Agent: curl/8.x
Accept: */*
```

The `Host` header is the name you typed in the address bar. The browser puts it there automatically, and it has done so since 1997, which is how one web server has always been able to host many sites on one IP. The TCP connection goes to `192.168.100.22:80` regardless of the name; the **name travels inside the request**, and Traefik reads it before deciding where to forward.

This gives you a powerful debugging move: you can set the header by hand with `curl -H "Host: nginx.home.arpa" http://192.168.100.22`. The connection goes to the IP, the header claims the name, Traefik routes by the header. No DNS needed. Section 4 of the How does exactly this.

### Host-based versus path-based, and the path-rewrite problem

Traefik can match on the `Host` header, on the path, or on both. Path-based routing sounds simpler because it needs no names: `http://192.168.100.22/nginx` for nginx, `/grafana` for Grafana. The catch shows up on the first request:

```text
 browser asks:    GET /nginx          Host: 192.168.100.22
 Traefik matches: path prefix /nginx  → Service nginx
 nginx receives:  GET /nginx          ← it has no file or directory called "nginx"
 nginx answers:   404 Not Found
```

The app was built assuming it lives at `/`. It has no idea Traefik put it under `/nginx`. To make it work you would add a **strip-prefix middleware** so Traefik removes `/nginx` before forwarding, and then the app's own links break instead, because it generates `<a href="/about">` and the browser asks Traefik for `/about`, which matches nothing. Some apps have a "base path" setting to cope with this. Many don't. Every one of them is a special case.

Host-based routing avoids all of it. Each app gets its own name, every app sees `GET /`, and nothing is rewritten. The cost is that each name has to resolve to the server, which for now is one `/etc/hosts` line per app and, from [step 21](../10-dns/21-local-dns-adguard.md), one wildcard DNS rule for all of them.

## Background: names for a home network

### `.home.arpa`, `.local`, and made-up domains

Three kinds of names people use for LAN services:

| Name style | Example | What happens |
|---|---|---|
| `.home.arpa` ✅ | `nginx.home.arpa` | Reserved by [RFC 8375](https://www.rfc-editor.org/rfc/rfc8375) for exactly this. Public DNS always says "no such name", so nothing leaks, and no one can ever register it and hijack a typo. |
| `.local` | `nginx.local` | Reserved for **mDNS** (Bonjour on macOS). The Mac does not send `.local` names to normal DNS at all; it multicasts a query on the LAN and waits. Hosts-file entries often work, DNS-server entries often don't, and lookups can be slow or inconsistent. |
| A domain you don't own | `nginx.homelab.dev`, `nginx.lan` | Might resolve on the public internet today or tomorrow. `.dev` is a real TLD with forced HTTPS in Chrome; `.lan` is unreserved and could be sold. |
| A domain you do own | `nginx.example.com` with a private subdomain | Fine, and the route to real public certificates later. Costs money, and this guide doesn't need it until Cloudflare Tunnel. |

The portfolio repo's manifests originally used `portfolio.homelab.local`; the next step changes it to `.home.arpa` for this reason.

### How a name becomes the server, today

Right now the laptop has no DNS server that knows these names. The stub resolver on the Mac checks `/etc/hosts` first, so one line per name makes the browser send its request to `192.168.100.22`, with the name in the `Host` header. That is the whole trick, and it is per-device: a phone cannot read the Mac's hosts file.

## Background: image tags

`image: nginx` means `docker.io/library/nginx:latest`. The `latest` tag is a moving pointer: whatever the nginx maintainers pushed most recently. Next month it is a different image, and your pod behaves differently without a single line of your manifest changing.

`nginx:1.27-alpine` names a specific minor version on a specific base image. It still receives patch updates behind that tag, which is a reasonable compromise; `1.27.3-alpine` would be stricter, and a `@sha256:…` digest is immovable. The point is that **the manifest should say what is running**, and `latest` says nothing. [Step 17](../07-workloads/17-portfolio.md) goes further, because `latest` also interacts badly with the image pull policy.

| Tag | Means | Changes under you? |
|---|---|---|
| `nginx` or `nginx:latest` | Whatever is newest | Every release |
| `nginx:1.27-alpine` | The 1.27 line on Alpine Linux | Patch releases only |
| `nginx:1.27.3-alpine` | Exactly that release | Only if someone re-pushes the tag |
| `nginx@sha256:…` | Exactly those bytes | Never |

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

### Why not a NodePort or LoadBalancer Service per app?

| Option | How you'd reach nginx | Why not |
|---|---|---|
| `type: NodePort` | `http://192.168.100.22:31234` | A random high port per app, nothing memorable, no HTTPS story |
| `type: LoadBalancer` | On K3s, the node IP on the Service's port | Only one app can own port 80 on the node. Right for things that are not HTTP (DNS on port 53 in step 21), wrong for websites. |
| `ClusterIP` + Ingress ✅ | `http://nginx.home.arpa` | One port 80 shared by every app, routed by name, and TLS can be added centrally later |

## How the pieces fit

```text
 laptop                                          server 192.168.100.22
 ──────                                          ─────────────────────
 /etc/hosts: 192.168.100.22  nginx.home.arpa
 browser: http://nginx.home.arpa
   │ TCP to 192.168.100.22:80
   │ GET /  Host: nginx.home.arpa
   ▼
 ───────────────────────────────────────────────► Traefik (ports 80/443 on the node)
                                                   ├─ Host = nginx.home.arpa?  → Service nginx → Pod nginx
                                                   └─ anything else, path /    → Service hello → Pod hello (whoami, step 13)
```

Two Ingresses, one Traefik. The nginx rule has a `host:`, the whoami rule doesn't, so whoami is the catch-all.

## When

After [step 15](15-kubectl-from-laptop.md), so you can apply the manifest from the laptop. Also the moment you catch yourself leaving a `port-forward` running "for now".

## How

### 1. Remove the test pod and service

```bash
kubectl delete pod nginx
kubectl delete svc nginx-svc
```

These remove the two objects the imperative commands created. `kubectl run` made a bare pod (not a Deployment), so `delete pod` is the right verb; `expose` made a Service called `nginx-svc`. Expect `pod "nginx" deleted` and `service "nginx-svc" deleted`. Deleting first avoids a name collision with the Service called `nginx` the manifest is about to create, and it means nothing from the quick version is left behind to confuse things later.

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

A line-by-line reading of the parts that are easy to get wrong:

| Line | What it does | Common mistake |
|---|---|---|
| `---` | Separates three documents in one file; `apply -f` creates all three | Forgetting it, so the second `apiVersion:` becomes a duplicate key error |
| `apiVersion: apps/v1` vs `v1` vs `networking.k8s.io/v1` | Each kind lives in an API group; Deployments in `apps`, Services in the core group, Ingresses in `networking.k8s.io` | Pasting a field from one kind into another; the error names the unknown field (see the [troubleshooting](../troubleshooting.md) entry from step 20) |
| `selector.matchLabels` and `template.metadata.labels` | Must match; this is how the Deployment finds its pods | Changing one and not the other |
| `Service.spec.selector: app: nginx` | Same label again; this is how the Service finds the pods | A typo here gives a Service with no endpoints and a 503 from Traefik |
| `pathType: Prefix` | `/` matches every path under it | Omitting it; the field is required |
| `backend.service.port.number: 80` | The **Service** port, not the container port | Putting a `containerPort` here when the two differ |
| `requests` vs `limits` | Requests are what the scheduler reserves; limits are the ceiling. No CPU limit on purpose, so nginx can burst. | Setting a memory limit below what the process needs, which gets it `OOMKilled` |

The file in the repo today has a few extra lines on the Ingress (`tls:` and two annotations) that [step 20](../09-tls/20-https-with-cert-manager.md) added for HTTPS. The YAML above is the step 16 version, which is all you need here.

### 3. Apply it from the laptop

```bash
kubectl apply -f apps/nginx/nginx.yaml
kubectl get deploy,svc,ingress nginx
```

`apply -f` sends all three documents to the API server; expect `deployment.apps/nginx created`, `service/nginx created`, `ingress.networking.k8s.io/nginx created`. The second command lists three resource types in one go, filtered to the name `nginx`. Note there are **no spaces** after the commas; that caught me (see Challenges). Run from the homelab repo root so the relative path resolves.

```text
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx   1/1     1            1           20s

NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/nginx   ClusterIP   10.43.x.x      <none>        80/TCP    20s

NAME                              CLASS     HOSTS             ADDRESS          PORTS   AGE
ingress.networking.k8s.io/nginx   traefik   nginx.home.arpa   192.168.100.22   80      20s
```

The `ADDRESS` column filling in with the server IP means Traefik has picked the rule up.

Reading the rest: `READY 1/1` means one pod wanted, one running and passing its checks. `TYPE ClusterIP` with `EXTERNAL-IP <none>` is correct; the Service is internal, and Traefik is the only thing that needs to reach it. `CLASS traefik` shows which ingress controller claimed the rule; K3s marks Traefik as the default class, so you don't have to name it.

### 4. Test without touching DNS

You can prove routing works before adding any hosts entry, by sending the `Host` header by hand:

```bash
curl -s -H "Host: nginx.home.arpa" http://192.168.100.22 | head -5
```

`-s` silences the progress meter, `-H` adds a header to the request, and `head -5` keeps the first five lines. The connection goes to the IP; the header tells Traefik which rule to use.

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

No `Host` header this time (curl sends `Host: 192.168.100.22`, which matches no host rule), so Traefik falls through to the whoami Ingress from step 13. Expect whoami's plain-text output, starting with `Hostname: hello-…`. If this instead returns the nginx page, the nginx Ingress is missing its `host:` line.

### 5. Make the name resolve on the laptop

```bash
sudo sh -c 'echo "192.168.100.22  nginx.home.arpa" >> /etc/hosts'
```

Same shape as the `quantum` line in [step 14](../05-networking/14-dhcp-reservation.md): the `sh -c` wrapper is what lets the `>>` append run as root. Check with `grep home.arpa /etc/hosts`.

Now `http://nginx.home.arpa` in the browser shows the nginx welcome page. No port, no tunnel, no `port-forward`.

Each new app gets its own line in `/etc/hosts` for now. When that gets annoying, that's the signal to run a local DNS server (Pi-hole or AdGuard Home), which is on the long-term list.

### 6. Verify

```bash
kubectl describe ingress nginx          # rule, backend, and any events
kubectl logs deploy/nginx --tail=5      # the curl requests should appear here
```

| Command | What to look for |
|---|---|
| `kubectl describe ingress nginx` | A `Rules` block showing `nginx.home.arpa` → `/` → `nginx:80`, and under it the pod endpoint `10.42.x.x:80`. If the endpoint shows `<error: endpoints … not found>`, the Service selector doesn't match the pod labels. |
| `kubectl logs deploy/nginx --tail=5` | The last five lines of nginx's access log. Each `curl` and browser visit is one line with the method, path and status. `deploy/nginx` picks a pod of that Deployment for you. |

A log line has the shape `10.42.0.43 - - [date] "GET / HTTP/1.1" 200 … "10.42.0.1"`. The first address is Traefik, not your laptop, and the last one is not your laptop either; the Challenges table explains why.

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

## Glossary

| Term | Meaning |
|---|---|
| Imperative | Commands that change the cluster directly (`run`, `expose`, `scale`); no record of intent |
| Declarative | A file describing the desired state; `kubectl apply -f` makes the cluster match |
| `--dry-run=client -o yaml` | Print the manifest a command would create, without creating it |
| Deployment | Keeps N copies of a pod template running and rolls out changes |
| Service | A stable ClusterIP and DNS name in front of the pods matching a label selector |
| `port` / `targetPort` | The Service's own port / the pod port it forwards to |
| ClusterIP | The default Service type: reachable only inside the cluster (`10.43.x.x`) |
| NodePort / LoadBalancer | Service types that expose a port on the node; not used for websites here |
| Ingress | A routing rule: host + path → Service. Does nothing without a controller. |
| Ingress controller | The proxy that implements Ingress rules. Traefik on K3s, listening on 80 and 443. |
| IngressClass | Which controller handles a rule; `traefik` is K3s's default |
| Label / selector | Key-value tags on objects, and the query that matches them; how Deployment, Service and pods link |
| `Host` header | The name the browser typed, carried inside every HTTP request; what Traefik routes on |
| Host-based routing | One name per app, apps see `/`, nothing rewritten |
| Path-based routing | One path prefix per app; needs prefix stripping and breaks app-generated links |
| Strip-prefix middleware | A Traefik rule that removes a path prefix before forwarding |
| `.home.arpa` | RFC 8375 domain reserved for home networks; never resolves publicly |
| `.local` / mDNS | Multicast DNS (Bonjour); macOS treats `.local` specially, avoid it |
| `/etc/hosts` | Per-device name → IP file, checked before DNS |
| Image tag | The part after `:` in an image name; `latest` is a moving pointer, `1.27-alpine` is pinned |
| Requests / limits | What the scheduler reserves for a container / the ceiling it may use |
| Catch-all rule | An Ingress rule with no `host:`; matches anything no other rule claims |
| `X-Forwarded-For` | Header a proxy adds with the original client address |

---

[◀ Previous: Running kubectl From the Laptop](15-kubectl-from-laptop.md) · [🏠 Home](../../README.md) · [Next: First Real Workload: The Portfolio Site ▶](../07-workloads/17-portfolio.md)
