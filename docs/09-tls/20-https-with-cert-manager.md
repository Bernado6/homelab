# 20. HTTPS Everywhere With cert-manager and a Private CA

## What

Every site so far is plain HTTP and the browser says **Not Secure**. This step:

1. Installs **cert-manager**, the standard Kubernetes add-on that issues and renews TLS certificates.
2. Creates a **private certificate authority** (CA) inside the cluster.
3. Trusts that CA on the laptop, once.
4. Adds three lines to each Ingress, after which Traefik serves it on **443** with a valid certificate and redirects HTTP to HTTPS.

End state: `https://portfolio.home.arpa`, `https://grafana.home.arpa`, `https://prometheus.home.arpa`, `https://nginx.home.arpa`, all with a padlock.

## Why

| Reason | Detail |
|---|---|
| **No more "Not Secure"** | And no more browser warnings when a site uses features that require HTTPS (clipboard, service workers, some auth flows). |
| **Grafana and Prometheus are admin UIs** | Their passwords currently cross the Wi-Fi in clear text. |
| **CKAD** | TLS Secrets (`kubernetes.io/tls`), the `tls:` block on an Ingress, and `kubectl create secret tls` are all examinable. |
| **Prepares for a real domain** | When you buy one, you swap the issuer to Let's Encrypt and the Ingress annotations stay exactly the same. That's the point of cert-manager. |

### Why a private CA and not Let's Encrypt?

Let's Encrypt only issues for names it can verify on the public internet. `*.home.arpa` is reserved for home networks and never resolves publicly, so no public CA will ever sign it. The options:

| Option | Trust on the laptop | Certificates | Renewal |
|---|---|---|---|
| Self-signed per site | Click through a warning on every site, every browser | Manual `openssl` each time | Manual |
| **Private CA via cert-manager** ✅ | Trust **one** root once | Automatic per Ingress | Automatic |
| Let's Encrypt with a real domain + DNS-01 | Already trusted everywhere | Automatic | Automatic |

Private CA now, Let's Encrypt later. The second and third rows differ by one line in the Ingress annotation.

### The manual way first, so the automation makes sense

What cert-manager automates is this:

```bash
# generate a key and a self-signed cert for one host
openssl req -x509 -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 -nodes \
  -keyout tls.key -out tls.crt -days 365 -subj "/CN=nginx.home.arpa" \
  -addext "subjectAltName=DNS:nginx.home.arpa"

# put it in the cluster as a TLS Secret
kubectl create secret tls nginx-tls --cert=tls.crt --key=tls.key

# reference it from the Ingress
#   spec.tls[0].secretName: nginx-tls
```

That's the CKAD version: a Secret of type `kubernetes.io/tls` with keys `tls.crt` and `tls.key`, named in the Ingress. cert-manager watches the Ingress annotation, creates exactly that Secret, and renews it before expiry. Worth doing by hand once; delete the Secret afterwards so cert-manager can own it.

> ⚠️ Those two files land in whatever directory you ran `openssl` from. Run it in `$TMPDIR` or delete them afterwards. The repo's [`.gitignore`](../../.gitignore) excludes `*.key`, `*.crt` and `*.pem` so a slip here can't be committed, but the key is still on disk until you remove it.

## When

After the monitoring stack, so there's something worth protecting. Before any app with a login that you'd actually mind leaking.

## How

### 1. Install cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true \
  --set resources.requests.memory=64Mi --set resources.limits.memory=128Mi \
  --set webhook.resources.requests.memory=32Mi --set webhook.resources.limits.memory=64Mi \
  --set cainjector.resources.requests.memory=64Mi --set cainjector.resources.limits.memory=128Mi
