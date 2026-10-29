# 17. First Real Workload: The Portfolio Site

Everything before this used someone else's image. This step builds my own, which drags in a whole second half of the Kubernetes story: how images are built, where they live, how a cluster fetches them, and why a build that works on the laptop can crash on the server. It also introduces namespaces, Pod Security, probes and rolling updates properly. It went through first time, mostly because I read about the traps before hitting them, and this page tries to pass that on.

## What

My portfolio is a React app (Create React App) that has lived in a private S3 bucket behind CloudFront. This step runs the same site on the homelab: build it into a container image, publish the image to **GitHub Container Registry (GHCR)**, and deploy it to K3s with a Deployment, Service and host-based Ingress at `http://portfolio.home.arpa`.

The source, Dockerfile, and Kubernetes manifests live in the app's own repo: [Bernado6/Data-Science-Personal-Website](https://github.com/Bernado6/Data-Science-Personal-Website). This homelab repo documents the deployment; it does not duplicate the manifests.

After this step:

- A `git push` to the portfolio repo produces an amd64 image on GHCR, built by GitHub Actions.
- The cluster runs two replicas of it in a `portfolio` namespace that enforces the `restricted` Pod Security level.
- `http://portfolio.home.arpa` serves the site from the mini PC on the LAN.

## Background: container images and registries

### What an image is

A container image is a **filesystem snapshot plus some metadata** (which command to run, which port the app listens on, which user to run as). It is not a virtual machine: there is no kernel inside. When the container starts, the node's own Linux kernel runs the process from that snapshot, isolated from everything else on the machine.

The snapshot is stored as a stack of **layers**. Each instruction in a Dockerfile that changes files (`COPY`, `RUN`) produces one layer, and each layer is a compressed tar archive identified by the **SHA-256 digest** of its contents. Layers are shared: if the portfolio image and the nginx image both start from the same base, the node downloads that base once.

In outline, a static-site image like this one looks like:

```text
 ghcr.io/bernado6/portfolio:<tag>
 ┌───────────────────────────────────────────────┐
 │ top layer:  the built site, copied into the   │  ← changes every push
 │             web root                          │
 │ middle:     any config the Dockerfile adds    │
 │ base:       nginx-unprivileged's own layers   │  ← shared with any other image on that base
 └───────────────────────────────────────────────┘
   + config: which user to run as, which port, which command
```

### Names, tags, digests

An image reference has four parts:

| Part of `ghcr.io/bernado6/portfolio:latest` | Meaning | Default if omitted |
|---|---|---|
| `ghcr.io` | The **registry** host | `docker.io` (Docker Hub) |
| `bernado6/portfolio` | The **repository**: namespace and name. Lowercase only, which is why it is `bernado6` and not `Bernado6`. | `library/` on Docker Hub for official images like `nginx` |
| `:latest` | The **tag**: a human-readable, movable label | `latest` |
| `@sha256:…` (alternative to a tag) | The **digest**: content hash, immovable | none |

A tag is a pointer. `latest` today and `latest` tomorrow can be two different images. The short-SHA tag the workflow also pushes (`:<short-sha>`) points at exactly one build forever, which is why section 6 pins to it.

### Manifests and multi-arch images

When the node asks the registry for `portfolio:latest`, the registry answers with a **manifest**: a small JSON document listing the layers and the config, each by digest. The node then fetches only the layers it doesn't already have.

Some tags point at a **manifest list** (also called an image index) instead: a list of manifests, one per CPU architecture and OS. The client picks the entry that matches the machine it is running on. That is how `docker pull nginx` gives an arm64 image on a Mac and an amd64 one on the server from the same tag. It is also what `docker manifest inspect` prints, and why it can show more than one `architecture` line for one tag.

### Registries, and how the cluster pulls

