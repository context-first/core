# Study Guide — Kubernetes Core (domain C)

> **DRAFT — NOT FOR PUBLICATION.** Eighth instance of the
> study-guide format. Scoped from domain C of
> [skills-inventory.md](skills-inventory.md). Anchored in
> movies-bartr's `deploy/movies/base/` manifests and the k3s/k3d
> + Traefik + Prometheus-Operator local-cluster pattern that
> every other guide in this curriculum assumes.

## Why this exists

Kubernetes is the layer everything else in the curriculum sits
on. Containers (A), local platform (B), Kustomize (D), GitOps
(E), observability (G), security (J) — all of them assume the
operator can read a manifest, find a pod, read its logs, debug a
crashloop, and trust their mental model of what's running.

The agent can generate Kubernetes manifests endlessly. Reading
them, predicting what they'll do, and recovering when they don't
do it is the operator's job. This guide makes that fluent.

**What this guide is anchored in:**

- movies-bartr's `deploy/movies/base/` — a real, working,
  spec-compliant set of K8s manifests: Namespace, Deployment,
  Service, Ingress, ServiceMonitor, NetworkPolicy.
- The **k3s/k3d + klipper-lb + Traefik** local-cluster pattern.
  Curriculum-wide assumption; the inventory's C2 and C16 rows
  encode the opinion.
- [study-guide-kustomize.md](study-guide-kustomize.md) Module 7
  for the dev-LoadBalancer / prod-IngressRoute pattern.
- [study-guide-observability.md](study-guide-observability.md)
  for ServiceMonitor + probes.
- [study-guide-security.md](study-guide-security.md) for
  NetworkPolicy + RBAC depth.

## How to use this guide

Same protocol as the other study guides. One curriculum-level
rule specific to this one: **the operator does every lab against
a real local cluster they own**, not a screenshot or a mocked-up
output. Predictions ("the pod will crashloop because…") are
written down *before* the apply; the apply confirms or
falsifies. K8s is too big to learn by reading; it's exactly the
right size to learn by predict-and-apply.

## Modules

### Module 1 — The cluster mental model (C1)

> **The API server is the only writer.** Internalize that one
> sentence and most of K8s makes sense.

#### Concept

A Kubernetes cluster has two kinds of machines:

- **Control plane node(s)** — run `kube-apiserver`, `etcd`,
  `kube-scheduler`, `kube-controller-manager`,
  `cloud-controller-manager`.
- **Worker nodes** — run `kubelet` + `containerd` (or another
  CRI runtime) + `kube-proxy`.

The single most useful invariant: **everything writes to the
API server; nothing else writes to etcd directly.**

- You type `kubectl apply -f deploy.yaml`. `kubectl` is an HTTP
  client; it `POST`s the manifest to `kube-apiserver`.
- The API server validates the object, persists it to `etcd`,
  and notifies watchers.
- A controller (the Deployment controller, in this case) sees
  the new Deployment, creates a ReplicaSet (write to API
  server), which creates Pods (write to API server).
- The scheduler sees an unscheduled Pod, picks a node, writes
  `spec.nodeName` (write to API server).
- The kubelet on that node sees a Pod assigned to it, asks
  containerd to start the containers, and reports status back
  (write to API server).

Every component is a watcher / reconciler of the API server.
This is what people mean by "declarative" and "reconciliation
loops" — you write *desired state* to the API server, and the
cluster's controllers work to make *actual state* match.

Three consequences:

1. **`kubectl get` and `kubectl describe` always tell you
   what's true,** because the API server is the source of truth.
2. **A "stuck" object is almost always a controller missing,
   misconfigured, or RBAC-denied** — not a kernel-level
   mystery.
3. **CRDs (Module 11) work exactly like built-in types** —
   they're just more objects in the API server, with their own
   controllers watching them.

#### Example

The Pod creation flow, traced top to bottom:

```
$ kubectl apply -f deploy.yaml
  └─ POST /apis/apps/v1/namespaces/movies/deployments → kube-apiserver
       └─ validate + write to etcd
            └─ deployment-controller (watching) sees the new Deployment
                 └─ POST /apis/apps/v1/.../replicasets → kube-apiserver
                      └─ replicaset-controller (watching) sees the new RS
                           └─ POST /api/v1/.../pods × N → kube-apiserver
                                └─ scheduler (watching) sees pending Pods
                                     └─ PATCH .../pods/<name> spec.nodeName=node1
                                          └─ kubelet on node1 (watching) sees it
                                               └─ asks containerd to start containers
                                                    └─ PATCH .../pods/<name>/status
```

Every arrow is an API server call. There is no out-of-band
channel.

#### Lab

1. On any cluster: `kubectl get events --sort-by='.lastTimestamp'
   -A | tail -20`. Read the sequence of events for a recent
   change. Identify the controllers involved.
2. **Watch the reconciliation in real time.** In one terminal:
   `kubectl get pods -w -n movies`. In another: `kubectl
   scale deployment movies-api -n movies --replicas=3`. Watch
   pods appear, schedule, become ready.
3. **Falsify "the API server is the only writer."** Try to write
   directly to etcd (you can't, without etcd credentials and
   knowing the binary key format). Confirm that *every* tool
   you've ever used (`kubectl`, k9s, Lens, Argo, Flux) talks to
   the API server.
4. `kubectl get --raw='/healthz?verbose'` — direct API server
   call, no abstraction. Read the component status.
5. **Find the controllers.** `kubectl get pods -n kube-system`
   on k3s; identify `kube-controller-manager`, `kube-scheduler`,
   `kube-apiserver`. (On k3s these may all be in one binary;
   look for `k3s-server`.)

#### Knowledge check

1. Name the four core control-plane components and one sentence
   each on what they do.
2. "The API server is the only writer." What does that mean
   concretely and why does it simplify debugging?
3. You `kubectl apply` a Deployment. List the chain of API server
   writes that happens before your container starts.
4. A Pod stays in `Pending` forever. Three places to look,
   based on the mental model.
5. CRDs (custom resources) behave like built-in resources because
   of one property of the API server. Which?

---

### Module 2 — Local distro: k3s, k3d, and what to skip (C2, C3)

> **Strong opinion:** k3s on VMs / bare metal, k3d on laptops.
> *Not* minikube, *not* kind, *not* Docker Desktop's Kubernetes.

#### Concept

Local Kubernetes distros are not interchangeable. The
curriculum's recommendation:

| Distro | Use it when | Why |
|---|---|---|
| **k3s** | A real VM (DO droplet, EC2, Multipass, WSL distro, bare metal) | Single binary, ~50MB; production-shaped; the same distro powers edge fleets at real scale; ships sane defaults (Traefik, klipper-lb, local-path provisioner, metrics-server, CoreDNS). |
| **k3d** | A laptop, ephemeral lab | k3s running inside Docker containers; one node or many; spin up / tear down in seconds; same defaults as k3s. |
| Docker Desktop K8s | Skip | Slower, fewer of the things-that-bite-you in prod show up locally, opaque networking, ties you to Docker Desktop. |
| minikube | Skip | Older mental model, less production-shaped, default ingress is nginx (yet another component to learn). |
| kind | Edge cases only | Production-shaped, fast — but no built-in LB / Ingress / metrics; you're assembling pieces. Use it for upstream-K8s testing; skip for everyday work. |
| EKS / AKS / GKE local | Skip for dev | Real clusters are slow to spin up, cost money, and aren't reproducible per-developer. |

