# 20. HTTPS Everywhere With cert-manager and a Private CA

Every screenshot in this guide so far has a "Not Secure" badge in the corner. This step removes it, and it does so by building a small piece of real infrastructure: a certificate authority of your own, run by the cluster, trusted by your devices. Most people's first contact with TLS is a browser warning and a shrug. This page explains what the warning actually means, why a padlock is a statement about trust rather than encryption, and how the whole thing can be automated so you never think about it again.

## What

Every site so far is plain HTTP and the browser says **Not Secure**. This step:

1. Installs **cert-manager**, the standard Kubernetes add-on that issues and renews TLS certificates.
2. Creates a **private certificate authority** (CA) inside the cluster.
3. Trusts that CA on the laptop, once.
4. Adds three lines to each Ingress, after which Traefik serves it on **443** with a valid certificate and redirects HTTP to HTTPS.

End state: `https://portfolio.home.arpa`, `https://grafana.home.arpa`, `https://prometheus.home.arpa`, `https://nginx.home.arpa`, all with a padlock.

## Background: what TLS actually does

HTTP sends everything as readable text. Anyone on the path between the browser and the server, which on your home network means anyone on the Wi-Fi, can read a Grafana password as it goes past. **TLS** (Transport Layer Security, the successor to SSL) wraps the connection so that:

| Property | Meaning | How |
|---|---|---|
| **Confidentiality** | Nobody in between can read it | Encryption with a key only the two ends know |
| **Integrity** | Nobody in between can alter it | Every message is checked against a cryptographic signature |
| **Authentication** | The browser is talking to the real `grafana.home.arpa`, not an impostor | **Certificates**, which is where this whole step lives |

The first two are mechanical and the browser does them for free once a connection exists. The third is the hard one, because encrypting a conversation with an impostor is worse than useless. "Not Secure" and every certificate warning you've ever seen are about authentication, not encryption.

### Keys and certificates

Everything rests on **asymmetric keys**: a pair where anything encrypted with one half can only be decrypted with the other. One half, the **private key**, never leaves the server. The other, the **public key**, is handed to anyone who asks.

A **certificate** is a public key plus a claim about who it belongs to, in a standard format (X.509):

```text
Certificate
├── Subject:      CN=grafana.home.arpa              ← "this key belongs to this name"
├── SAN:          DNS:grafana.home.arpa              ← Subject Alternative Names: the names it's valid for
├── Public key:   (ECDSA P-256)
├── Validity:     Oct 27 2026 → Jan 25 2027          ← notBefore / notAfter
├── Issuer:       O=homelab, CN=Homelab Root CA      ← who vouches for the claim
└── Signature:    (made with the ISSUER's private key)
```

The signature is the point. Anyone can make a certificate claiming to be `grafana.home.arpa`. What makes one believable is that it's **signed by someone the browser already trusts**.

### The chain of trust

Browsers and operating systems ship with a **trust store**: a few hundred **root certificates** belonging to certificate authorities (CAs) such as Let's Encrypt, DigiCert and Google. When the browser receives a site's certificate, it checks:

```text
site certificate ("leaf")
   signed by ──► an issuer certificate
                    signed by ──► a ROOT certificate that is in my trust store?  ✅ padlock
                                                                                 ❌ warning
```

The browser also checks that the name it asked for appears in the certificate's **SAN** list, and that today's date is inside the validity window. Fail any check and you get the warning page.

So "Not Secure" on plain HTTP means *no certificate at all*. A full-page warning on HTTPS means *there is a certificate, but I don't trust who signed it*. Those are different situations, and this step moves every site from the first to a padlock, skipping the second, by making your devices trust a root of your own.

### What the padlock proves, and what it doesn't

| The padlock means | The padlock does not mean |
|---|---|
| The connection is encrypted | The site is honest or safe |
| The name in the address bar matches the certificate | The server is well configured |
| A CA your device trusts signed it | The CA is one *you* chose to trust (your OS chose most of them) |

That last row is what makes a private CA legitimate: you're adding one more root to a list that already has hundreds you never reviewed, and this one you control.

### Why `.home.arpa` can't get a public certificate

Public CAs like Let's Encrypt only sign a certificate after proving the applicant controls the name, by serving a challenge file over HTTP or publishing a DNS record (the **ACME** protocol, with HTTP-01 and DNS-01 challenges). That requires the name to resolve on the public internet. `*.home.arpa` never does, by design, so no public CA will ever issue for it. The options:

| Option | Trust on the laptop | Certificates | Renewal |
|---|---|---|---|
| Self-signed per site | Click through a warning on every site, every browser | Manual `openssl` each time | Manual |
| **Private CA via cert-manager** ✅ | Trust **one** root once | Automatic per Ingress | Automatic |
| Let's Encrypt with a real domain + DNS-01 | Already trusted everywhere | Automatic | Automatic |

Private CA now, Let's Encrypt later. The second and third rows differ by one line in the Ingress annotation.

## Background: how Kubernetes holds a certificate

Kubernetes has a Secret type for exactly this: `kubernetes.io/tls`, with two keys, `tls.crt` (the certificate, public) and `tls.key` (the private key). An Ingress refers to it by name in its `tls:` block, and the Ingress controller (Traefik) reads it to serve HTTPS for that host.

```text
Ingress "grafana"
  spec.tls[0].hosts: [grafana.home.arpa]
  spec.tls[0].secretName: grafana-tls  ──────► Secret "grafana-tls" (type kubernetes.io/tls)
                                                 ├── tls.crt
                                                 └── tls.key
                                               Traefik reads it, serves https://grafana.home.arpa
```

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

Reading that `openssl` line: `req -x509` makes a self-signed certificate rather than a signing request; `-newkey ec -pkeyopt ec_paramgen_curve:prime256v1` generates an elliptic-curve key (small and fast, what cert-manager uses too); `-nodes` means no passphrase on the key, which a server needs; `-days 365` is the validity; `-subj` is the subject; `-addext subjectAltName` is the SAN, which modern browsers require and which the subject alone no longer satisfies.

## Background: what cert-manager is

cert-manager is a controller, like the ones that run Deployments, but for certificates. It adds a few object kinds to the cluster and watches them:

| Object | What it is | Who creates it here |
|---|---|---|
| **Issuer** / **ClusterIssuer** | *How* to get certificates signed: self-sign, use a CA key in a Secret, or talk to an ACME server. `Issuer` works in one namespace, `ClusterIssuer` everywhere. | You, once, in `cluster-issuers.yaml` |
| **Certificate** | *A request*: "I want a cert for these names, from this issuer, stored in this Secret, renewed before it expires" | cert-manager, one per annotated Ingress. Or you, for the root CA |
| **CertificateRequest** | One attempt to fulfil a Certificate | cert-manager, internally |
| **Order** / **Challenge** | ACME bookkeeping | Only when the issuer is Let's Encrypt. Not used with a private CA |

The piece that makes it hands-off is the **ingress-shim**: cert-manager watches every Ingress for the annotation `cert-manager.io/cluster-issuer`. When it sees one, it reads the Ingress's `tls:` block and creates a `Certificate` for those hosts into that Secret name. From there:

```text
Ingress annotation ──► Certificate (auto-created) ──► CertificateRequest ──► ClusterIssuer signs it
                                                                               │
Traefik ◄── Secret <name>-tls (tls.crt + tls.key) ◄────────────────────────────┘
          cert-manager watches expiry and repeats the cycle at two-thirds of the lifetime
```