A registry is an HTTP server that stores manifests and layers and speaks the OCI distribution API. The node's container runtime (**containerd** on K3s; there is no Docker on the server) talks to it directly. The laptop is never involved in a pull. This is the single most important fact in the step: **the cluster pulls, you don't push to the cluster**. You push to a registry, and the cluster pulls from there.

| Registry | Who runs it | Notes |
|---|---|---|
| Docker Hub (`docker.io`) | Docker | The default; rate-limits anonymous pulls |
| GHCR (`ghcr.io`) ✅ | GitHub | Free for public images, tied to the repo and Actions, no extra account |
| A `registry:2` pod | You | Useful later; needs TLS or an insecure-registry setting on the node |

## Background: CPU architectures, and "exec format error"

A compiled binary is machine code for one instruction set. The two that matter here:

| Architecture | Also called | Where |
|---|---|---|
| `arm64` | `aarch64`, Apple Silicon | The MacBook |
| `amd64` | `x86_64` | The OptiPlex's Intel i5, and GitHub's standard runners |

A container image contains compiled binaries (nginx, in this case), so an image is for one architecture. `docker build` on the Mac, with no flags, builds for the machine it runs on: arm64. Ship that image to the amd64 server and the kernel is asked to run arm64 machine code. It can't, and it says so in the only words it has:

```text
exec /usr/sbin/nginx: exec format error
```

The pod shows `CrashLoopBackOff`, and the message is in `kubectl logs`. It looks like a broken image; it is a mismatched one.

Three ways to end up with an amd64 image from an arm64 laptop:

| Method | How | Cost |
|---|---|---|
| Build on CI ✅ | GitHub Actions runners are amd64, and the workflow pins `platforms: linux/amd64` | Three minutes per push, no laptop CPU involved |
| Cross-build locally | `docker build --platform linux/amd64 …` | Docker runs the build under QEMU emulation; several times slower |
| Multi-arch build | `docker buildx build --platform linux/amd64,linux/arm64` | Produces a manifest list that works on both; twice the build time |

`uname -m` on each machine tells you which you have before you build anything. That one command is what kept this step from failing.

## Background: GitHub Actions, GHCR and package visibility

### The workflow

`.github/workflows/docker-image.yml` in the portfolio repo describes a job that GitHub runs on every push to `master`. The usual shape of such a workflow: check out the code, log in to GHCR with the token GitHub provides to each workflow run, then call `docker/build-push-action` to build the Dockerfile for `linux/amd64` and push it with two tags, `latest` and the commit's short SHA. The runner is a fresh amd64 virtual machine each time, which is exactly why it solves the architecture problem.

### Visibility

A GHCR image is a **package** attached to your GitHub account or organisation. New packages are **private** by default, even when the source repo is public. A private image needs credentials to pull, which on Kubernetes means creating a `docker-registry` Secret and referencing it as an `imagePullSecrets` entry on the pod. Without that, the node gets a `401` and the pod sits in `ImagePullBackOff`.

| Visibility | Cluster needs | Fits when |
|---|---|---|
| Private | An `imagePullSecrets` Secret with a GitHub token | The code or the built artefact is sensitive |
| Public ✅ | Nothing | The artefact is already public, as a portfolio site is |

Making the package public is a one-time setting. The test for "is it public?" is whether an anonymous request for the manifest succeeds, which is what section 3 checks.

### Provenance attestations: the "architecture: unknown" entry

Recent versions of `docker/build-push-action` attach a **provenance attestation** to each push: a signed record of how the image was built (which workflow, which commit, which base image). It is stored in the registry as an extra entry in the manifest list, with `architecture: unknown` and `os: unknown`, because it isn't runnable on anything. Tools that list manifest entries show it; container runtimes skip it and pick the entry matching their platform. It is not a broken second build, and nothing needs fixing.

## Background: Kustomize and the `images:` transform

### Why not just `kubectl apply -f k8s/`?

`kubectl apply -f k8s/` would apply every YAML file in the folder, in alphabetical order, with no way to change a value without editing the file. **Kustomize** is a layer on top: a `kustomization.yaml` lists the files to include and describes **transformations** to apply before sending them to the cluster. It is built into `kubectl` as `apply -k`, so there is nothing to install.