The case for k3s/k3d isn't speed alone:

1. **The defaults are correct.** Traefik (ingress), klipper-lb
   (LoadBalancer Services with no cloud), local-path
   (PersistentVolumes), metrics-server (HPA + `kubectl top`),
   CoreDNS. All of these need to be assembled by hand on
   minikube/kind.
2. **The networking is real.** A `type: LoadBalancer` Service
   binds an actual port on the node, addressable from your
   host. No port-forward (see C16 / Module 5).
3. **The same distro runs in prod.** k3s on a Raspberry Pi, k3s
   on a 64-core edge node, k3s on a fleet of 800+ Domino's
   stores. The thing on your laptop is the thing in production.
4. **The footprint is small.** ~50MB binary, ~200MB RAM idle.
   Compare to Docker Desktop K8s (~3GB image, ~2GB RAM idle).

#### Example

Spinning up a k3d cluster sized for the curriculum (movies +
prometheus + grafana + traefik + room for experiments):

```bash
k3d cluster create lab \
    --servers 1 \
    --agents 2 \
    --port 8080:80@loadbalancer \
    --port 8443:443@loadbalancer \
    --k3s-arg "--disable=traefik@server:*" \
    --wait

# We disabled the bundled Traefik so we can install a current
# Traefik via its Helm chart or manifests at the version we want.
```

Or, the simpler "just give me a cluster" version that keeps the
bundled defaults:

```bash
k3d cluster create lab --servers 1 --agents 1 --wait
kubectl cluster-info
kubectl get nodes
```

Spinning up k3s on a Multipass VM or DO droplet:

```bash
curl -sfL https://get.k3s.io | sh -
sudo cat /etc/rancher/k3s/k3s.yaml   # kubeconfig
```

That's it. One curl. Production-shaped cluster in 30 seconds.

#### Lab

1. Install `k3d` (`brew install k3d` / your package manager).
   `k3d cluster create lab --servers 1 --agents 1 --wait`.
   Time the bring-up.
2. `kubectl get nodes` — confirm one server + one agent.
   `kubectl get pods -A` — read what's running by default
   (CoreDNS, metrics-server, Traefik, local-path).
3. `k3d node list` — note these are containers on your Docker
   host. `docker ps | grep k3d` — same.
4. **Compare to Docker Desktop K8s.** If you have it enabled,
   `kubectl config use-context docker-desktop`. Time how long
   *that* takes to enable. Note RAM use before/after.
5. `k3d cluster delete lab && k3d cluster create lab2 ...`.
   Note that spinning a fresh cluster takes ~10 seconds.
6. **If you're on a VM (Multipass / DO droplet / WSL)**, install
   k3s directly: `curl -sfL https://get.k3s.io | sh -`. Compare
   the experience.

#### Knowledge check

1. k3s vs k3d — when do you use each?
2. Three concrete defaults k3s ships that you'd have to install
   yourself on kind.
3. Why is "the thing on my laptop is the thing in production"
   an operational property worth optimizing for?
4. minikube and Docker Desktop K8s — name one concrete
   property of each that makes the curriculum recommend
   against them.
5. A 64-core production edge node and a Raspberry Pi can both
   run k3s. What does that tell you about the design?

---

### Module 3 — kubectl fluency: the 80% and the aliases (C4, C5, C6, C7)

> **If you are typing `kubectl` in full, every hour of work
> has a tax on it.** Fluency in `kubectl` + `k9s` is the
> highest-leverage one-hour investment in this whole curriculum.

#### Concept

The 80% of `kubectl`:

| Verb | What it does | Read it like |
|---|---|---|
| `get` | List objects | `ls` |
| `describe` | Detailed view of one object incl. events | `cat` + event log |
| `logs` | Container logs | `tail` |
| `exec` | Run a command in a container | `ssh + run` |
| `apply` | Create or update from manifest | `make install` |
| `delete` | Remove an object | `rm` |
| `edit` | Open the live object in `$EDITOR` and re-apply on save | last-resort surgery |
| `port-forward` | Local port → pod port (see C16 — usually wrong answer) | `ssh -L` |
| `cp` | Copy files in/out of a pod | `scp` |
| `top` | Resource usage (needs metrics-server) | `top` |
| `rollout` | Manage Deployment rollouts (status, history, undo, restart) | revision control |
| `scale` | Adjust replicas | `--replicas=N` |
| `wait` | Block until a condition | script-friendly |