kubectl -n cert-manager get pods
```

Three pods: `cert-manager` (the controller), `cainjector`, `webhook`. All `1/1 Running`. The whole thing costs under 100Mi in practice.

On chart versions older than 1.15 the CRD flag is `--set installCRDs=true` instead of `crds.enabled`.

### 2. Create the CA

[`apps/cert-manager/cluster-issuers.yaml`](../../apps/cert-manager/cluster-issuers.yaml) contains three objects that build on each other:

| Object | Kind | Role |
|---|---|---|
| `selfsigned-bootstrap` | ClusterIssuer | Can only self-sign. Used once. |
| `homelab-root-ca` | Certificate, `isCA: true` | The root. Ten-year validity, ECDSA key, stored in Secret `homelab-root-ca` in the `cert-manager` namespace. |
| `homelab-ca` | ClusterIssuer | Signs with that Secret. **This is the one every Ingress names.** |

```bash
kubectl apply -f apps/cert-manager/cluster-issuers.yaml
kubectl get clusterissuer
kubectl -n cert-manager get certificate
```

Wait for `homelab-root-ca` to show `READY True`, then `homelab-ca` flips to `READY True` as well. A `ClusterIssuer` works across all namespaces, which is why the portfolio in its own namespace can use it.

### 3. Trust the CA on the laptop

Export the root certificate and add it to the Mac's system keychain:

```bash
kubectl -n cert-manager get secret homelab-root-ca -o jsonpath='{.data.ca\.crt}' | base64 -d > ~/homelab-root-ca.crt
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain ~/homelab-root-ca.crt
```

Chrome, Edge and Safari read the system keychain. **Firefox keeps its own store**: Settings → Privacy & Security → Certificates → View Certificates → Authorities → Import, tick "Trust this CA to identify websites".

The `.crt` file is public; it's the certificate, not the key. The key never leaves the cluster.

### 4. The HTTP → HTTPS redirect

Traefik serves TLS Ingresses on 443 but keeps answering plain HTTP on 80 unless told otherwise. One Traefik `Middleware` handles it for every site:

```bash
kubectl apply -f apps/cert-manager/redirect-https-middleware.yaml
```

It's referenced from each Ingress as `default-redirect-https@kubernetescrd` (namespace-name@provider). On Traefik 2 the CRD apiVersion is `traefik.containo.us/v1alpha1`; the file notes this.

### 5. Switch each Ingress to TLS

Three lines per Ingress. In the `metadata.annotations`:

```yaml
cert-manager.io/cluster-issuer: homelab-ca
traefik.ingress.kubernetes.io/router.middlewares: default-redirect-https@kubernetescrd
```

And in `spec`:

```yaml
tls:
- hosts: [<host>]
  secretName: <name>-tls
```

**nginx** ([`apps/nginx/nginx.yaml`](../../apps/nginx/nginx.yaml), already updated):

```bash
kubectl apply -f apps/nginx/nginx.yaml
```

**Grafana and Prometheus** ([`apps/monitoring/values.yaml`](../../apps/monitoring/values.yaml), already updated with `annotations:` and `tls:` under both `ingress:` blocks, and Grafana's `root_url` switched to `https://`):

```bash
helm upgrade monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring -f apps/monitoring/values.yaml
```

**Portfolio**, in its own repo, `k8s/ingress.yaml`. The file already had the cert-manager lines commented out from step 17. Make the metadata and spec look like this:

```yaml
metadata:
  name: portfolio
  namespace: portfolio
  labels:
    app.kubernetes.io/name: portfolio
  annotations:
    cert-manager.io/cluster-issuer: homelab-ca
    traefik.ingress.kubernetes.io/router.middlewares: default-redirect-https@kubernetescrd
spec:
  ingressClassName: traefik
  tls:
    - hosts: [portfolio.home.arpa]
      secretName: portfolio-tls
  rules:
    - host: portfolio.home.arpa
      # ...unchanged
```

Remove the old `traefik.ingress.kubernetes.io/router.entrypoints: web` annotation; it pinned the router to port 80 only. Then:

```bash
cd ~/Desktop/MyProjects/mle/Data-Science-Personal-Website
kubectl apply -k k8s/
git commit -am "Ingress: TLS from the homelab CA" && git push
```

### 6. Watch the certificates get issued

```bash
kubectl get certificate -A
```

```text
NAMESPACE    NAME             READY   SECRET           AGE
default      nginx-tls        True    nginx-tls        30s
monitoring   grafana-tls      True    grafana-tls      30s
monitoring   prometheus-tls   True    prometheus-tls   30s
portfolio    portfolio-tls    True    portfolio-tls    30s
```

You never created those `Certificate` objects. cert-manager made one per annotated Ingress, got each signed by `homelab-ca`, and stored the result in the named Secret. If one is stuck at `False`:

```bash
kubectl -n <ns> describe certificate <name>      # Events at the bottom say why
kubectl -n <ns> get certificaterequest,order      # the intermediate objects
```

### 7. Verify