```yaml
# k8s/kustomization.yaml, in outline (four resource files plus one image rule)
resources:
- namespace.yaml
- deployment.yaml
- service.yaml
- ingress.yaml
images:
- name: ghcr.io/bernado6/portfolio
  newTag: <short-sha>
```

### The `images:` transform

The `images:` block says: "wherever any listed resource uses an image called `ghcr.io/bernado6/portfolio`, replace the tag with `newTag`". The Deployment file keeps a plain reference, and the version lives in exactly one place. `kustomize edit set image …` (section 6) rewrites that block for you, so a release is one command plus an `apply`.

Other transforms you'll meet later: `namespace:` to put everything in one namespace, `commonLabels:` to label every object, and `patches:` to tweak fields per environment (a dev overlay with one replica, a prod overlay with three). CKAD expects you to recognise `apply -k`.

## Background: namespaces and Pod Security Admission

### Namespaces

A **namespace** is a named partition of the cluster. Object names only have to be unique within a namespace, `kubectl` commands default to one (`default` unless the context says otherwise), and policies can be attached per namespace. Everything in steps 13 and 16 lives in `default`. The portfolio gets its own, created by `namespace.yaml`, which is why every command on this page carries `-n portfolio`.

The trap, which I fell into twenty seconds after a successful rollout: `kubectl get pods` with no `-n` shows `default`, where there is no portfolio, and it looks like the deploy vanished. `-A` shows all namespaces when you are not sure where something went.

### Pod Security Admission and the `restricted` level

Since Kubernetes 1.25, the API server can enforce a baseline of pod security per namespace, through labels on the namespace. Three levels exist:

| Level | Allows | Meant for |
|---|---|---|
| `privileged` | Anything | System components |
| `baseline` | Most things, but no host namespaces, no privileged containers, no dangerous capabilities | General workloads |
| `restricted` ✅ | Only pods that follow current hardening best practice | Anything that doesn't need special powers, which is most web apps |

One label turns it on: `pod-security.kubernetes.io/enforce: restricted` on the namespace. From then on the API server **rejects** any pod in that namespace that fails the rules, at `apply` time, with an error naming the offending field. There is no runtime surprise; a non-compliant Deployment simply never gets its pods.

What `restricted` requires, field by field, and what each one protects against:

| Field | Required value | What it prevents |
|---|---|---|
| `securityContext.runAsNonRoot` | `true` | The process running as UID 0 inside the container, which is UID 0 on the node if anything escapes |
| `securityContext.allowPrivilegeEscalation` | `false` | A process gaining more privileges than its parent, for example through a setuid binary |
| `securityContext.capabilities.drop` | `["ALL"]` | Linux capabilities, the fine-grained pieces of root power. `NET_BIND_SERVICE` may be added back if the app genuinely needs a port below 1024. |
| `securityContext.seccompProfile.type` | `RuntimeDefault` or `Localhost` | Unusual system calls; the default profile blocks the ones nothing normal needs |
| No `hostNetwork`, `hostPID`, `hostIPC`, no `privileged`, no `hostPath` volumes | absent / false | The container touching the node directly |

The knock-on effects explain two choices in the portfolio's Dockerfile and manifest. Standard nginx starts as root and then drops to a worker user; that fails `runAsNonRoot`. The `nginxinc/nginx-unprivileged` image starts as a normal user instead. A normal user cannot bind port 80 (below 1024), so that image listens on **8080**, and the Service maps 80 → 8080. `readOnlyRootFilesystem: true` is not required by `restricted`, but it is in the same spirit: nginx is given two `emptyDir` scratch volumes for the only places it needs to write (`/tmp` and `/var/cache/nginx`) and nothing else is writable.

## Background: probes, pull policy and rolling updates

### Readiness versus liveness