The alias set that makes every workday faster (from
[study-guide-local-platform.md](study-guide-local-platform.md)
Module 6, restated here because it's the C-domain piece):

```bash
alias k=kubectl
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgd='kubectl get deploy'
alias kgn='kubectl get ns'
alias kgno='kubectl get nodes'
alias kctx='kubectl config use-context'
alias kns='kubectl config set-context --current --namespace'
alias kdp='kubectl describe pod'
alias kl='kubectl logs'
alias klf='kubectl logs -f'
alias ke='kubectl exec -it'
```

Plus the **context/namespace switching** tooling — `kubectx` and
`kubens` (single binaries; `brew install kubectx`,
`apt install kubectx`, or `go install
github.com/ahmetb/kubectx/cmd/kubectx@latest`):

```bash
kubectx           # list contexts; arrow-key picker if fzf installed
kubectx prod      # switch to prod cluster
kubens            # list namespaces; picker
kubens movies     # switch default namespace
```

**Strong opinion:** if you're working with > 1 cluster, you need
context-switching that *makes the current cluster visible in your
prompt*. The wrong cluster + the wrong namespace + a script that
assumes "I know what I'm pointing at" is how production gets
deleted. Pair `kubectx`/`kubens` with a prompt that shows
`<cluster>:<namespace>` always.

The **shell completion** that ships with `kubectl` is essential:

```bash
# zsh
source <(kubectl completion zsh)
# and add this for the `k` alias:
complete -F __start_kubectl k
```

Now `k get pods -n <TAB>` completes namespaces; `k logs <TAB>`
completes pod names. Same wins as the broader shell baseline.

**k9s** — the terminal UI:

```bash
brew install derailed/k9s/k9s
# or: go install github.com/derailed/k9s@latest
```

Run `k9s`. Navigate with arrow keys, `:pods`, `:svc`, `:deploy`,
`:ns` to jump to a resource. `l` for logs, `d` for describe,
`s` for shell, `<ctrl-k>` to kill, `<ctrl-d>` to delete. Worth
an hour to learn; an hour returned every day after.

#### Example

A normal debugging sequence using the alias set:

```bash
# Set the cluster + namespace; confirm.
kctx lab
kns movies

# What's running?
kgp                                  # kubectl get pods
kgp -o wide                          # + node, IP, etc.

# That pod is CrashLoopBackOff. Describe it.
kdp movies-api-7c8b9d-xyz12          # kubectl describe pod
# (read the Events section at the bottom — it almost always tells you)

# Logs.
kl movies-api-7c8b9d-xyz12           # current
klf movies-api-7c8b9d-xyz12          # follow
kl movies-api-7c8b9d-xyz12 -p        # previous (after a crash)

# Get a shell in (if not distroless).
ke movies-api-7c8b9d-xyz12 -- sh

# Restart all pods of a deployment.
k rollout restart deploy/movies-api

# Watch the rollout.
k rollout status deploy/movies-api
```

In k9s, the same sequence is: arrow-key to the pod, `l` for
logs, `s` for shell, `r` to restart the parent deployment.

#### Lab

1. Install: `kubectl` (you have it), `kubectx`, `kubens`, `k9s`.
   Set up the aliases in your `~/.zshrc`. Source the completion
   for both `kubectl` and the `k` alias.
2. On your k3d/k3s cluster: practice the 80%. `kgp -A`,
   `kgd -A`, `kgs -A`. Pick a random pod; `kdp <name>`,
   `kl <name>`, `ke <name> -- sh` (if the image has a shell).
3. **The context-switching drill.** Add a second cluster context
   (k3d cluster create lab2). `kubectx` between them. Confirm
   your prompt shows the current context. *Try to delete
   something from the wrong cluster on purpose* (a scratch
   namespace, nothing important). Note how easy the mistake is
   without a visible prompt.
4. **k9s walkthrough.** Run `k9s`. Tab to namespaces. Select
   `movies`. View pods. Select one; `l` for logs; `s` for
   shell; `<esc>` back. Type `:svc<enter>`; view services.
   Type `:deploy<enter>`; view deployments. Type `:q` to quit.
5. **The completion test.** `k logs <TAB>` should list pod
   names in the current namespace. If it doesn't, completion
   isn't wired up; fix before moving on.
6. **Read your shell rc.** `cat ~/.zshrc | grep -E
   'kubectl|kubectx|k9s|alias k='`. Should have all of them.

#### Knowledge check

1. Five `kubectl` verbs you use 80% of the time.
2. The `k=kubectl` alias is the obvious one. What problem does
   `kubectx`/`kubens` solve that aliases alone don't?
3. Why does the curriculum recommend making the current context
   + namespace visible in your shell prompt?
4. `kubectl logs <pod> -p` — when do you use the `-p` flag?
5. `k9s` is a TUI. Why is "yet another way to run `kubectl`"
   worth a separate row in the inventory?
6. Your `k get pods -n <TAB>` does not complete namespaces.
   What's broken?

---

### Module 4 — Pods, ReplicaSets, Deployments: who owns whom (C8)

> **Three nested abstractions; one chain of ownership.** Most
> "the deployment didn't update" stories trace to misunderstanding
> the chain.

#### Concept

The ownership chain, smallest to largest:

```
Container (process)
   └─ Pod (one or more containers, shared network + storage namespace)
        └─ ReplicaSet (maintains N identical pods)
             └─ Deployment (manages rolling updates between ReplicaSets)
```

- **Pod** — the smallest deployable unit. One or more containers
  that share a network namespace (they `localhost` each other)
  and storage (mounted volumes are shared). **Pods are
  ephemeral**; treat them as disposable.
- **ReplicaSet** — keeps N replicas of a pod template running.
  If a pod dies, the RS creates a replacement. You almost never
  create one by hand.
- **Deployment** — manages rolling updates. When the pod
  template changes, the Deployment creates a *new* ReplicaSet
  with the new template, scales it up while scaling the old one
  down. The Deployment doesn't own pods directly; it owns
  ReplicaSets, which own pods.

What this means operationally:

- `kubectl delete pod <name>` → the ReplicaSet creates a
  replacement immediately. To "really" remove it, delete the
  Deployment (or scale the Deployment to 0).
- `kubectl edit deployment <name>` → the Deployment's pod
  template changes → a new ReplicaSet is created → a rolling
  update happens.
- `kubectl rollout history deployment <name>` → list of past
  ReplicaSets (each ReplicaSet is a revision).
- `kubectl rollout undo deployment <name>` → scale the previous
  RS back up, scale the current one down.

Three other abstractions worth knowing now but not the
spec-floor common case:

- **StatefulSet** — like a Deployment but pods have stable
  identities (`mypod-0`, `mypod-1`) and stable storage. For
  databases, message brokers, anything that cares which pod it
  is.
- **DaemonSet** — one pod per node. For per-node agents (log
  shippers, network proxies, metrics-node-exporter).
- **Job / CronJob** — run-to-completion (one-shot or
  scheduled). For migrations, batch jobs, the
  movies-bartr `webv` load generator.

#### Example

Movies-bartr's Deployment, abbreviated to the structural fields
(securityContext detail is in
[study-guide-security.md](study-guide-security.md) Module 1):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: movies-api
  namespace: movies
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: movies-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app.kubernetes.io/name: movies-api
    spec:
      containers:
        - name: movies-api
          image: movies-api:1.0.0
          ports:
            - { name: http, containerPort: 8080, protocol: TCP }
          # ... probes, resources, securityContext per other guides
```

The selector → template label chain is *load-bearing*: the
Deployment finds its ReplicaSet's pods by matching
`spec.selector` against pod labels. If the selector doesn't
match the template's labels, the Deployment can't track its own
pods. (Most common mistake: editing one without the other.)

`maxSurge: 1, maxUnavailable: 0` is the safe default for a
single-replica service — bring up the new pod first, then take
the old one down. Spec-shaped.

#### Lab

1. `kubectl apply -f deploy/movies/base/deployment.yaml -n
   movies`. (Or `kustomize build deploy/movies/base | kubectl
   apply -f -` for the full set.)
2. `kgp -n movies -o wide`. Note the pod's name format:
   `<deployment>-<rs-hash>-<pod-hash>`.
3. `kubectl get rs -n movies`. See the ReplicaSet. Its name is
   `<deployment>-<rs-hash>` — same hash as in the pod name.
4. **Test ownership.** `kubectl delete pod
   movies-api-<...>-<...> -n movies`. Watch a new pod appear
   within seconds. Same RS, new pod hash.
5. **Trigger a rollout.** `kubectl set image deploy/movies-api
   movies-api=movies-api:1.0.1 -n movies` (or edit the image
   tag in the manifest and re-apply). `kgp -n movies -w` to
   watch.
6. `kubectl rollout history deploy/movies-api -n movies`. See
   two revisions. `kubectl rollout undo deploy/movies-api -n
   movies`. Watch the previous revision come back.
7. **The selector-mismatch trap.** Edit the Deployment to
   change `spec.selector.matchLabels` *without* changing the
   template labels. Apply. Read the error. Restore.

#### Knowledge check

1. The four-layer ownership chain — name it from container up
   to Deployment.
2. You `kubectl delete pod`. Why doesn't it stay deleted?
3. `kubectl rollout history` shows revisions. What are
   revisions, in terms of underlying objects?
4. `maxSurge` and `maxUnavailable` — name a scenario where each
   matters.
5. The Deployment's `spec.selector` and `spec.template.metadata.labels` —
   why must they match, and what happens if they don't?
6. StatefulSet, DaemonSet, Job — name one workload that fits
   each.

---

### Module 5 — Services, DNS, and the LoadBalancer answer (C9, C16)

> **`kubectl port-forward` is a smell.** Curriculum-strong
> opinion. Real ports via `type: LoadBalancer` are the path that
> matches production.

#### Concept

A **Service** is a stable network identity for a set of pods.
Pods are ephemeral; their IPs change; you can't address them
directly. The Service gives you:

- A stable **DNS name** (`<service>.<namespace>.svc.cluster.local`).
- A stable **virtual IP** (the ClusterIP).
- Automatic **load balancing** across matching pods (via
  `kube-proxy` / IPVS / eBPF, depending on the cluster).

Three (and a half) Service types:

| Type | What it does | Use when |
|---|---|---|
| **ClusterIP** | Stable in-cluster VIP. Not reachable from outside. | The default. In-cluster service-to-service. |
| **NodePort** | Same as ClusterIP, plus exposes the port on every node at a high port (30000-32767). | Legacy / quick demos / when you can't run a LB. |
| **LoadBalancer** | Same as NodePort, plus asks the cloud (or `klipper-lb` on k3s) for an external IP / real port. | **What you want for any service the operator touches locally.** |
| **ExternalName** | DNS CNAME to an external host. No proxying. | In-cluster alias for an external API. |

Plus **headless Services** (`clusterIP: None`) — returns the pod
IPs directly via DNS, used by StatefulSets and for direct pod
addressing.

#### The curriculum opinion on ingress

**Default to `type: LoadBalancer` for any service the operator
will interact with from their host.** On k3s/k3d, the built-in
`klipper-lb` watches LoadBalancer Services and binds the
requested port on the node (which is your laptop, for k3d).
`grafana :3000`, `prometheus :9090`, `movies-api :8080` —
all real ports, addressable as `http://localhost:<port>`.

Why this matters:

1. **No `kubectl port-forward`** — that command taxes every lab
   with terminal management, dies on the first network blip,
   and teaches a habit that doesn't transfer to production.
2. **You actually test the Service.** A `port-forward` skips the
   Service entirely and connects to the pod. The Service might
   be misconfigured and you'd never know.
3. **The mental model matches prod.** In production, your service
   *is* reachable through a real LB / Ingress at a real port.
   Your local environment should be the same shape.

The DNS form to remember:

- Same namespace: `movies-api` or `movies-api:8080`.
- Other namespace: `movies-api.movies` or
  `movies-api.movies.svc.cluster.local`.

#### Example

Movies-bartr's base Service (ClusterIP) plus a dev overlay that
flips it to LoadBalancer (the pattern from
[study-guide-kustomize.md](study-guide-kustomize.md) Module 7):

```yaml
# deploy/movies/base/service.yaml — ClusterIP, in-cluster only.
apiVersion: v1
kind: Service
metadata:
  name: movies-api
  namespace: movies
  labels:
    app.kubernetes.io/name: movies-api
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: movies-api
  ports:
    - { name: http, port: 8080, targetPort: http, protocol: TCP }
```

```yaml
# deploy/movies/overlays/dev/service-lb.yaml — flip to LoadBalancer.
apiVersion: v1
kind: Service
metadata:
  name: movies-api
spec:
  type: LoadBalancer
  ports:
    - { name: http, port: 8080, targetPort: http, protocol: TCP }
```

After applying the dev overlay against k3d, `kubectl get svc -n
movies` shows `EXTERNAL-IP: <localhost or node IP>`, and
`curl http://localhost:8080/healthz` works directly. No
port-forward.

Calling from another pod (in-cluster):

```bash
# from any pod, any namespace:
curl http://movies-api.movies:8080/healthz
```

#### Lab

1. Apply the movies-bartr base. `kgs -n movies` — see the
   ClusterIP Service.
2. **In-cluster DNS test.** `kubectl run -n movies --rm -it
   debug --image=nicolaka/netshoot -- sh`. Inside the debug
   pod: `nslookup movies-api`; `curl
   http://movies-api:8080/healthz`. Works.
3. **The `port-forward` baseline (what we're going to stop
   doing).** `kubectl port-forward -n movies svc/movies-api
   8080:8080`. In another shell: `curl
   http://localhost:8080/healthz`. Works — but you've tied up a
   terminal and skipped the Service.
4. **The LoadBalancer way.** `kubectl patch svc movies-api -n
   movies -p '{"spec": {"type": "LoadBalancer"}}'` (or apply
   the dev overlay). `kgs -n movies` — note the EXTERNAL-IP.
   `curl http://localhost:8080/healthz`. Works *without* the
   port-forward terminal.
5. **Demonstrate the failure mode it catches.** Edit the
   Service to point to a bogus selector (`app.kubernetes.io/name:
   does-not-exist`). Apply. The port-forward would still work
   (it talks directly to the pod). The LoadBalancer correctly
   returns connection refused. Restore.
6. **DNS exercise.** From the debug pod: `nslookup
   movies-api.movies.svc.cluster.local` — fully qualified.
   `nslookup movies-api.movies` — short form. `nslookup
   movies-api` — works only because the debug pod is *in* the
   movies namespace.

#### Knowledge check

1. Four Service types — when do you use each?
2. The full DNS form is `<svc>.<ns>.svc.cluster.local`. Why does
   the short form `<svc>` work from same-namespace, and what's
   the "search list" mechanism behind it?
3. `port-forward` works against pods *or* Services. Which does
   it talk to by default? Why does that matter for catching
   Service-config bugs?
4. The curriculum's opinion: default to `type: LoadBalancer` on
   k3s/k3d for anything the operator touches. Name three
   concrete things this buys you.
5. `klipper-lb` is k3s's built-in LB. On a cloud-managed K8s
   (EKS/AKS/GKE), what plays the same role and how does that
   affect cost?
6. Headless Service (`clusterIP: None`) — when do you reach
   for it?

---

### Module 6 — Probes: liveness, readiness, startup (C10)

> **The most-misconfigured controls in Kubernetes.** Get them
> wrong and your pod crash-loops, your rollouts deadlock, or
> your service is "up" while returning 500s.

#### Concept

Three probes, three jobs:

| Probe | Question it answers | What happens on failure |
|---|---|---|
| **Liveness** | Is the process *alive and healthy*? | The kubelet kills the container; the RS replaces it. |
| **Readiness** | Is the process *ready to serve traffic*? | The endpoint is removed from the Service; traffic stops. |
| **Startup** | Is the process *finished starting up*? | While running, liveness/readiness are paused. After timeout, treated as liveness failure. |

The discipline:

- **Readiness ≠ Liveness.** A pod can be alive (process running)
  but not ready (database connection not warmed up). Conflating
  the two is the single most common probe mistake — under load
  spikes, traffic doesn't briefly stop hitting the pod; the
  pod gets killed entirely.
- **Readiness should be *cheap and local***. Don't make
  /readyz hit your downstream database — a 5-second DB blip
  becomes a pod removal. The right pattern: /readyz checks
  in-process state (data loaded? worker started?).
- **Liveness should be *cheap and conservative***. A liveness
  endpoint that ever returns 503 under normal load is going
  to kill the pod under any load spike. A simple "the process
  is responding to HTTP" check is usually right.
- **Startup probes** exist for slow-starting apps (loading
  large models, warming caches). They prevent the liveness
  probe from killing the pod during legitimately-slow startup.

Probe configuration:

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: http
  initialDelaySeconds: 2     # wait this long before the first probe
  periodSeconds: 10          # probe every N seconds
  timeoutSeconds: 2          # fail if no response within N seconds
  failureThreshold: 3        # after N consecutive failures, restart

readinessProbe:
  httpGet:
    path: /readyz
    port: http
  initialDelaySeconds: 1
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 1        # readiness can be more sensitive
```

Other probe types: `tcpSocket` (just "did the port accept a
connection?"), `exec` (run a command in the container, check
exit code), `grpc` (since 1.24+, native gRPC probe). HTTP is
the default for HTTP services and the spec's expectation.

#### Example

Movies-bartr's probes (from `deploy/movies/base/deployment.yaml`):

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: http
  initialDelaySeconds: 2
  periodSeconds: 10
  timeoutSeconds: 2
readinessProbe:
  httpGet:
    path: /readyz
    port: http
  initialDelaySeconds: 1
  periodSeconds: 5
  timeoutSeconds: 2
```

In the Go code (per
[study-guide-go.md](study-guide-go.md)):

- `/healthz` returns 200 unconditionally if the HTTP server is
  serving. The process is alive.
- `/readyz` returns 200 only if the movies catalog has loaded
  from `/data`. Returns 503 until that's done.

This is the right split. A failed catalog load = "don't send
traffic, but don't kill me yet" (readiness fails, liveness
passes). A wedged HTTP server = "kill and restart" (liveness
fails).

