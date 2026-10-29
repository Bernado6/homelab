# 13. My First App

A tiny test app (`traefik/whoami`) using the three most important Kubernetes objects: a **Deployment**, a **Service** and an **Ingress**.

Sixty lines of YAML, one `kubectl apply`, and a web page that tells you which pod served it. It is the smallest useful thing a cluster can do, and it touches every concept the CKAD exam is built on. This page reads the manifest line by line, then follows one browser request through all three objects to the container and back.

## What

Deploy two copies of a small web server, give them one stable address inside the cluster, and expose that address to the home network through Traefik, so that `http://<SERVER_IP>` in a browser shows the app.

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

## Background: how Kubernetes objects work

Everything in Kubernetes is an **object**: a record stored in the cluster's datastore with a `kind`, a `metadata.name`, and a `spec` describing what you want. You do not run commands like "start two containers"; you write down the desired state and **apply** it. Controllers inside the cluster then compare desired with actual, continuously, and act on the difference. This is **declarative** management, and it is why the same `kubectl apply -f` works whether you are creating something new or changing something that exists.

Every object in the manifest has the same four top-level keys:

| Key | What it is | In this manifest |
|---|---|---|
| `apiVersion` | Which API group and version defines this kind | `apps/v1` for Deployment, `v1` (core) for Service, `networking.k8s.io/v1` for Ingress |
| `kind` | The type of object | `Deployment`, `Service`, `Ingress` |
| `metadata` | Name, namespace, labels, annotations | `name: hello` on all three. No namespace given, so they land in `default` |
| `spec` | The desired state, different for every kind | The interesting part |

A `---` line separates documents in one YAML file, so one file can hold several objects and one `apply` creates all of them. The three objects here share the name `hello` but are different kinds, so there is no clash.

### Labels and selectors: how objects find each other

Objects in Kubernetes do not hold pointers to each other. They find each other by **labels**, which are free-form `key: value` tags in `metadata.labels`, and **selectors**, which are queries over those labels. In this manifest the label is `app: hello` and it is doing all the joining:

```text
 Deployment "hello"
   spec.selector.matchLabels: app=hello   ── "the pods I manage are the ones labelled app=hello"
   spec.template.metadata.labels: app=hello ── "and here is the label I stamp on each pod I create"
                 │
                 ▼
 Pod hello-xxxxx  (labels: app=hello)
 Pod hello-yyyyy  (labels: app=hello)
                 ▲
                 │
 Service "hello"
   spec.selector: app=hello               ── "send my traffic to whatever pods carry app=hello"
```

Change the label on one pod and the Service stops sending it traffic and the Deployment creates a replacement, because by the only test that matters, that pod is no longer one of "hello"'s. This loose coupling is what makes rolling updates, canaries and debugging-by-relabelling possible, and it is a favourite CKAD exam topic.

## Background: Deployment, ReplicaSet and Pod

You wrote a Deployment. The cluster ended up with two Pods. There is a third object in between that the manifest never mentions:

```text
 Deployment "hello"                 ── owns the rollout strategy and the history
   └── ReplicaSet "hello-<hash>"    ── owns the count: keeps exactly `replicas` pods alive
         ├── Pod "hello-<hash>-xxxxx"
         └── Pod "hello-<hash>-yyyyy"
```

| Object | Job | Why it exists separately |
|---|---|---|
| **Pod** | Run one or more containers together, with one IP and shared storage | The unit of scheduling. Pods are disposable: when one dies it is not repaired, it is replaced |
| **ReplicaSet** | Keep *N* identical pods running, from a template. Too few: create. Too many: delete | Self-healing. You almost never create one by hand |
| **Deployment** | Manage ReplicaSets over time. On a change to the pod template it creates a *new* ReplicaSet and shifts pods from old to new gradually | Rolling updates and rollbacks with zero downtime |

Everything under `spec.template` is a complete **pod spec**: it is exactly what you would write as `kind: Pod`, embedded inside the Deployment so the ReplicaSet knows what to stamp out. The pod names in `kubectl get pods` are `hello-<replicaset hash>-<random>`, which is the ownership chain written into the name.