Both probes are HTTP requests to `/healthz` on the container, on a schedule. They answer different questions and have different consequences:

| | Readiness probe | Liveness probe |
|---|---|---|
| Question | "Can this pod take traffic right now?" | "Is this process alive, or hung?" |
| When it fails | The pod is **removed from the Service's endpoints**; no traffic reaches it. Nothing is restarted. | The container is **killed and restarted** by the kubelet |
| Typical cause | Still starting up, warming a cache, overloaded | Deadlock, out of file handles, stuck event loop |
| Danger if misused | Too strict, and the app is never "ready" and gets no traffic | Too strict, and a slow but healthy app is restarted in a loop |

Readiness is also what makes the rollout output in section 4 say "0 of 2 updated replicas are available" and then "1 of 2": a replica counts as available only once its readiness probe has passed. A third kind, the **startup probe**, holds both of these off for slow starters; [step 21](../10-dns/21-local-dns-adguard.md) needs it.

### `imagePullPolicy`, and the `latest` trap

Before starting a container, the kubelet decides whether to download the image:

| Policy | Behaviour | Default when |
|---|---|---|
| `Always` | Check the registry every time the pod starts | The tag is `latest` or there is no tag |
| `IfNotPresent` | Use the local copy if there is one, pull only if missing | Any other tag |
| `Never` | Never pull; the image must already be on the node | Never a default |

The portfolio manifest sets `IfNotPresent` explicitly, which with a `latest` tag is a silent trap. The first deploy pulls `latest`. You push new code, CI pushes a new `latest`, you `apply` again. The Deployment's pod template has not changed (same image string), so no rollout starts. Even if you force one, the node already has an image called `latest` and `IfNotPresent` tells it not to look again. The site stays old and nothing reports an error.

The fix is not to flip the policy to `Always`; it is to **change the tag**. A new tag changes the pod template, which triggers a rollout, and the node has never seen that tag, so it pulls. That is what the short-SHA tag and the Kustomize `images:` block are for.

### Rolling updates: `maxSurge` and `maxUnavailable`

When the pod template changes, a Deployment replaces old pods with new ones gradually. Two knobs control how:

| Setting | Meaning | Portfolio value |
|---|---|---|
| `maxSurge` | How many pods **above** `replicas` may exist during the rollout | `1` |
| `maxUnavailable` | How many pods **below** `replicas` may be unready during the rollout | `0` |

With `replicas: 2`, `maxSurge: 1`, `maxUnavailable: 0`, a rollout goes like this:

```text
 start:    old-A  old-B                     (2 ready)
 step 1:   old-A  old-B  new-C              (surge to 3; wait for new-C readiness)
 step 2:   old-A  new-C                     (old-B removed only after new-C is ready; still 2 ready)
 step 3:   old-A  new-C  new-D              (surge again)
 step 4:   new-C  new-D                     (done; never fewer than 2 ready at any moment)
```

A readiness probe is what makes this safe. Without one, a new pod counts as ready the instant its container starts, and the old pod is removed before the new one can actually serve. `maxUnavailable: 0` with no readiness probe is a false promise.

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

## How the pieces fit

```text
 laptop (arm64)                 GitHub                          GHCR                    server (amd64)
 ──────────────                 ──────                          ────                    ──────────────
 git push master ──────────────► Actions runner (amd64)
                                 docker build --platform linux/amd64
                                 docker push ───────────────────► ghcr.io/bernado6/portfolio
                                                                   :latest, :<short-sha>
                                                                   (public)
 kubectl apply -k k8s/ ─────────────────────────────────────────────────────────────────► API server
                                                                                          creates ns, svc,
                                                                                          deploy, ingress
                                                                   ◄──── containerd pulls amd64 manifest ──
                                                                                          2 pods, port 8080
 browser: http://portfolio.home.arpa  (Host header) ─────────────────────────────────────► Traefik → Service :80 → pods :8080
```