#### Lab

1. Deploy movies-bartr. `kgp -n movies`. Confirm `READY 1/1`.
2. `kubectl describe pod <name> -n movies` and find the
   `Readiness`, `Liveness` lines. Read the config.
3. **Break liveness.** Edit the Deployment to set
   `livenessProbe.httpGet.path: /not-a-real-path`. Apply.
   Watch: pod becomes unhealthy, kubelet restarts it,
   `RESTARTS` counter climbs.
4. Restore. **Break readiness.** Set
   `readinessProbe.httpGet.path: /not-a-real-path`. Apply.
   Watch: pod stays running (liveness OK) but `READY 0/1` —
   it's removed from the Service. `kgs -n movies` →
   `kubectl get endpoints movies-api -n movies` shows no
   endpoints.
5. Restore.
6. **The startup-probe scenario.** Add a `startupProbe` that
   gives the app 60 seconds to start. Set `failureThreshold:
   30, periodSeconds: 2`. This is the right pattern for an app
   that legitimately takes 30+ seconds to come up.
7. **Demonstrate why readiness must be local.** Add to
   `/readyz` a check that pings an external service (e.g.,
   `https://api.github.com/status`). Run a `tc` rule to block
   that traffic. Watch your pod become un-ready even though
   it's fine. *This is what "readiness should be cheap and
   local" prevents.*