```bash
curl -sI http://portfolio.home.arpa | head -1          # HTTP/1.1 308 Permanent Redirect
curl -sI https://portfolio.home.arpa | head -1         # HTTP/2 200
openssl s_client -connect 192.168.100.22:443 -servername grafana.home.arpa </dev/null 2>/dev/null | openssl x509 -noout -issuer -dates
```

The last line should show `issuer=O=homelab, CN=Homelab Root CA` and a `notAfter` about 90 days out (cert-manager's default leaf lifetime; it renews at 60). Then open each site: padlock, no warning.

The `curl` calls work without flags because step 3 put the root in the system keychain and macOS `curl` honours it. If you get a certificate error from `curl` but not the browser, the shell's `curl` is a Homebrew build with its own CA bundle; `curl --cacert ~/homelab-root-ca.crt` proves the point.

### Later: the same Ingresses on a real domain

Buy a domain, point a wildcard at the home IP or use a tunnel, and add a `letsencrypt` ClusterIssuer with a DNS-01 solver for your DNS provider. Then change one annotation value per Ingress from `homelab-ca` to `letsencrypt`. Nothing else moves.

## Challenges and solutions

| Challenge | What was going on | Solution |
|---|---|---|
| `kubectl apply -k k8s/` on the portfolio failed with `strict decoding error: unknown field "spec.ingressClassName", unknown field "spec.rules", unknown field "spec.tls"`, while the Ingress reported `unchanged` | I had pasted the TLS snippet into `deployment.yaml` instead of `ingress.yaml`. The API server rejected a Deployment carrying Ingress fields, and the real Ingress was never edited. | `git checkout -- k8s/deployment.yaml` to restore the committed Deployment, then put the annotations and `tls:` block in `ingress.yaml`. **Reading the error:** the rejected field names tell you which *kind* you edited by mistake, and `unchanged` on the object you meant to edit confirms it. |

Everything else worked first time: cert-manager install, the three-object CA chain (`homelab-root-ca` Ready in about 20 seconds), the keychain import, the Traefik 3 `Middleware` on `traefik.io/v1alpha1`, and all four leaf certificates issued within a minute of their Ingresses being annotated.

My verification run:

```text
NAMESPACE      NAME              READY   SECRET            AGE
cert-manager   homelab-root-ca   True    homelab-root-ca   14m
default        nginx-tls         True    nginx-tls         10m
monitoring     grafana-tls       True    grafana-tls       9m53s
monitoring     prometheus-tls    True    prometheus-tls    9m53s
portfolio      portfolio-tls     True    portfolio-tls     25s

$ curl -sI http://portfolio.home.arpa  | head -1
HTTP/1.1 308 Permanent Redirect
$ curl -sI https://portfolio.home.arpa | head -1
HTTP/2 200
$ openssl s_client -connect 192.168.100.22:443 -servername grafana.home.arpa </dev/null 2>/dev/null | openssl x509 -noout -issuer -dates
issuer=O=homelab, CN=Homelab Root CA
notBefore=Oct 27 19:44:17 2026 GMT
notAfter=Jan 25 19:44:17 2027 GMT
```

Note the leaf is valid for 90 days. cert-manager will renew it around day 60 without anyone touching anything; `kubectl get certificate -A` a few months from now will show the same names with a fresh `AGE`.

## Lessons

> 💡 **What I learned:**
> - **A TLS Secret is just two keys, `tls.crt` and `tls.key`.** Make one by hand once with `kubectl create secret tls` and the Ingress `tls:` block stops being mysterious.
> - **cert-manager is an Ingress annotation away.** It watches for `cert-manager.io/cluster-issuer`, creates the Certificate, fills the Secret, and renews it. The Ingress never changes again.
> - **A private CA means trusting one thing, once.** Every site after that is a real padlock, on every browser that reads the system store.
> - **`ClusterIssuer` vs `Issuer`.** Cluster-wide vs one namespace. For a homelab with one CA, cluster-wide.
> - **Public names get public certs, private names get private CAs.** `.home.arpa` was the right choice in step 16 precisely because it can never collide with the public internet, and the cost is bringing your own CA.

---

[◀ Previous: Using the Metrics](../08-observability/19-using-the-metrics.md) · [🏠 Home](../../README.md) · [Next: Troubleshooting ▶](../troubleshooting.md)