Two separate flows: the image goes laptop → GitHub → GHCR → server, and never touches the laptop's Docker. The manifests go laptop → API server directly, as in step 16.

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

`git status --short` prints one line per changed or untracked file, with `??` marking untracked ones. This is the moment to look for things that should not be committed.

Two stray files, `portfolio@2.0.0` and `react-scripts`, were created by a mistyped `npm` command. Delete them, don't commit them:

```bash
rm -f 'portfolio@2.0.0' react-scripts
```

The quotes around `portfolio@2.0.0` are because `@` can be special in some shells; `-f` suppresses an error if a file is already gone.

Set the Ingress host to the `.home.arpa` name used across this guide (the default in the repo is `portfolio.homelab.local`, and `.local` misbehaves on macOS):

```bash
sed -i '' 's/portfolio\.homelab\.local/portfolio.home.arpa/g' k8s/ingress.yaml
```

Same macOS `sed -i ''` form as step 15; the backslashes escape the dots so they match literally, and `/g` replaces every occurrence in the file (the host appears once in `rules:` and may appear again under `tls:` later). Check with `grep host k8s/ingress.yaml`.

Then commit everything and push:

```bash
git add -A
git commit -m "Container image, GHCR workflow and K3s manifests"
git push origin master
```

`git add -A` stages everything, which is why the `git status` check came first. The push is what triggers the build.

#### 2. Watch the build

Pushing to `master` triggers `.github/workflows/docker-image.yml`. On GitHub: **Actions → Build and publish container image**. About three minutes. It publishes:

- `ghcr.io/bernado6/portfolio:latest`
- `ghcr.io/bernado6/portfolio:<short-sha>`

Open the run and expand the build step: the log shows the Dockerfile instructions executing on the runner, the `linux/amd64` platform, and at the end the digest it pushed. Note the short SHA from the tag list; section 6 uses it.

#### 3. Make the package public (once)

GHCR packages are private by default, and a private image needs a pull secret in the cluster. The site is public anyway, so make the image public:

**GitHub → your profile → Packages → portfolio → Package settings → Change visibility → Public.**

Then prove the cluster will be able to pull it, from the laptop with no login:

```bash
docker manifest inspect ghcr.io/bernado6/portfolio:latest | grep architecture
```

`docker manifest inspect` asks the registry for the manifest list without downloading any layers, and `grep architecture` keeps the per-platform lines. Running it logged out is the point: if it works for anonymous you, it works for the node.

```text
            "architecture": "amd64",
            "architecture": "unknown",
```

`amd64` is the image the server will run. The `unknown` entry is not a broken build: it's the **provenance attestation** that `docker/build-push-action` attaches alongside the image (a signed record of how it was built). Kubernetes ignores it and pulls the amd64 one.

If `docker manifest` complains, `curl -sI https://ghcr.io/v2/bernado6/portfolio/manifests/latest` returning anything other than `401` is enough.

That `curl` hits the registry API directly. `-I` asks for headers only, `-s` keeps it quiet. A `401 Unauthorized` means the package is still private.

### Part B: deploy (from the laptop)

#### 4. Apply with Kustomize

```bash
cd ~/Desktop/MyProjects/mle/Data-Science-Personal-Website
kubectl apply -k k8s/
kubectl -n portfolio rollout status deployment/portfolio
kubectl -n portfolio get pods,svc,ingress
```

`apply -k` reads `k8s/kustomization.yaml`, which lists the four files and sets the image tag in one place. Everything lands in its own **`portfolio` namespace**, so `-n portfolio` is needed on every command, or you'll get "not found" from the `default` namespace.

| Command | What it does |
|---|---|
| `kubectl apply -k k8s/` | Build the Kustomize output and apply it. `-k` takes a directory containing `kustomization.yaml`, not a file. |
| `kubectl -n portfolio rollout status deployment/portfolio` | Block and print progress until every replica is available, or fail after the Deployment's `progressDeadlineSeconds`. The best way to know a deploy has actually finished. |
| `kubectl -n portfolio get pods,svc,ingress` | The three things you want to see afterwards, in that namespace |

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