#### Knowledge check

1. Three probes — name them and what action the kubelet takes
   on failure of each.
2. The "readiness ≠ liveness" rule — what's the failure mode
   when you conflate them?
3. Why should readiness be local? What downstream-coupling
   mistake does this discipline prevent?
4. Startup probe — what specific problem does it solve that
   the other two don't?
5. `failureThreshold: 3` on liveness vs `failureThreshold: 1`
   on readiness — what's the rationale for the asymmetry?

---

### Module 7 — Resources: requests, limits, QoS (C11)

> **The numbers fix what the cluster gives you.** Requests
> drive scheduling; limits drive enforcement; the ratio drives
> QoS class; QoS class drives eviction order.

#### Concept

Two numbers per resource (CPU, memory):

- **Request** — the amount the scheduler reserves for the
  container. Used to find a node with enough free capacity.
  The container is guaranteed at least this much.
- **Limit** — the upper bound. Enforced by cgroups.
  - **CPU limit**: container is *throttled* at the limit
    (latency spikes; no kill).
  - **Memory limit**: container is *OOMKilled* if it tries
    to exceed (instant kill, RS restarts).

The three QoS classes K8s assigns automatically:

| Class | When | Eviction order |
|---|---|---|
| **Guaranteed** | requests == limits for *every* container, *every* resource | Last to be evicted |
| **Burstable** | At least one request set; not Guaranteed | Middle |
| **BestEffort** | No requests or limits set on anything | First to be evicted under node pressure |

The spec floor: **always set requests and limits.** A
BestEffort pod is one node-pressure event away from being
killed.

The two failure modes named in the inventory:

- **OOMKilled** — memory limit exceeded. `kubectl describe pod`
  → `Last State: Terminated, Reason: OOMKilled`. Fix: raise
  the limit or fix the leak.
- **Evicted** — node ran out of resources (memory, disk); the
  kubelet kicked pods off, lowest QoS first. `kubectl get
  pods` shows the evicted pods stuck in `Evicted` status
  forever (clean them up with `kubectl delete pod --field-selector
  status.phase=Failed`).

A subtle one:

- **CPU throttling** — your service "slows down under load."
  `kubectl top pod` shows CPU at the limit; latency degrades;
  no log entry, no event. Hard to see without metrics. The
  signal: p99 latency climbs as load climbs but error rate
  doesn't.

#### Example

Movies-bartr's resources (from `deployment.yaml`):

```yaml
resources:
  requests:
    cpu: "100m"        # 0.1 cores guaranteed
    memory: "128Mi"
  limits:
    cpu: "500m"        # max 0.5 cores
    memory: "512Mi"
```

This is **Burstable** (requests < limits). Reasonable for a
spec-shaped service: it can burst above its baseline when busy,
but is guaranteed 100m / 128Mi. The 4× headroom on memory is
deliberate — Go's GC has spikes and the JSON catalog adds
fixed overhead.

A Guaranteed equivalent would be:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

— same request and limit on every resource. Use Guaranteed for
latency-sensitive workloads where eviction would be very
expensive.

#### Lab

1. Apply movies-bartr. `kubectl describe pod <name> -n movies
   | grep -A5 'QoS\|Requests\|Limits'`. Confirm Burstable.
2. **Force OOMKilled.** Set `memory.limit: 16Mi`. Apply.
   Watch: the Go runtime tries to allocate, exceeds 16MiB,
   gets killed. `RESTARTS` climbs. `kubectl describe pod` →
   `OOMKilled`. Restore.
3. **Force CPU throttling.** Set `cpu.limit: 50m`. Apply. Run
   a load test (`webv`, `hey`, `vegeta` — whatever's
   handy). Watch latency degrade while error rate stays at 0.
   `kubectl top pod` shows CPU pinned at 50m. *No log entry.*
   Restore.
4. **Demonstrate eviction.** On a small cluster: deploy a
   BestEffort pod (no resources block) and a Guaranteed pod.
   Stress the node with `stress-ng --vm 1 --vm-bytes 80%`.
   Watch which gets evicted first.
5. `kubectl top nodes` and `kubectl top pods -A`. Get used to
   reading these.
6. **The asymmetry lab.** Set `requests` low and `limits` high
   (e.g., 100m/2000m, 128Mi/2Gi). What QoS class? Now set
   `requests` only. What QoS class?

#### Knowledge check

1. Request vs limit — what's the difference, semantically and
   in cgroup terms?
2. Three QoS classes — when does each apply, and what's the
   eviction order?
3. OOMKilled vs Evicted — different events. Which is "I
   exceeded my limit" and which is "the node didn't have
   enough"?
4. CPU throttling is invisible in logs. What's the metric to
   watch?
5. Burstable vs Guaranteed for a latency-sensitive service —
   what's the trade-off?

---

### Module 8 — ConfigMaps and Secrets (C12)

> **Secrets are not a security boundary on their own.**
> Base64 isn't encryption. The protection is the RBAC around
> them and what your cluster does for encryption-at-rest.

#### Concept

Two K8s objects for configuration:

- **ConfigMap** — non-sensitive key/value config. Stored as
  plaintext in etcd. Use for: feature flags, log levels,
  config-file contents.
- **Secret** — "sensitive" key/value. Stored **base64-encoded**
  in etcd (which is *not* encryption). Use for: API keys,
  TLS certs, database passwords.

The bits that matter:

1. **Encryption at rest is opt-in.** On a vanilla K8s, etcd
   stores Secrets base64-encoded but not encrypted. Managed
   K8s (EKS, AKS, GKE) usually enables encryption at rest by
   default. **Check before assuming.**
