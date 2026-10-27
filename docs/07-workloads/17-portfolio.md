# 17. First Real Workload: The Portfolio Site

## What

My portfolio is a React app (Create React App) that has lived in a private S3 bucket behind CloudFront. This step runs the same site on the homelab: build it into a container image, publish the image to **GitHub Container Registry (GHCR)**, and deploy it to K3s with a Deployment, Service and host-based Ingress at `http://portfolio.home.arpa`.

The source, Dockerfile, and Kubernetes manifests live in the app's own repo: [Bernado6/Data-Science-Personal-Website](https://github.com/Bernado6/Data-Science-Personal-Website). This homelab repo documents the deployment; it does not duplicate the manifests.

## Why

| Reason | Detail |
|---|---|
| **A real app, same shape as steps 13 and 16** | Deployment + Service + Ingress again, but with my own image, two replicas, health probes, and a locked-down security context. |
| **Learn the image pipeline** | Build on CI, push to a registry, pull on the cluster. That's how every production cluster gets its software. |
| **CKAD coverage** | Multi-container-free but hits: probes, resources, security contexts, namespaces, rolling updates, Kustomize. |
| **Cheaper than CloudFront for a demo** | It's a static site. Serving it from a mini PC on the LAN costs nothing. |

### Why a registry at all?

The cluster **pulls** images. It cannot see Docker on the laptop. Three ways to bridge that:

| Option | How | When it fits |
|---|---|---|
| **Registry (GHCR)** ✅ | CI builds and pushes; cluster pulls by name | Any time the code is on GitHub. Free for public images. |
| Import into containerd | `docker save … \| ssh server 'sudo k3s ctr images import -'` | Quick one-off test, no GitHub involved |
| Run your own registry | A `registry:2` pod on the cluster | Later, once you're building images for several apps |

### The architecture trap

The laptop is an **Apple Silicon Mac (arm64)**. The server is an **Intel i5 (amd64)**. An image built with a plain `docker build` on the Mac is arm64 and will crash on the server with `exec /usr/sbin/nginx: exec format error`.

Two ways out:

- Let **GitHub Actions** build it. The runner is amd64, and the workflow pins `platforms: linux/amd64`. This is the path this step uses.
- Build locally with `docker build --platform linux/amd64 …`. Slower on a Mac because it emulates, but works for the import option above.

### Where do the manifests live?

| Pattern | Used for | Example |
|---|---|---|
| **Manifests next to the app's source** | Real apps you maintain | `k8s/` in the portfolio repo |
| Manifests in the homelab repo | Throwaway tests and shared infra | `apps/hello`, `apps/nginx` |

The app repo owns its manifests because the Dockerfile, the port, the health endpoint and the manifests change together. The homelab repo owns the cluster-wide things.

## When

After [step 16](../06-working-with-the-cluster/16-nginx-ingress.md). You need `kubectl` on the laptop, a fixed IP, and the host-based Ingress pattern.

## How

### Part A: publish the image (portfolio repo)

#### 1. Commit the container files and push

In the portfolio repo, check what's untracked:

```bash
cd ~/Desktop/MyProjects/mle/Data-Science-Personal-Website
git status --short
```

Two stray files, `portfolio@2.0.0` and `react-scripts`, were created by a mistyped `npm` command. Delete them, don't commit them:

```bash
rm -f 'portfolio@2.0.0' react-scripts
```

Set the Ingress host to the `.home.arpa` name used across this guide (the default in the repo is `portfolio.homelab.local`, and `.local` misbehaves on macOS):

```bash
sed -i '' 's/portfolio\.homelab\.local/portfolio.home.arpa/g' k8s/ingress.yaml
```

Then commit everything and push:

```bash
git add -A
git commit -m "Container image, GHCR workflow and K3s manifests"
git push origin master
```

#### 2. Watch the build

Pushing to `master` triggers `.github/workflows/docker-image.yml`. On GitHub: **Actions → Build and publish container image**. About three minutes. It publishes:

- `ghcr.io/bernado6/portfolio:latest`
- `ghcr.io/bernado6/portfolio:<short-sha>`

#### 3. Make the package public (once)

GHCR packages are private by default, and a private image needs a pull secret in the cluster. The site is public anyway, so make the image public:

**GitHub → your profile → Packages → portfolio → Package settings → Change visibility → Public.**

Then prove the cluster will be able to pull it, from the laptop with no login:

```bash
docker manifest inspect ghcr.io/bernado6/portfolio:latest | grep architecture
```

```text
            "architecture": "amd64",
            "architecture": "unknown",
```

`amd64` is the image the server will run. The `unknown` entry is not a broken build: it's the **provenance attestation** that `docker/build-push-action` attaches alongside the image (a signed record of how it was built). Kubernetes ignores it and pulls the amd64 one.

If `docker manifest` complains, `curl -sI https://ghcr.io/v2/bernado6/portfolio/manifests/latest` returning anything other than `401` is enough.

### Part B: deploy (from the laptop)

#### 4. Apply with Kustomize

```bash
cd ~/Desktop/MyProjects/mle/Data-Science-Personal-Website
kubectl apply -k k8s/
kubectl -n portfolio rollout status deployment/portfolio
kubectl -n portfolio get pods,svc,ingress
```

`apply -k` reads `k8s/kustomization.yaml`, which lists the four files and sets the image tag in one place. Everything lands in its own **`portfolio` namespace**, so `-n portfolio` is needed on every command, or you'll get "not found" from the `default` namespace.

My run:

```text
namespace/portfolio created
service/portfolio created
deployment.apps/portfolio created
ingress.networking.k8s.io/portfolio created

Waiting for deployment "portfolio" rollout to finish: 0 of 2 updated replicas are available...
Waiting for deployment "portfolio" rollout to finish: 1 of 2 updated replicas are available...
deployment "portfolio" successfully rolled out

NAME                             READY   STATUS    RESTARTS   AGE
pod/portfolio-5ddbc576ff-79p8q   1/1     Running   0          93s
pod/portfolio-5ddbc576ff-z64sq   1/1     Running   0          93s

NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/portfolio   ClusterIP   10.43.177.188   <none>        80/TCP    93s

NAME                                  CLASS     HOSTS                 ADDRESS          PORTS   AGE
ingress.networking.k8s.io/portfolio   traefik   portfolio.home.arpa   192.168.100.22   80      93s
```

The "0 of 2, then 1 of 2" lines are the readiness probes doing their job: a replica only counts as available once `/healthz` answers.

#### 5. Test with a Host header, then with a name

```bash
curl -s -o /dev/null -w '%{http_code}\n' -H "Host: portfolio.home.arpa" http://192.168.100.22/healthz
curl -s -H "Host: portfolio.home.arpa" http://192.168.100.22 | head -3
```

`200` from `/healthz` and HTML from `/` means routing works. Then:

```bash
sudo sh -c 'echo "192.168.100.22  portfolio.home.arpa" >> /etc/hosts'
```

Open **`http://portfolio.home.arpa`**.

#### 6. Roll out a new version

`latest` plus `imagePullPolicy: IfNotPresent` means the node will **never re-pull** once it has a `latest`. Pushing new code and re-applying does nothing. For a real rollout, pin the tag to the short SHA the workflow printed:

```bash
cd k8s
kustomize edit set image ghcr.io/bernado6/portfolio=ghcr.io/bernado6/portfolio:<short-sha>
cd ..
kubectl apply -k k8s/
kubectl -n portfolio rollout status deployment/portfolio
```

The Deployment is `maxUnavailable: 0, maxSurge: 1`, so a new pod comes up and passes its readiness probe before an old one goes away. Watch it with `kubectl -n portfolio get pods -w`.

## What's in the manifests worth understanding

| Setting | Where | Why it's there |
|---|---|---|
| `pod-security.kubernetes.io/enforce: restricted` | `namespace.yaml` | The namespace **refuses** pods that run as root, keep capabilities, or lack a seccomp profile. The Deployment is written to pass. Try removing `runAsNonRoot` and watch the apply get rejected. |
| `nginxinc/nginx-unprivileged`, port **8080** | Dockerfile | Non-root nginx can't bind port 80 (needs root or a capability). 8080 avoids the problem. |
| `readOnlyRootFilesystem: true` + `emptyDir` for `/tmp` and `/var/cache/nginx` | `deployment.yaml` | The container can't write anywhere except the two scratch volumes. |
| Readiness and liveness probes on `/healthz` | `deployment.yaml` | Readiness gates traffic; liveness restarts a hung container. Both CKAD staples. |
| `requests` 10m/32Mi, `limits` 200m/64Mi | `deployment.yaml` | Two replicas cost about 130MB at most. Fine on 8GB. |
| `images:` with `newTag` | `kustomization.yaml` | One place to bump the version. |

## Challenges and solutions

This one went through first time: push, Actions build, package made public, `kubectl apply -k`, hosts entry, site up at `http://portfolio.home.arpa`.

| Challenge | What was going on | Solution |
|---|---|---|
| `kubectl get pods` right after the rollout showed only `hello` and `nginx`, no portfolio | Looking in the `default` namespace. The portfolio pods are in `portfolio`. | `kubectl get pods -n portfolio`. This is the namespace lesson happening live, 20 seconds after a successful rollout. |
| `docker manifest inspect` lists `"architecture": "unknown"` next to `amd64` | The build attaches a provenance attestation as a second manifest entry. | Nothing to fix. Kubernetes pulls the `amd64` one. |
| `zsh: command not found: kubetctl` | Typo. | The `k` alias from step 15 exists for exactly this reason. |

And the things that *would* have bitten, avoided by checking first:

| Would-be challenge | Why it didn't happen |
|---|---|
| `exec format error` on the pods | Checked `uname -m` on both machines before building. The Mac is arm64, the server amd64, so the image was built on GitHub Actions with `platforms: linux/amd64` instead of locally. |
| `ImagePullBackOff` with a 401 | Made the GHCR package public before applying. |
| Name not resolving on the laptop | Host set to `portfolio.home.arpa` in `k8s/ingress.yaml` before the push, then one `/etc/hosts` line, same pattern as step 16. |
| Junk files committed | `git status` before `git add -A` caught `portfolio@2.0.0` and `react-scripts`, left over from a mistyped npm command. |

## Lessons

> 💡 **What I learned:**
> - **Check the CPU architecture before you build.** `uname -m` on the laptop, `uname -m` on the server. If they differ, build on CI or pass `--platform`.
> - **The cluster pulls, you don't push.** A registry is the hand-off point. For anything on GitHub, GHCR plus Actions is free and zero-maintenance.
> - **`latest` + `IfNotPresent` is a silent no-op on redeploy.** Pin tags to a SHA and bump them. The tag change is what triggers the rollout.
> - **Namespaces are easy to forget.** "Not found" after a successful apply usually means you're looking in `default`.
> - **A restricted namespace is a great teacher.** It rejects anything sloppy, and the error message tells you exactly which field.
> - **Keep manifests with the app they describe.** The Dockerfile, the port, the health endpoint and the Deployment change together.

---

[◀ Previous: Exposing an App With an Ingress](../06-working-with-the-cluster/16-nginx-ingress.md) · [🏠 Home](../../README.md) · [Next: Monitoring: Prometheus and Grafana ▶](../08-observability/18-prometheus-grafana.md)