`replicas: 2` is deliberate. With two pods, every refresh of the page can land on a different one, which makes the Service's load-balancing *visible*. It is also the first time you see that your app is not "a server" but "a number of interchangeable copies".

### Resource requests and limits

The one part of this manifest that is not strictly required, and the part most often left out of tutorials:

```yaml
resources:
  requests:
    cpu: 10m
    memory: 16Mi
  limits:
    memory: 64Mi
```

| | Request | Limit |
|---|---|---|
| Means | "I need at least this much." The scheduler only places the pod on a node with that much free | "Never let me use more than this" |
| Used for | **Placement**: the scheduler adds up requests to know whether a node is full | **Enforcement**: the kernel caps the container |
| CPU | `10m` = 10 millicores = 1% of one core | No CPU limit here (see below) |
| Memory | `16Mi` = 16 mebibytes reserved | `64Mi`: above this the container is **OOMKilled** and restarted |

Why a memory limit but no CPU limit: CPU is **compressible**. A container that wants more CPU than is available is slowed down, not killed, so a CPU limit mostly just throttles your own app for no gain on a single-tenant box. Memory is **incompressible**. A container that keeps growing will take the node down with it unless something stops it, and the limit is that something. On an 8GB machine running a dozen things, every pod that lacks a memory limit is a pod that can starve the others.

The combination (requests set, limits set only partly) places the pod in the **Burstable** QoS class: guaranteed its requests, allowed to use more, first to be evicted after pods with no requests at all when memory runs short. You will meet the opposite failure in [step 18](../08-observability/18-prometheus-grafana.md), where a limit set too *low* makes Grafana crash-loop.

## Background: the Service

Pods have IPs, but they are **ephemeral**: every replacement pod gets a new address, and there are two of them anyway. Something has to give the pair one stable name and address and spread traffic across whichever pods currently exist. That is a **Service**.

| Field | Meaning here |
|---|---|
| `selector: app: hello` | The pods to send traffic to, found by label. The cluster keeps an **EndpointSlice** listing their current IPs, updated as pods come and go |
| `port: 80` | The port the Service listens on, at its own IP |
| `targetPort: 80` | The port on the pod the traffic is forwarded to. The same here, but the two can differ, and that difference is the cause of the `port-forward` confusion in the Challenges table |
| `type` (not set) | Defaults to **ClusterIP** |

The Service types, and which one this is:

| Type | What it gives you | Reachable from |
|---|---|---|
| **ClusterIP** ✅ (default, used here) | A virtual IP in `10.43.0.0/16` and a DNS name `hello.default.svc.cluster.local` | Inside the cluster only. Traefik is inside the cluster, which is all we need |
| NodePort | ClusterIP plus a high port (30000 to 32767) opened on every node | `http://<SERVER_IP>:3xxxx`. Works, ugly, and you end up remembering port numbers |
| LoadBalancer | NodePort plus an external IP from a cloud, or from ServiceLB on K3s | Used by Traefik itself (ports 80/443) and later by AdGuard (port 53) in [step 21](../10-dns/21-local-dns-adguard.md). Not for every app |
| ExternalName | A DNS alias to something outside the cluster | Not here |

A ClusterIP is not an interface anywhere; it is a rule. kube-proxy programs the node so that packets to `10.43.x.x:80` are rewritten to one of the pod IPs in the EndpointSlice, chosen roughly at random. That is the load-balancing, and it is why refreshing the page alternates between pods.

Inside the cluster, CoreDNS from [step 12](12-installing-k3s.md) resolves the Service's name, so another pod could simply `curl http://hello` (same namespace) or `curl http://hello.default.svc.cluster.local` (from anywhere). The address is stable; the pods behind it can churn.

## Background: the Ingress and the Ingress controller

A ClusterIP is invisible from the laptop. The last object bridges that gap, and it is really two things working together:

| | Ingress (the object) | Ingress controller (the program) |
|---|---|---|
| What it is | A **routing rule**: "requests matching this host and path go to this Service on this port" | A reverse proxy that **watches** Ingress objects and configures itself from them |
| Who provides it | You, in YAML | K3s, as the `traefik` pod in `kube-system` |
| Without the other | A rule nobody reads; the Ingress sits there doing nothing | A proxy with no routes; every request gets 404 |