2. **RBAC is the real boundary.** A user with `get secrets`
   permission can read every Secret in their scope, regardless
   of "encryption." Lock down secret access via RBAC (Module
   10).
3. **ConfigMap for "non-sensitive" config that turns sensitive
   later is a leak waiting to happen.** A Grafana admin
   password put in a ConfigMap because "it's just a config" is
   a credential exposed to anyone with `get configmaps`.
4. **Secrets-in-repo is the spec §13 forbidden zone.** Covered
   in [study-guide-gitops.md](study-guide-gitops.md) Module 6
   (SOPS / Sealed Secrets / ESO) and
   [study-guide-security.md](study-guide-security.md) Module 4
   (runtime injection patterns). Not repeated here.

Three injection patterns at runtime (recap from
[study-guide-security.md](study-guide-security.md) Module 4):

- `envFrom: - configMapRef: { name: X }` or `secretRef`.
- `env: - valueFrom: { configMapKeyRef / secretKeyRef }`.
- `volumeMounts` + `volumes` projecting a CM or Secret as
  files.

Pattern selection: env-injection for simple key/value;
volume-mount for files (TLS certs, kubeconfigs); for secrets
specifically, prefer volume-mount (lower leak risk to other
processes).

#### Example

A ConfigMap for non-sensitive runtime config:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: movies-config
  namespace: movies
data:
  LOG_LEVEL: "info"
  CACHE_TTL_SECONDS: "300"
```

Consumed by the Deployment:

```yaml
spec:
  containers:
    - name: movies-api
      envFrom:
        - configMapRef:
            name: movies-config
```

A Secret for hypothetical credentials:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: movies-secrets
  namespace: movies
type: Opaque
stringData:
  DB_PASSWORD: "hunter2"   # stringData is base64-encoded for you
```

```yaml
spec:
  containers:
    - name: movies-api
      env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: movies-secrets
              key: DB_PASSWORD
```

The spec-shaped movies-bartr service has no Secrets (no DB,
no external credentials, no TLS terminated at the app). The
example is here for the pattern; in production, the actual
secret comes from ESO or SOPS, not a literal `stringData`.

#### Lab

1. Create a scratch ConfigMap and Secret in the `movies` ns.
   Apply both.
2. **Read them.** `kubectl get cm movies-config -o yaml`.
   `kubectl get secret movies-secrets -o yaml`. Note the
   Secret's `data` block is base64.
3. **Decode the Secret.** `kubectl get secret movies-secrets
   -o jsonpath='{.data.DB_PASSWORD}' | base64 -d`. *Not
   encryption.*
4. Mount the Secret into the Deployment via `envFrom`. Roll
   the pod. `kubectl exec -it <pod> -- env | grep DB_PASSWORD`
   (assumes the image has a shell — if it's distroless, use a
   debug ephemeral container).
5. **Demonstrate the `/proc/1/environ` leak** (from
   [study-guide-security.md](study-guide-security.md) Module
   4). Confirm the password is visible there.
6. **Switch to volume-mount.** Mount the Secret as a file at
   `/etc/movies/secrets/`. Roll. Confirm the file is there
   with mode 0644 (or 0400 if you set `defaultMode`).
7. Clean up.

#### Knowledge check

1. Base64 is not encryption. Why is "Secret" still useful as a
   K8s object type vs just using ConfigMaps for everything?
2. Encryption at rest for etcd is opt-in on vanilla K8s. How
   do you check whether *your* cluster has it?
3. RBAC is the real boundary for Secret access. What's the
   smallest scope you'd give a Deployment that only needs to
   read one Secret?
4. The "ConfigMap for non-sensitive config that turns sensitive
   later" trap — example?
5. envFrom vs valueFrom vs volume-mount for Secrets — when do
   you reach for each? (Cross-reference security guide M4.)

---

### Module 9 — RBAC basics: ServiceAccount, Role, RoleBinding (C13)

> **RBAC is "who can do what to which objects in which
> scope."** Four nouns, one decision matrix.

#### Concept

The four primitives:

| Object | What it represents | Scope |
|---|---|---|
| **ServiceAccount (SA)** | An identity for a pod / process | Namespace |
| **Role** | A set of allowed verbs on a set of resources | Namespace |
| **ClusterRole** | Same, cluster-wide | Cluster |
| **RoleBinding** | Binds a Role to a Subject (SA, User, Group) | Namespace |
| **ClusterRoleBinding** | Binds a ClusterRole to a Subject | Cluster |

The cross-product matters:

- A `RoleBinding` can bind either a `Role` *or* a `ClusterRole`
  (in which case the ClusterRole's permissions are limited to
  the binding's namespace).
- A `ClusterRoleBinding` only binds `ClusterRole`.

**Default deny.** A pod with no explicit RBAC and the default
ServiceAccount can do *almost nothing* against the K8s API
(except things specifically allowed to `system:authenticated`,
which is very little).

**The two common scenarios:**

1. **A controller / operator needs cluster API access.** Example:
   the Prometheus Operator needs to watch ServiceMonitors
   cluster-wide. Pattern: a dedicated SA + ClusterRole +
   ClusterRoleBinding.
2. **A workload needs *no* API access.** Example: movies-bartr.
   Pattern: `automountServiceAccountToken: false` at the pod
   level. No token mounted; no API access possible. This is the
   spec floor — covered in
   [study-guide-security.md](study-guide-security.md) Module 1.

**Avoid:**

- `cluster-admin` for workloads. The "I'll narrow it later"
  ClusterRoleBinding that never gets narrowed.
- Sharing one SA across many workloads. Per-workload SAs scope
  blast radius.

#### Example

A ServiceAccount + Role + RoleBinding for a workload that
needs to read pods in its own namespace (rare, but a clean
example):

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-reader
  namespace: movies
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: movies
rules:
  - apiGroups: [""]                  # core/v1 API group
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader
  namespace: movies
subjects:
  - kind: ServiceAccount
    name: pod-reader
    namespace: movies
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Bound to a Deployment via `spec.template.spec.serviceAccountName:
pod-reader`.

What you *don't* want, and the Prometheus Operator stack does
correctly:

```yaml
# A ClusterRole because watching ServiceMonitors is cluster-wide.
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
  - apiGroups: ["monitoring.coreos.com"]
    resources: ["servicemonitors", "podmonitors", "probes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["services", "endpoints", "pods"]
    verbs: ["get", "list", "watch"]
```

Narrow, named, auditable. *Not* `cluster-admin`.

#### Lab

1. `kubectl auth can-i --list -n movies --as=system:serviceaccount:movies:default`.
   Read what the default SA in movies *can't* do. Most things.
2. Create the `pod-reader` SA + Role + RoleBinding above.
   Apply.
3. `kubectl auth can-i list pods -n movies
   --as=system:serviceaccount:movies:pod-reader`. → `yes`.
   `kubectl auth can-i list deployments -n movies
   --as=system:serviceaccount:movies:pod-reader`. → `no`.
4. **Patch the movies Deployment** to use the new SA
   temporarily. From inside a pod (use a debug ephemeral
   container if distroless), `curl
   https://kubernetes.default/api/v1/namespaces/movies/pods
   -H "Authorization: Bearer
   $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)"
   --cacert
   /var/run/secrets/kubernetes.io/serviceaccount/ca.crt`. Read
   pods successfully. Try `/deployments` — denied.
5. **Restore movies to `automountServiceAccountToken: false`**
   (its spec-floor state). Now even with code to read the API,
   no token exists in the pod.
6. **Audit a real RBAC binding.** `kubectl get
   clusterrolebindings -o wide | head`. Find any binding to
   `cluster-admin`. Read the subjects. Note who has root.

#### Knowledge check

1. ServiceAccount, Role, ClusterRole, RoleBinding,
   ClusterRoleBinding — five primitives. Which scopes are
   namespace vs cluster?
2. A RoleBinding can bind a ClusterRole. What does that mean
   in terms of effective permissions?
3. The spec-floor pattern for a workload that doesn't talk to
   the K8s API. Two YAML lines.
4. Per-workload SA vs one-SA-per-namespace — what's the
   blast-radius argument?
5. `cluster-admin` for a Deployment is almost always wrong.
   Name a concrete case where it's been used (in your
   experience or what you've seen documented) and how you'd
   narrow it.

---

### Module 10 — Custom Resources (CRDs) (C14)

> **CRDs make K8s extensible.** Every operator, every
> controller-pattern tool, every "platform on K8s" is built
> on them. ServiceMonitor (from the Prometheus Operator) is
> the example the spec floor needs.

#### Concept

A **CustomResourceDefinition (CRD)** registers a new
*resource type* with the K8s API server. Once registered, the
new type behaves *exactly like a built-in*:

- `kubectl get <type>` works.
- `kubectl describe / apply / delete` work.
- RBAC works (you can grant `get servicemonitors` to a Role).
- Watching for changes works.

What CRDs *don't* do: act on themselves. The CRD defines the
schema; a separate **controller** (often packaged as an
"operator") watches the API for objects of that type and
*does something* in response.

The pattern:

```
CRD: defines a new schema (e.g., kind: ServiceMonitor)
   ↓
Operator (controller): watches for objects of that kind
   ↓
The operator does work in response (e.g., re-renders the
Prometheus scrape config and reloads Prometheus)
```

Examples worth knowing:

- **Prometheus Operator** → `Prometheus`, `Alertmanager`,
  `ServiceMonitor`, `PodMonitor`, `PrometheusRule`,
  `ThanosRuler`. The spec's observability stack runs on
  these.
- **cert-manager** → `Certificate`, `Issuer`, `ClusterIssuer`.
  TLS-cert lifecycle automation.
- **Traefik** → `IngressRoute`, `Middleware`, `TLSStore`.
  Traefik's CRD-based config (alternative to native
  Ingress).
