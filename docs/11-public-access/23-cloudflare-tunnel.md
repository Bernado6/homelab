# 23. Public Access With Cloudflare Tunnel

> **Prerequisite: [step 22](22-domain-and-cloudflare.md)**, a domain on a Cloudflare account with Zero Trust set up. The tunnel itself is free.

## What

Make `https://benardkipngeno.com` reach the portfolio running on the homelab, from anywhere, with a certificate browsers already trust, **without opening a single port on the router**.

The portfolio lives at the **bare domain** (the apex), with `www.benardkipngeno.com` as an alias, not at a `portfolio.` subdomain. A personal site belongs at the root; subdomains are for later things like a blog or a second app.

A small pod (`cloudflared`) opens an outbound connection to Cloudflare's network and keeps it open. When someone visits your domain, Cloudflare terminates TLS at its edge and relays the request down that connection to the portfolio Service inside the cluster.

## Why

### The options for public access

| Option | Opens router ports? | Home IP visible? | TLS | Works behind Safaricom's CGNAT? | Cost |
|---|---|---|---|---|---|
| **Cloudflare Tunnel** ✅ | No | No | Cloudflare's, trusted everywhere | **Yes**, outbound only | Free + domain |
| Port forwarding 80/443 | Yes | Yes | Yours, via Let's Encrypt | Often **no**: many home-fibre connections share a public IP, so inbound ports never reach you | Free + domain |
| Tailscale Funnel | No | No | Tailscale's | Yes | Free, but the name is `quantum.<tailnet>.ts.net`, not yours |
| Keep using S3 + CloudFront | n/a | n/a | AWS | n/a | Already live; a few dollars a month |

Cloudflare Tunnel is the only row with no inbound exposure, your own domain, and immunity to carrier-grade NAT. It's also what a lot of small companies do for internal tools, so it transfers.

### Why not just keep CloudFront?

The CloudFront deployment is public today and nothing is wrong with it. The homelab copy becomes the primary once the tunnel is up, which lets you retire the AWS cost, and you learn the pattern that every later app will use if you ever want it reachable away from home. Keep CloudFront until the tunnel has run for a few weeks, then decide.

### What stays private

Only the portfolio. Grafana, Prometheus and AdGuard stay on `*.home.arpa`, reachable at home only. Exposing an admin UI to the internet, even behind a password, is a different risk class, and the right tool for that is Cloudflare Access in front of the tunnel, which is a later step if you ever need it.

## When

The day the domain exists. Nothing in the cluster has to change before then.

## How

### 1. Domain and Cloudflare

Done in [step 22](22-domain-and-cloudflare.md): the domain is registered, its zone is on Cloudflare, and Zero Trust is set up. Everything below happens in **Cloudflare → Zero Trust**.

### 2. Create the tunnel

Main sidebar, under **Protect & connect: Networking → Tunnels → Create a tunnel → Cloudflared.** (Older guides say *Zero Trust → Networks → Tunnels*; the redesign moved it.) Name it `homelab`. On the next screen Cloudflare shows install commands for various platforms; you only need the **token**, the long string after `--token`. Copy it. It's a credential.

### 3. Put the token in the cluster

```bash
kubectl create namespace cloudflared
kubectl -n cloudflared create secret generic tunnel-token --from-literal=token='<paste-token>'
```

Created with `kubectl`, referenced by name from the manifest, never in git. Same pattern as the Grafana password in step 18.

### 4. Deploy cloudflared

[`apps/cloudflared/cloudflared.yaml`](../../apps/cloudflared/cloudflared.yaml): a Deployment with two replicas, non-root, read-only filesystem, the token from the Secret, and probes on cloudflared's own `/ready` endpoint.

```bash
kubectl apply -f apps/cloudflared/cloudflared.yaml
kubectl -n cloudflared get pods
kubectl -n cloudflared logs deploy/cloudflared --tail=5
```

The log should say `Registered tunnel connection` four times across the two pods (each pod opens two connections). In the Cloudflare dashboard the tunnel flips to **Healthy**.

### 5. The public hostnames

Back in the tunnel's config, **Public Hostname → Add a public hostname**, twice:

| Field | First hostname (the apex) | Second hostname (the alias) |
|---|---|---|
| Subdomain | *(leave empty)* | `www` |
| Domain | `benardkipngeno.com` | `benardkipngeno.com` |
| Type | `HTTP` | `HTTP` |
| URL | `portfolio.portfolio.svc.cluster.local:80` | `portfolio.portfolio.svc.cluster.local:80` |