An Ingress on its own does nothing. On a bare `kubeadm` cluster, people write Ingresses, nothing happens, and the missing piece is the controller. K3s avoids that by bundling Traefik and making it the default **IngressClass**, which is why this manifest does not need an `ingressClassName`.

This particular Ingress has **no `host`**. A rule without a host matches every hostname, so it is a catch-all: whatever name or IP you type, Traefik sends `/` and everything under it (`pathType: Prefix`) to the `hello` Service on port 80. That is fine for a first test and wrong for a second app, because two catch-alls would fight. [Step 16](../06-working-with-the-cluster/16-nginx-ingress.md) adds `host:` rules so each app owns a name.

### How Traefik is reachable at the server's IP at all

Traefik runs as a pod with a `10.42.x.x` address, which the laptop cannot reach either. Its Service is `type: LoadBalancer`, and the `svclb-traefik` pod from step 12 is K3s's ServiceLB binding ports **80** and **443** on the node's real IP and forwarding them to Traefik. Nothing in this manifest arranged that; it came with the cluster. That is the last hop that makes `http://<SERVER_IP>` work.

### Why an Ingress and not the alternatives

| Way to reach a pod from the laptop | How | Verdict |
|---|---|---|
| **Ingress through Traefik** ✅ | A rule; Traefik already listens on 80/443 | Permanent, one entry point for every app, host-based routing and TLS later. The right default |
| `kubectl port-forward` | A temporary tunnel from a `kubectl` process to one pod or Service | Debugging only. Dies when you close the terminal, listens on the machine running `kubectl`, one target at a time. Good for setup wizards ([step 21](../10-dns/21-local-dns-adguard.md)); bad as a way to "expose" anything |
| NodePort Service | Opens a port in the 30000s on the node | Works without Traefik; port numbers to remember, no names, no TLS |
| LoadBalancer Service per app | ServiceLB binds a port on the node IP | Port 80 is already Traefik's; two apps cannot share it. Reserved for non-HTTP things like DNS |

## Background: the two private ranges inside the cluster

| Address range | What it is |
|---|---|
| `192.168.x.x` | Home network |
| `10.43.x.x` | Kubernetes Service IPs (K3s default) |
| `10.42.x.x` | Pod IPs (K3s default) |