Three pods run it: `cert-manager` (the controller), `webhook` (validates the objects when you apply them), and `cainjector` (keeps the webhook's own certificate current).

### Bootstrapping a CA from nothing

A CA is just a certificate with `isCA: true` plus its private key, used to sign other certificates. cert-manager can create one for itself with a three-object chain:

```text
selfsigned-bootstrap   (ClusterIssuer, selfSigned)   ──signs──► homelab-root-ca (Certificate, isCA, 10 years)
                                                                    │ stored in Secret "homelab-root-ca"
                                                                    ▼
                                                         homelab-ca (ClusterIssuer, ca: that Secret)
                                                                    │
                                                                    └──signs──► every leaf certificate
```

The bootstrap issuer is used exactly once. The root's private key lives in the cluster and never leaves. Its public certificate is what you install on the laptop.

## Background: HTTP to HTTPS, and Traefik's two doors

Traefik listens on two **entrypoints**: `web` on port 80 and `websecure` on 443. An Ingress with a `tls:` block is served on `websecure`. Plain HTTP on `web` keeps working unless something redirects it, and a site that answers on both is a site where a typed URL still goes unencrypted.

Traefik's **Middleware** CRD is a reusable request-processing step. One `redirectScheme` middleware, applied to every Ingress through an annotation, turns each HTTP request into a **308 Permanent Redirect** to the HTTPS URL. 308 rather than 301 because 308 guarantees the method and body are preserved (a POST stays a POST), which matters for forms and APIs and costs nothing otherwise.

The annotation value `default-redirect-https@kubernetescrd` is Traefik's naming: `<namespace>-<name>@<provider>`, where `kubernetescrd` means "a Middleware defined as a Kubernetes custom resource".

Once on HTTPS, Traefik also negotiates **HTTP/2**, which is why the verification shows `HTTP/2 200` rather than `HTTP/1.1`. HTTP/2 is only ever used over TLS in browsers, so the protocol upgrade comes free with the certificate.

## Why

| Reason | Detail |
|---|---|
| **No more "Not Secure"** | And no more browser warnings when a site uses features that require HTTPS (clipboard, service workers, some auth flows). |
| **Grafana and Prometheus are admin UIs** | Their passwords currently cross the Wi-Fi in clear text. |
| **CKAD** | TLS Secrets (`kubernetes.io/tls`), the `tls:` block on an Ingress, and `kubectl create secret tls` are all examinable. |
| **Prepares for a real domain** | When you buy one, you swap the issuer to Let's Encrypt and the Ingress annotations stay exactly the same. That's the point of cert-manager. |

### Why a private CA and not Let's Encrypt?

Let's Encrypt only issues for names it can verify on the public internet. `*.home.arpa` is reserved for home networks and never resolves publicly, so no public CA will ever sign it. The comparison table in the Background section above is the decision; the short version is that a private CA costs one trust-store import and nothing else, and the Ingress side is identical to the Let's Encrypt setup you'd use with a real domain.

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

Jetstack is the company that created cert-manager and still hosts its chart. `--create-namespace` makes `cert-manager` on the way in. `crds.enabled=true` installs the custom resource definitions (the new object kinds above) as part of the release; without it there's nothing for `kubectl apply` of a ClusterIssuer to be checked against. The `--set` lines are inline values, the same idea as a values file, used here because there are only six of them.

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

`homelab-ca` can't be Ready until the Secret it signs with exists, which is why there's a few seconds' gap between the two. If `homelab-ca` stays `False`, `kubectl describe clusterissuer homelab-ca` names the missing Secret.

### 3. Trust the CA on the laptop

Export the root certificate and add it to the Mac's system keychain:

```bash
kubectl -n cert-manager get secret homelab-root-ca -o jsonpath='{.data.ca\.crt}' | base64 -d > ~/homelab-root-ca.crt
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain ~/homelab-root-ca.crt
```

The first line pulls the `ca.crt` key out of the Secret (the backslash escapes the dot in the jsonpath) and decodes it into a file. The second uses macOS's `security` tool: `-d` for the admin domain, `-r trustRoot` meaning "trust this as a root", `-k` naming the system keychain so every user and app on the Mac sees it.

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

The first annotation is what the ingress-shim watches for. The second attaches the redirect. The `tls:` block names the hosts and the Secret; cert-manager fills the Secret, Traefik reads it. The Secret doesn't need to exist when you apply the Ingress; Traefik serves a temporary default certificate for a few seconds until it does.

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

The usual causes are a typo in the issuer name, or a host in `tls.hosts` that isn't in the Ingress `rules`. The Events spell out which.

### 7. Verify

```bash
curl -sI http://portfolio.home.arpa | head -1          # HTTP/1.1 308 Permanent Redirect
curl -sI https://portfolio.home.arpa | head -1         # HTTP/2 200
openssl s_client -connect 192.168.100.22:443 -servername grafana.home.arpa </dev/null 2>/dev/null | openssl x509 -noout -issuer -dates
```

The first proves the redirect middleware. The second proves TLS works and that HTTP/2 came with it. The third opens a raw TLS connection to the server, asks for `grafana.home.arpa` by name (`-servername` is the SNI field, which is how one address serves many certificates), and prints who issued the certificate and when it expires.

The last line should show `issuer=O=homelab, CN=Homelab Root CA` and a `notAfter` about 90 days out (cert-manager's default leaf lifetime; it renews at 60). Then open each site: padlock, no warning.

The `curl` calls work without flags because step 3 put the root in the system keychain and macOS `curl` honours it. If you get a certificate error from `curl` but not the browser, the shell's `curl` is a Homebrew build with its own CA bundle; `curl --cacert ~/homelab-root-ca.crt` proves the point.

### Later: the same Ingresses on a real domain

Buy a domain, point a wildcard at the home IP or use a tunnel, and add a `letsencrypt` ClusterIssuer with a DNS-01 solver for your DNS provider. Then change one annotation value per Ingress from `homelab-ca` to `letsencrypt`. Nothing else moves.

With DNS-01, cert-manager proves control of a name by creating a temporary TXT record in the domain's zone through the provider's API, which works even for names that only resolve at home. That's the path [step 22](../11-public-access/22-domain-and-cloudflare.md) sets up the domain for.

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

Traps that didn't bite me but are the common ones:

| Trap | Symptom | Fix |
|---|---|---|
| Browser still warns after the keychain import | Firefox, which ignores the system store | Import into Firefox separately, section 3 |
| Phone warns on every site | Phones have their own trust store | Install the `.crt` on the phone, covered in [step 21](../10-dns/21-local-dns-adguard.md) section 9 |
| `curl` fails, browser fine | Homebrew `curl` with its own CA bundle | `curl --cacert ~/homelab-root-ca.crt`, or `/usr/bin/curl` |
| Certificate `READY False` | Issuer name typo, or a `tls.hosts` entry not in `rules` | `kubectl describe certificate`, read the Events |
| Site works on 443 but 80 still serves plain HTTP | Middleware annotation missing on that Ingress | Add the `router.middlewares` annotation |

## Lessons

> 💡 **What I learned:**
> - **A TLS Secret is just two keys, `tls.crt` and `tls.key`.** Make one by hand once with `kubectl create secret tls` and the Ingress `tls:` block stops being mysterious.
> - **cert-manager is an Ingress annotation away.** It watches for `cert-manager.io/cluster-issuer`, creates the Certificate, fills the Secret, and renews it. The Ingress never changes again.
> - **A private CA means trusting one thing, once.** Every site after that is a real padlock, on every browser that reads the system store.
> - **`ClusterIssuer` vs `Issuer`.** Cluster-wide vs one namespace. For a homelab with one CA, cluster-wide.
> - **Public names get public certs, private names get private CAs.** `.home.arpa` was the right choice in step 16 precisely because it can never collide with the public internet, and the cost is bringing your own CA.
> - **The warning is about trust, not encryption.** A self-signed certificate encrypts just as well as a public one. What the browser objects to is not knowing who vouched for the name.
> - **Redirect with 308, not 301.** Same effect for a page load, and it doesn't quietly turn a POST into a GET.

## Glossary

| Term | Meaning |
|---|---|
| TLS | The protocol that encrypts and authenticates a connection; HTTPS is HTTP over TLS |
| Public / private key | A pair where one half encrypts or verifies what the other half decrypts or signs; the private half never leaves the server |
| Certificate (X.509) | A public key plus a name claim, signed by an issuer |
| SAN | Subject Alternative Name, the list of DNS names a certificate is valid for; browsers require it |
| CA | Certificate authority: an entity whose signature browsers trust |
| Root / leaf | The self-signed certificate at the top of a chain / the per-site certificate at the bottom |
| Chain of trust | Leaf, signed by an issuer, signed by a root that's in the trust store |
| Trust store | The OS or browser's list of root certificates it accepts; the system keychain on macOS |
| Self-signed | A certificate signed with its own key; trusted by nobody until you add it |
| ACME / HTTP-01 / DNS-01 | The protocol public CAs use to prove you control a name, and its two challenge types |
| `kubernetes.io/tls` | The Secret type holding `tls.crt` and `tls.key` |
| Issuer / ClusterIssuer | cert-manager's "how to sign" object, per namespace / cluster-wide |
| Certificate / CertificateRequest | cert-manager's "I want this signed" object / one attempt to fulfil it |
| ingress-shim | The part of cert-manager that turns an Ingress annotation into a Certificate |
| Entrypoint | Traefik's listening port: `web` is 80, `websecure` is 443 |
| Middleware | A reusable Traefik request step, defined as a Kubernetes custom resource |
| `@kubernetescrd` | Traefik's suffix for middlewares defined as Kubernetes objects, named `<namespace>-<name>` |
| 308 | Permanent redirect that preserves the HTTP method, unlike 301 |
| SNI | The name a client asks for during the TLS handshake, so one address can serve many certificates |
| HTTP/2 | The faster HTTP version browsers use only over TLS |

---

[◀ Previous: Using the Metrics](../08-observability/19-using-the-metrics.md) · [🏠 Home](../../README.md) · [Next: Local DNS for the Whole House ▶](../10-dns/21-local-dns-adguard.md)