Two more things in that output worth noticing. Kustomize ordered the objects sensibly, namespace first, so nothing was created in a namespace that didn't exist yet. And the pod names share the middle part, `5ddbc576ff`: that is the hash of the pod template, identifying the ReplicaSet both pods belong to. After a rollout, the new pods get a different hash.

#### 5. Test with a Host header, then with a name

```bash
curl -s -o /dev/null -w '%{http_code}\n' -H "Host: portfolio.home.arpa" http://192.168.100.22/healthz
curl -s -H "Host: portfolio.home.arpa" http://192.168.100.22 | head -3
```

The first `curl` throws the body away (`-o /dev/null`) and prints only the HTTP status code (`-w '%{http_code}\n'`), which is the cleanest way to check a health endpoint. The second fetches the home page and shows its first three lines. Both set the `Host` header by hand, as in step 16, so they work before any name resolves.

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

| Command | What it does |
|---|---|
| `kustomize edit set image <name>=<name>:<tag>` | Rewrites the `images:` block in the `kustomization.yaml` of the current directory. The standalone `kustomize` binary (`brew install kustomize`) has this `edit` subcommand; `kubectl kustomize` only renders. You could also edit `newTag:` by hand. |
| `kubectl apply -k k8s/` | The Deployment's image string has changed, so the API server starts a rollout |
| `rollout status` | Watch it finish, as before |

Commit the changed `kustomization.yaml` afterwards, so the repo records which build is deployed.

The Deployment is `maxUnavailable: 0, maxSurge: 1`, so a new pod comes up and passes its readiness probe before an old one goes away. Watch it with `kubectl -n portfolio get pods -w`.

`-w` is "watch": it keeps printing lines as pods change state. You will see a third pod appear with a new template hash, go `Running`, then one old pod go `Terminating`, then the sequence repeats once. `kubectl -n portfolio rollout history deployment/portfolio` lists past revisions, and `rollout undo` goes back one if a release is bad.

## What's in the manifests worth understanding

| Setting | Where | Why it's there |
|---|---|---|
| `pod-security.kubernetes.io/enforce: restricted` | `namespace.yaml` | The namespace **refuses** pods that run as root, keep capabilities, or lack a seccomp profile. The Deployment is written to pass. Try removing `runAsNonRoot` and watch the apply get rejected. |
| `nginxinc/nginx-unprivileged`, port **8080** | Dockerfile | Non-root nginx can't bind port 80 (needs root or a capability). 8080 avoids the problem. |
| `readOnlyRootFilesystem: true` + `emptyDir` for `/tmp` and `/var/cache/nginx` | `deployment.yaml` | The container can't write anywhere except the two scratch volumes. |
| Readiness and liveness probes on `/healthz` | `deployment.yaml` | Readiness gates traffic; liveness restarts a hung container. Both CKAD staples. |
| `requests` 10m/32Mi, `limits` 200m/64Mi | `deployment.yaml` | Two replicas cost about 130MB at most. Fine on 8GB. |
| `images:` with `newTag` | `kustomization.yaml` | One place to bump the version. |

The resources row deserves a sentence. `requests` are what the scheduler sets aside: 10 millicores (1% of one core) and 32MiB per pod. `limits` are the ceiling: a pod that tries to use more than 64MiB of memory is killed (`OOMKilled`), and one that wants more than 200m of CPU is throttled rather than killed. For a static site served by nginx these are generous. Step 19 revisits them with real measurements.

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

If any of those do happen to you, the diagnostic path is the same each time: `kubectl -n portfolio get pods` for the status, `kubectl -n portfolio describe pod <name>` for the Events at the bottom (pull errors and Pod Security rejections show here), and `kubectl -n portfolio logs <name>` for anything the container itself said, such as `exec format error`.

## Lessons