That URL is the portfolio **Service**, by its in-cluster DNS name (`<service>.<namespace>.svc.cluster.local`). The tunnel pod talks to it directly over the pod network, bypassing Traefik entirely. Cloudflare creates the DNS records for you: a proxied CNAME at the apex (using CNAME flattening, since a plain CNAME isn't allowed at the root of a zone) and one for `www`.

Both names serve the same site. To make `www` redirect to the bare domain instead of serving a copy, add a redirect later under **Rules → Redirect Rules**; optional, and search engines treat the two as one site either way.

Open `https://benardkipngeno.com` from your phone on mobile data. Padlock, public certificate, no warnings.

### 6. Verify the path

```bash
curl -sI https://benardkipngeno.com | grep -iE '^(HTTP|server|cf-ray)'
curl -sI https://www.benardkipngeno.com | head -1
```

`server: cloudflare` and a `cf-ray` header prove the request went through Cloudflare's edge. In the portfolio's nginx log the client address will be the cloudflared pod, with the real visitor IP in `CF-Connecting-IP`.

### 7. Tighten what Cloudflare passes through

In the Cloudflare dashboard for the domain, two settings worth flipping now:

- **SSL/TLS → Overview:** `Full`. Cloudflare to origin is plain HTTP here because the tunnel itself is encrypted, so `Full` versus `Flexible` doesn't change security, but `Full` is the habit to keep for when an origin does speak TLS.
- **Security → Bots:** turn on bot fight mode. Free, and it cuts the noise in your logs.

### Later: Let's Encrypt through the same cert-manager

Once Cloudflare manages the domain, a `letsencrypt` ClusterIssuer with the Cloudflare DNS-01 solver can issue public certificates for any name under it, including ones served only at home. Step 20's Ingresses change one annotation value. Not needed for the tunnel, but it's the natural extension.

## Challenges and solutions

| Challenge | What was going on | Solution |
|---|---|---|
| Couldn't find "Public Hostname" anywhere | The dashboard redesign renamed it. Tunnels live under **Networking → Tunnels**, and hostnames are now **Routes**. | **Add route → Published application.** The other three route types (Private hostname, Private CIDR, Workers VPC) are for reaching private things *from* your devices, the opposite direction. |
| The route form had a single "Service URL" field, not a type and an address | New form layout. | `http://portfolio.portfolio.svc.cluster.local:80`, with the `http://` prefix, because the Service speaks plain HTTP inside the cluster. |
| Replicas table showed `2024.12.2 · Unsupported` in red | I pinned the image to a release Cloudflare has since dropped support for. The tunnel still worked, hence `Healthy`. | Looked up the newest tag (`2026.10.6`, published that same day), put it in the manifest, and `kubectl apply -f` so the file stays the source of truth. The rollout replaced one pod at a time (`1 out of 2 new replicas have been updated…`, then `1 old replicas are pending termination…`) and `curl` returned 200 throughout. The newer build logs a `precheck` of TCP connectivity and API reachability before connecting, which the old one didn't. |
| `kubectl apply` warned that the namespace was *missing the last-applied-configuration annotation* | I had run `kubectl create namespace cloudflared` by hand before applying a manifest that also contains the Namespace. | Harmless; kubectl patches the annotation itself. Next time, skip the manual `create` and let the manifest own the namespace. |
| Tunnel name field looked empty on the create screen | The name was there as grey placeholder text. | It saved as `homelab` anyway. |

What went smoothly: the token Secret, the Deployment, four tunnel connections within 15 seconds, and both routes resolving on the first try.

My verification, about a minute after adding the routes:

```text
$ kubectl -n cloudflared logs deploy/cloudflared --tail=5
INF Registered tunnel connection connIndex=0 … location=jnb01 protocol=quic
INF Registered tunnel connection connIndex=1 … location=mba03 protocol=quic
INF Registered tunnel connection connIndex=2 … location=mba03 protocol=quic
INF Registered tunnel connection connIndex=3 … location=jnb04 protocol=quic

$ dig benardkipngeno.com +short
104.21.44.67
172.67.196.115

$ curl -sI https://benardkipngeno.com | grep -iE '^(HTTP|server|cf-ray)'
HTTP/1.1 200 OK
Server: cloudflare
CF-RAY: a52421b8b96915d5-JNB

$ curl -sI https://www.benardkipngeno.com | head -1
HTTP/1.1 200 OK
```

Three things worth reading in that output. The `location` fields are Cloudflare's edge sites: `jnb` is Johannesburg and `mba` is Mombasa, so the tunnel terminates a few hundred kilometres from the server, not in Europe. The two addresses from `dig` belong to Cloudflare, which is the whole point: a visitor never learns the home IP. And the `JNB` suffix on the ray ID says which edge served that particular request.

The replica table in the dashboard shows the home connection's public IP under *Origin IP*. That is the one address the tunnel exists to hide, so crop it out of any screenshot that goes in the repo.

## Lessons

> 💡 **What I learned:**
> - **Outbound beats inbound.** A tunnel the server opens needs no router rules, hides the home IP, and works behind CGNAT where port forwarding can't.
> - **The tunnel targets a Service, not an Ingress.** `portfolio.portfolio.svc.cluster.local` is a name every pod can resolve. Traefik is for traffic arriving at the node; the tunnel pod is already inside.
> - **Expose the app, not the admin tools.** Public hostnames are opt-in per Service. Everything else stays on `*.home.arpa`.
> - **A token is a credential.** Secret via `kubectl`, referenced by name, rotated from the Cloudflare dashboard if it ever leaks.

---

[◀ Previous: Buying a Domain and Setting Up Cloudflare](22-domain-and-cloudflare.md) · [🏠 Home](../../README.md) · [Next: Troubleshooting ▶](../troubleshooting.md)