- **Flux** → `GitRepository`, `Kustomization`,
  `HelmRelease` (from
  [study-guide-gitops.md](study-guide-gitops.md) Module 4).
- **ArgoCD** → `Application`, `AppProject`.

The operator-pattern insight: **everything you want a
controller to maintain** can be expressed as a CRD + a
reconciliation loop. CRDs are the API surface; controllers
are the action. This is the same pattern as the built-in
types (Deployment + deployment-controller); you're just
adding new ones.

#### Example

A `ServiceMonitor` from movies-bartr (the spec §8.1
requirement):

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: movies-api
  namespace: movies
  labels:
    app.kubernetes.io/name: movies-api
    monitoring.coreos.com/instance: prometheus
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: movies-api
  namespaceSelector:
    matchNames:
      - movies
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
      scheme: http
```

This isn't a K8s built-in type. It exists because the
Prometheus Operator's CRDs are installed in the cluster.
When you `kubectl apply` this:

1. The API server validates it against the CRD schema.
2. The Prometheus Operator (a Deployment running somewhere in
   the cluster) sees the new ServiceMonitor.
3. The Operator updates the Prometheus instance's scrape
   config to include movies-api.
4. Prometheus reloads its config and starts scraping
   `http://movies-api.movies:8080/metrics` every 30 seconds.

Without the CRD + operator, you'd hand-write Prometheus
scrape config and reload Prometheus yourself on every change.
With them, you write one declarative YAML and the operator
does the rest.

#### Lab

1. `kubectl api-resources` — list every resource type the API
   server knows about. Scroll for the non-built-ins (anything
   with an `apiGroup` other than core/apps/batch/etc.).
2. `kubectl get crds` — list installed CRDs. On a movies-bartr
   cluster with Prometheus Operator: see `servicemonitors`,
   `prometheuses`, `alertmanagers`, etc.
3. `kubectl explain servicemonitor.spec` — read the schema.
   `kubectl explain servicemonitor.spec.endpoints` — drill
   in. CRDs ship `kubectl explain` documentation if defined
   correctly.
4. **Apply movies-bartr's ServiceMonitor.** `kgs -A | grep
   monitor` — find the Prometheus Operator. Confirm it logs
   that it saw the new ServiceMonitor.
5. **Find a CRD with no controller.** `kubectl apply
   --dry-run=server -f` a ServiceMonitor in a cluster that
   *doesn't* have Prometheus Operator. The apply succeeds
   (the CRD might be installed by something else) but nothing
   happens. *This is why "CRD + operator" come as a pair.*
6. **The operator pattern lab.** Read the Prometheus
   Operator's Deployment (`kubectl get deploy -n monitoring
   prometheus-operator -o yaml | head -100`). See it's just a
   Go binary with permissions to watch ServiceMonitors and
   write Prometheus configs.

#### Knowledge check

1. CRD vs controller — what's the difference, and what does
   each contribute?
2. Five CRDs you've used (or could use) in the curriculum,
   and what controller owns each.
3. `kubectl explain` works on CRDs. Why does that matter for
   "the new type behaves like a built-in"?
4. You `kubectl apply` a ServiceMonitor in a cluster with the
   CRD installed but no operator running. What happens?
5. The operator pattern is one of K8s's most important
   primitives. State it in one sentence.

---

### Module 11 — NetworkPolicy (C15) — pointer

> **Covered in depth in
> [study-guide-security.md](study-guide-security.md)
> Module 2.** This module is a pointer + the C-domain framing.

#### Concept

NetworkPolicy is a K8s object that lets you write "this pod
can/cannot talk to that pod." Default behavior in K8s is *every
pod can reach every other pod*; NetworkPolicy is the lever that
changes that.

The C-domain framing: **NetworkPolicy is a K8s primitive, not a
security tool.** It's part of the same mental model as Pods and
Services. Implementation is left to the CNI (Calico, Cilium,
Antrea, k3s's flannel + the kube-router NP add-on, etc.).
**Confirm your CNI implements NetworkPolicy before you ship
one** — k3s's default flannel does *not* (you need to install
kube-router or switch CNIs); k3d depends on how it was
spun up; managed K8s (EKS/AKS/GKE) usually does by default.

For the curriculum: the **default-deny + narrow-allow pattern**
(security guide Module 2) is the right shape. Don't write one
big policy; write one default-deny that selects every pod, then
narrow-allow policies per workload.

#### Example

The high-level structure (full version in security guide):

```yaml
# Policy 1 — default-deny in the namespace.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: movies
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
# Policy 2 — allow specific traffic to movies-api.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: movies-api
  namespace: movies
spec:
  podSelector:
    matchLabels: { app.kubernetes.io/name: movies-api }
  ingress: [ ... ]    # see security guide for full details
  egress: [ ... ]