> 💡 **What I learned:**
> - **Check the CPU architecture before you build.** `uname -m` on the laptop, `uname -m` on the server. If they differ, build on CI or pass `--platform`.
> - **The cluster pulls, you don't push.** A registry is the hand-off point. For anything on GitHub, GHCR plus Actions is free and zero-maintenance.
> - **`latest` + `IfNotPresent` is a silent no-op on redeploy.** Pin tags to a SHA and bump them. The tag change is what triggers the rollout.
> - **Namespaces are easy to forget.** "Not found" after a successful apply usually means you're looking in `default`.
> - **A restricted namespace is a great teacher.** It rejects anything sloppy, and the error message tells you exactly which field.
> - **Keep manifests with the app they describe.** The Dockerfile, the port, the health endpoint and the Deployment change together.

## Glossary

| Term | Meaning |
|---|---|
| Image | A layered filesystem snapshot plus config (command, port, user); no kernel inside |
| Layer | One compressed archive of filesystem changes, identified by digest, shared between images |
| Registry | An HTTP server that stores images; Docker Hub, GHCR |
| Repository | The image's name within a registry (`bernado6/portfolio`); lowercase only |
| Tag | A movable label on an image (`latest`, `<short-sha>`) |
| Digest | `sha256:…`, the content hash; the only immovable reference |
| Manifest | JSON listing an image's layers and config |
| Manifest list / image index | A list of manifests, one per platform, behind a single tag |
| Multi-arch image | A tag whose manifest list covers more than one architecture |
| Provenance attestation | A signed build record stored alongside the image; shows as `architecture: unknown` |
| `arm64` / `amd64` | Apple Silicon / Intel and AMD; a binary built for one won't run on the other |
| `exec format error` | The kernel's message when asked to run a binary for the wrong architecture |
| `--platform linux/amd64` | Build (or pull) for a specific architecture regardless of the host |
| GitHub Actions | GitHub's CI; runs a workflow on each push, on amd64 runners by default |
| GHCR | GitHub Container Registry, `ghcr.io`; packages are private until you change visibility |
| `imagePullSecrets` | How a pod authenticates to a private registry; not needed for public images |
| containerd | The container runtime on the K3s node; it does the pulling |
| Kustomize / `apply -k` | Built-in overlay tool: a `kustomization.yaml` lists resources and transforms |
| `images:` / `newTag` | The Kustomize transform that sets an image's tag in one place |
| Namespace | A named partition of the cluster; `-n` selects one, `-A` shows all |
| Pod Security Admission | Per-namespace enforcement of pod hardening levels via labels |
| `restricted` | The strictest level: non-root, no privilege escalation, all capabilities dropped, seccomp on |
| `runAsNonRoot` / `allowPrivilegeEscalation` / `capabilities.drop` / `seccompProfile` | The four `securityContext` fields `restricted` checks |
| `readOnlyRootFilesystem` | The container cannot write outside its mounted volumes |
| `emptyDir` | A scratch volume that lives as long as the pod |
| Readiness probe | Fails: pod removed from the Service, no restart |
| Liveness probe | Fails: container restarted |
| `imagePullPolicy` | `Always`, `IfNotPresent` or `Never`; `IfNotPresent` plus `latest` never re-pulls |
| Rolling update | Replacing pods gradually when the template changes |
| `maxSurge` / `maxUnavailable` | Extra pods allowed above `replicas` / pods allowed unready below it |
| ReplicaSet hash | The middle part of a pod name; same hash means same pod template |
| `rollout status` / `history` / `undo` | Wait for, list, or revert Deployment revisions |
| Requests / limits | Scheduler reservation / hard ceiling; memory over the limit means `OOMKilled` |

---

[◀ Previous: Exposing an App With an Ingress](../06-working-with-the-cluster/16-nginx-ingress.md) · [🏠 Home](../../README.md) · [Next: Monitoring: Prometheus and Grafana ▶](../08-observability/18-prometheus-grafana.md)