Both `10.x` ranges are private addresses ([RFC 1918](https://www.rfc-editor.org/rfc/rfc1918)), chosen by K3s so they do not collide with a typical home `192.168.x.x` LAN. They are different in kind:

| | Pod network `10.42.0.0/16` | Service network `10.43.0.0/16` |
|---|---|---|
| Assigned by | Flannel, one IP per pod, from this node's slice `10.42.0.0/24` | The API server, one virtual IP per Service |
| Exists as | A real address on a real virtual interface inside the pod | A rule in kube-proxy's tables; no interface anywhere has it |
| Lifetime | Dies with the pod | Lives as long as the Service object |
| Visible from the LAN | No | No |

Both appear in `hostname -I` on the server (`10.42.0.0`, `10.42.0.1` on the `flannel.1` and `cni0` interfaces), which is the confusing extra output the [troubleshooting](../troubleshooting.md) page mentions. Only the `192.168.x.x` address is reachable from the laptop.

## Background: what `traefik/whoami` is, and why it is the right first app

`traefik/whoami` is a tiny Go web server, a few megabytes, with no configuration, that answers every request with a plain-text dump of **what it saw**: its own hostname (the pod name), its own IPs, the client's address, and every HTTP header the request arrived with. A response looks roughly like:

```text
Hostname: hello-<hash>-xxxxx            ← the pod name: which replica answered
IP: 127.0.0.1
IP: 10.42.0.x                           ← the pod's own IP
RemoteAddr: 10.42.0.y:zzzzz             ← who connected to the pod: Traefik, not your laptop
GET / HTTP/1.1
Host: 192.168.100.x                     ← what you typed in the address bar
User-Agent: Mozilla/5.0 …
X-Forwarded-For: 192.168.100.n          ← your laptop's address, added by Traefik
X-Forwarded-Host: 192.168.100.x
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Forwarded-Server: traefik-<hash>      ← which Traefik pod proxied it
X-Real-Ip: 192.168.100.n
```

That one page proves the whole chain: the `Hostname` line shows the Deployment made pods and the Service is spreading load; `RemoteAddr` being a `10.42.x.x` address shows the request came *through* Traefik rather than straight to the pod; the `X-Forwarded-*` headers are the proxy saying so and preserving the original client. Any app you deploy later that misbehaves can be swapped for `whoami` to check whether the problem is the routing or the app. It is the `echo` of Kubernetes.

## When

Straight after [step 12](12-installing-k3s.md), on the server over SSH, with the homelab repo cloned so the manifest path exists. (From [step 15](../06-working-with-the-cluster/15-kubectl-from-laptop.md) onward the same commands run from the laptop.)

## How

### 1. Apply the manifest

Apply it from the repo root:

```bash
kubectl apply -f apps/hello/hello.yaml
kubectl get pods,svc,ingress
```

`kubectl apply -f <file>` sends every object in the file to the API server, creating the ones that do not exist and updating the ones that do. Expect three confirmation lines:

```text
deployment.apps/hello created
service/hello created
ingress.networking.k8s.io/hello created
```

`kubectl get pods,svc,ingress` asks for three kinds at once in the current namespace (`default`). In the first few seconds the pods show `ContainerCreating` while the image is pulled; run it again and expect the shape:

```text
NAME                         READY   STATUS    RESTARTS   AGE
pod/hello-<hash>-xxxxx       1/1     Running   0          …
pod/hello-<hash>-yyyyy       1/1     Running   0          …

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/hello        ClusterIP   10.43.x.x      <none>        80/TCP    …
service/kubernetes   ClusterIP   10.43.0.1      <none>        443/TCP   …

NAME                              CLASS     HOSTS   ADDRESS          PORTS   AGE
ingress.networking.k8s.io/hello   traefik   *       192.168.100.x    80      …
```

Reading it: two pods from one Deployment, each `1/1` (one container, ready). The Service has a `10.43.x.x` ClusterIP and no external IP, as a ClusterIP should. (`service/kubernetes` is the API server's own Service and is always there.) The Ingress shows `CLASS traefik` even though the manifest never said so, because Traefik is the default class; `HOSTS *` is the catch-all; and `ADDRESS` is the node IP that ServiceLB gave Traefik.

### 2. Open it

Opening `http://<SERVER_IP>` in a browser shows the whoami page. Refresh and the `Hostname` switches between the two pods.

Use the LAN address from [step 11](../03-ubuntu-server/11-first-login-and-ssh.md). Plain `http://`; port 80 is implied. The browser will say **Not Secure**, which is correct for plain HTTP and is fixed in [step 20](../09-tls/20-https-with-cert-manager.md). Read the page against the annotated example in the Background: `Hostname` is the pod, `X-Forwarded-For` is your laptop.

From the server itself the same check is `curl http://localhost`, which goes to Traefik on port 80 and prints the same text.

### 3. Look at what was created

Not in the original three commands, but worth two minutes now that there is something to look at:

```bash
kubectl get replicaset                       # the object between Deployment and Pods
kubectl get pods -o wide                     # adds each pod's 10.42.x.x IP and the node
kubectl get endpointslices                   # the pod IPs the Service is currently sending to
kubectl describe ingress hello               # the rule as Traefik sees it
kubectl logs deploy/hello                    # whoami logs each request it serves
```

`get replicaset` shows one ReplicaSet named `hello-<hash>` with `DESIRED 2, CURRENT 2, READY 2`. `-o wide` shows the two pod IPs, which you can match against the `IP:` lines on the web page. The EndpointSlice lists the same two IPs: that is the Service's live target list. Delete a pod with `kubectl delete pod <name>` and watch `get pods` again: a replacement appears within seconds with a new name and IP, and the EndpointSlice updates. That is the ReplicaSet doing its one job.

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

The same path with every hop named, including the ones the manifest did not create:

```text
 laptop 192.168.100.n
   │  GET http://192.168.100.x/        (DNS not involved: you typed an IP)
   ▼
 node 192.168.100.x, port 80
   │  svclb-traefik (ServiceLB, from step 12): hostPort 80 → DNAT to Traefik's pod
   ▼
 Traefik pod 10.42.0.a
   │  reads every Ingress in the cluster
   │  this one has no host and path "/" Prefix → matches anything
   │  backend: Service "hello", port 80
   │  adds X-Forwarded-For: 192.168.100.n, X-Real-Ip, X-Forwarded-Host …
   ▼
 Service "hello", ClusterIP 10.43.x.x:80
   │  not a machine: a kube-proxy rule on the node
   │  EndpointSlice says app=hello pods are 10.42.0.b and 10.42.0.c
   │  picks one, rewrites destination to <pod IP>:80 (targetPort)
   ▼
 Pod hello-<hash>-xxxxx, 10.42.0.b, container port 80
   │  whoami builds the text page: its Hostname, its IPs, the headers it received
   ▼
 back the same way: pod → Traefik → node port 80 → laptop
```

| Hop | Object or component | Address it works at | Created by |
|---|---|---|---|
| 1 | Node's network port | `192.168.100.x:80` | The router's DHCP ([step 14](../05-networking/14-dhcp-reservation.md) pins it) |
| 2 | ServiceLB (`svclb-traefik`) | hostPort 80 on the node | K3s, step 12 |
| 3 | Traefik, the Ingress controller | `10.42.0.a` | K3s, step 12 |
| 4 | Ingress `hello` | a rule, no address | this manifest |
| 5 | Service `hello` | `10.43.x.x:80` | this manifest |
| 6 | Pod `hello-…` | `10.42.0.b:80` | the ReplicaSet, from the Deployment in this manifest |

Everything from hop 3 down is standard Kubernetes and would look identical on any cluster. Hops 1 and 2 are how K3s solves "there is no cloud load balancer here", and they come back in [step 21](../10-dns/21-local-dns-adguard.md) when the same trick publishes port 53.

## Challenges and solutions

The app itself applied cleanly. The problems came from trying `kubectl port-forward` as a second way in, which is where the difference between a Service port and a container port, and between the server's `localhost` and the laptop's, became real:

| Challenge | What was going on | Solution |
|---|---|---|
| `port-forward` says "does not have a service port 80" | With `svc/<name>`, the right-hand port is the **Service** port, not the container port. | `svc/nginx-svc 8000:5000` works and kubectl maps 5000 → 80 for you. With `pod/<name>` use the container port directly. |
| `port-forward` works on the server but `localhost:8000` on the laptop says connection refused | The forward listens on the **server's** localhost, because that is where `kubectl` was running. | Either run it with `--address 0.0.0.0` and open `http://<SERVER_IP>:8000`, or tunnel from the laptop: `ssh -L 8000:localhost:8000 user@<SERVER_IP>` then open `http://localhost:8000`. Better still, use the Ingress, which is what this step is for. |
| Browser shows **Not Secure** | Plain HTTP on the home network. | Expected until HTTPS with cert-manager in [step 20](../09-tls/20-https-with-cert-manager.md). |
| Pods stay `Pending` | The scheduler cannot find room for the requests, or the node is `NotReady` | `kubectl describe pod <name>`; the Events at the bottom say why. With `10m` CPU and `16Mi` memory this should not happen on an idle node |
| Pods `ImagePullBackOff` | The node cannot download `traefik/whoami` from Docker Hub | Check internet from the server; `kubectl describe pod` shows the pull error |
| Page loads but `Hostname` never changes | Browser keep-alive reusing one connection to Traefik | Hard-refresh, or `curl http://<SERVER_IP>` several times; each `curl` is a new connection |
| A second app applied with its own catch-all Ingress, and the wrong app answers | Two Ingresses with no `host` both match everything | Give each Ingress a `host:` rule, [step 16](../06-working-with-the-cluster/16-nginx-ingress.md) |
| `kubectl apply` says the file was not found | Not in the repo root | `cd` to the clone of this repo; the path is relative: `apps/hello/hello.yaml` |

## Lessons

> 💡 **What I learned:** the whoami page is a great debugging tool. It shows the pod name, pod IP and the headers Traefik adds, so you can *see* the path the request took. Always set **resource requests and limits**, especially on an 8GB machine. These are core CKAD topics.

> 💡 **Also:**
> - **Labels are the glue.** Nothing in the manifest points at anything else; `app: hello` joins Deployment to Pods to Service. Change the label and the joins break.
> - **An Ingress is a rule, not a server.** Traefik is the server. K3s bundling it is why this worked without installing anything.
> - **A ClusterIP is a rule too.** It exists only in kube-proxy's tables on the node, which is why nothing outside the cluster can reach it and why `port-forward` is the wrong permanent answer.

## Glossary

| Term | Meaning |
|---|---|
| Object | Any record in the cluster: a `kind`, a `metadata.name`, a `spec`; managed declaratively |
| Declarative | You state the desired result and controllers make it so; `apply` creates or updates alike |
| Manifest | A YAML file of one or more objects, separated by `---` |
| `default` namespace | Where objects go when no namespace is given; where `hello` lives |
| Label / selector | A `key: value` tag on an object / a query over labels; how Deployments and Services find pods |
| Pod | The smallest unit Kubernetes runs; one IP, one or more containers, disposable |
| ReplicaSet | Keeps exactly N pods alive from a template; created and owned by the Deployment |
| Deployment | Manages ReplicaSets over time for rolling updates and rollbacks |
| Pod template | The `spec.template` block: a full pod spec the ReplicaSet stamps out |
| Request | Minimum resource a container needs; used by the scheduler for placement |
| Limit | Maximum a container may use; memory over the limit means OOMKilled |
| `10m` / `16Mi` / `64Mi` | 10 millicores (1% of a core) / 16 mebibytes / 64 mebibytes |
| OOMKilled | The kernel killed the container for exceeding its memory limit; exit code 137 |
| Compressible / incompressible | CPU can be throttled without harm; memory cannot, so it needs a limit |
| QoS class / Burstable | Kubernetes's eviction priority tiers; requests-without-full-limits is Burstable |
| Service | A stable virtual IP and DNS name in front of a changing set of pods |
| ClusterIP | The default Service type; reachable inside the cluster only |
| NodePort / LoadBalancer / ExternalName | The other Service types; a high node port / an external IP (ServiceLB on K3s) / a DNS alias |
| `port` / `targetPort` | The Service's own port / the pod port it forwards to |
| EndpointSlice | The live list of pod IPs behind a Service, maintained from the selector |
| kube-proxy | Programs the node so Service IPs are rewritten to pod IPs |
| CoreDNS | Resolves `hello.default.svc.cluster.local` to the ClusterIP inside the cluster |
| Ingress | A routing rule: host and path to Service and port. Does nothing without a controller |
| Ingress controller | The reverse proxy that reads Ingresses and routes traffic; Traefik here |
| IngressClass | Which controller handles an Ingress; Traefik is the K3s default so none is specified |
| `pathType: Prefix` | Match the path and everything under it |
| Catch-all Ingress | A rule with no `host`; matches every hostname. Fine once, a conflict twice |
| ServiceLB / `svclb-traefik` | K3s binding ports 80/443 on the node IP and forwarding to Traefik |
| `10.42.0.0/16` | The pod network (Flannel); real addresses on real virtual interfaces |
| `10.43.0.0/16` | The Service network; virtual addresses that exist only as rules |
| `X-Forwarded-For` and friends | Headers a reverse proxy adds to preserve the original client and host |
| `traefik/whoami` | A tiny web server that prints its hostname, IPs and the request headers; the cluster's `echo` |
| `kubectl port-forward` | A temporary tunnel from the machine running kubectl to one pod or Service; for debugging, not exposure |

---

[◀ Previous: Installing Kubernetes (K3s)](12-installing-k3s.md) · [🏠 Home](../../README.md) · [Next: A Fixed IP for the Server ▶](../05-networking/14-dhcp-reservation.md)