```

#### Lab

See [study-guide-security.md](study-guide-security.md) Module
2's lab (full default-deny + narrow-allow + break-on-purpose
sequence). The C-domain addition:

1. **Confirm your CNI supports NetworkPolicy first.** Apply
   the default-deny. From a debug pod in the namespace
   (without matching labels), try to reach a pod in the
   namespace. If the connection succeeds, your CNI is not
   enforcing NetworkPolicy. Investigate before assuming.

#### Knowledge check

1. NetworkPolicy as a K8s primitive — who actually enforces
   it?
2. k3s's default CNI (flannel) doesn't enforce NetworkPolicy.
   What are your options?
3. Default-deny + narrow-allow — why this pattern over one big
   policy?
4. (Cross-reference) See security guide Module 2 for the rest.

---

### Module 12 — Reading a cluster cold (capstone)

> **The capstone.** If the operator can be dropped into an
> unfamiliar cluster and produce a one-page status report in
> 15 minutes, the rest of the curriculum is unlocked.

#### Concept

The skill: walk into a cluster you've never seen and answer:

1. **What kind of cluster is this?** Distribution, version,
   node count, ages.
2. **What's running?** Namespaces, workloads, recent events.
3. **What's healthy?** Pods in non-Running states, recent
   restarts, recent events with reason ≠ Normal.
4. **What's exposed?** Services with EXTERNAL-IPs, Ingresses,
   LoadBalancers.
5. **What controllers are managing it?** Operators in
   kube-system / -operators / wherever, what CRDs they've
   installed.
6. **What can I touch safely?** Your effective RBAC.

The 15-minute walkthrough as a runnable script:

```bash
# 1. What kind of cluster.
kubectl cluster-info
kubectl version --short
kubectl get nodes -o wide
kubectl get nodes -o jsonpath='{.items[*].status.nodeInfo.kubeletVersion}'

# 2. What's running.
kubectl get ns
kubectl get pods -A -o wide | head -50
kubectl get events -A --sort-by='.lastTimestamp' | tail -20

# 3. What's healthy.
kubectl get pods -A --field-selector=status.phase!=Running
kubectl get pods -A -o jsonpath='{range .items[?(@.status.containerStatuses[*].restartCount>0)]}{.metadata.namespace}/{.metadata.name}: {.status.containerStatuses[*].restartCount}{"\n"}{end}'

# 4. What's exposed.
kubectl get svc -A | grep -v ClusterIP
kubectl get ingress -A
kubectl get gateway -A 2>/dev/null   # if Gateway API is installed

# 5. Controllers + CRDs.
kubectl get pods -n kube-system
kubectl get pods -A | grep -iE 'operator|controller'
kubectl get crds

# 6. My RBAC.
kubectl auth can-i --list
kubectl auth can-i '*' '*'   # am I cluster-admin?
```

The output is a one-page Markdown summary:

```markdown
# Cluster status — <name> — <date>

**Cluster:** k3d v1.30, 1 server + 2 agents, all 14 days old
**Namespaces:** 8 (kube-system, default, monitoring, movies, ...)
**Total pods:** 47 (all Running)
**Recent events:** Normal scaling on movies-api 2h ago
**Restarts last 24h:** none > 1
**Exposed services:** 3 LoadBalancers (movies-api:8080,
  grafana:3000, prometheus:9090)
**Ingresses:** 0
**Operators visible:** prometheus-operator (monitoring),
  traefik (kube-system)
**CRDs installed:** monitoring.coreos.com (8), traefik.io (5)
**My RBAC:** cluster-admin (lab cluster)
```

Same skill, same template, scaled from a 3-node lab to a
800-cluster fleet (each cluster gets the same one-page).

#### Lab

1. On your k3d/k3s lab: run the script above. Produce the
   one-page report in 15 minutes.
2. **The "unfamiliar cluster" drill.** Have a teammate (or
   yourself, with intentional amnesia) hand you a kubeconfig
   for a cluster you didn't build. Time yourself producing
   the same report.
3. **Find one anomaly per cluster.** Even healthy clusters
   have noise — a pending pod that never scheduled, a
   misconfigured ServiceMonitor, an unused PVC. Train the
   eye.
4. **The "what happened in the last 24 hours" drill.**
   `kubectl get events -A --sort-by='.lastTimestamp' | tail
   -100`. Read the story. Identify any deploys, any
   restarts, any errors.
5. **The "what RBAC do I have" drill.** `kubectl auth can-i
   --list` from your default kubeconfig vs from a
   non-cluster-admin one. Note what changes.

#### Knowledge check

1. The six questions of the cold-cluster read — name them.
2. `kubectl auth can-i --list` — when do you reach for it?
3. The one-page report scales from a 3-node lab to a fleet.
   What makes it scale?
4. A pod in `CrashLoopBackOff` for 6 hours. What's the
   one-line `kubectl` command that almost always tells you
   why?
5. Three signals that a cluster is unhealthy that *don't*
   require looking at every individual pod.

---

## Per-release review

Same template as the other guides; see the
[Per-release review template section](study-guide-observability.md#per-release-review-template).

K8s-core-specific addition: **every release runs the
cold-cluster read** (Module 12) on the touched cluster before
the release is closed. The point isn't ceremony; it's catching
the unhealthy pod, the dangling RBAC, the new CRD nobody
noticed got installed.

A release without a clean cold-cluster read is fix-then-ship,
not ship.

## What this guide is and is not

- **Is:** the K8s slice every other guide assumes — control
  plane, distro choice, kubectl fluency, ownership chains,
  Services + DNS, probes, resources, ConfigMaps/Secrets,
  RBAC, CRDs, NetworkPolicy as primitive.
- **Is not:** a K8s certification study guide. CKA/CKAD/CKS
  cover this material in much more depth. The curriculum bar
  is fluency in the spec's surface, not the full exam.
- **Is not:** a deep-dive on any one subsystem.
  StatefulSets, PVs/PVCs/StorageClasses, HPA/VPA, PodDisruptionBudgets,
  PriorityClasses, taints/tolerations, affinity rules,
  Gateway API, EndpointSlices — all real, all eventually
  relevant, all out of scope here. They earn their own
  modules when they enter the spec.
- **Is not:** a managed-K8s comparison. EKS vs AKS vs GKE
  trade-offs deserve their own guide for an operator who'll
  actually run one. Curriculum bar is k3s/k3d.

## Open questions

- **StatefulSets + PVCs** — the data-plane half of K8s,
  deliberately out of scope because movies-spec is stateless.
  Worth its own module if/when the spec grows persistent
  data.
- **HPA / VPA / Cluster Autoscaler** — autoscaling is a
  spec-floor question once you have real traffic. Currently
  out of scope; possibly a separate guide.
- **Gateway API** — the successor to Ingress, increasingly
  the right answer for new clusters. Currently the curriculum
  uses Traefik's IngressRoute CRD (domain F); a Gateway API
  module belongs in F or here.
- **The "checklist + labs" question for NetworkPolicy and CRDs.**
  These point to other guides; should they be promoted to
  short standalone modules in C, or stay as pointers?
- **PodDisruptionBudget + priorityClass + taints/tolerations**
  — the cluster-citizenship layer. Worth a future module.

## Status

- Not yet run end-to-end with any operator.
- Anchored in movies-bartr's real `deploy/movies/base/`
  manifests and the k3s/k3d + klipper-lb + Traefik +
  Prometheus-Operator pattern the rest of the curriculum
  assumes.
- Unlocks for promotion to `methodology/` once at least one
  full run has used Module 12's cold-cluster read at release
  and reported on whether it caught real issues.
