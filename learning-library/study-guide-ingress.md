# Study Guide — Ingress and Routing (domain F)

> **DRAFT — NOT FOR PUBLICATION.** Eleventh instance of
> the study-guide format. Scoped from domain F of
> [skills-inventory.md](skills-inventory.md). Anchored in
> movies-bartr's three-Ingress setup
> (`deploy/movies/base/ingress.yaml`,
> `deploy/prometheus/base/ingress.yaml`,
> `deploy/grafana/base/ingress.yaml`) and the k3s Traefik
> `HelmChartConfig` entrypoint pattern
> (`deploy/traefik/base/entrypoints.yaml`).

## Why this exists

Ingress is the layer where most "it works locally but
not in prod" stories live. The Service is right, the pod
is healthy, the cluster is fine — but the request never
makes it past the ingress controller, or it lands on the
wrong backend, or the TLS handshake fails, or a rate-limit
middleware is silently dropping every fourth request.

The agent will generate an Ingress manifest endlessly,
and most of the time it'll be wrong in a way that
*looks* correct: no `ingressClassName`, host filters
that conflict with another router, the wrong entrypoint
on a multi-port cluster, an annotation from a different
controller's docs, TLS that "works" because the browser
silently accepted a self-signed cert.

This guide makes the operator fluent at reading an
ingress baseline, predicting how a request will route,
and knowing where the cliffs are. It's opinionated on
**Traefik** because that's the curriculum-wide
local-cluster pick (k3s/k3d ships it by default), and
opinionated on **vanilla `Ingress` over `IngressRoute`**
for the spec floor because portability matters more than
Traefik's extra expressiveness for most workloads.

**What this guide is anchored in:**

- movies-bartr's three real Ingresses on three real
  entrypoints, all on `localhost`, all surviving the
  agent's defaults that would have collided them.
- k3s's `HelmChartConfig` mechanism for patching the
  bundled Traefik chart with extra entrypoints —
  `deploy/traefik/base/entrypoints.yaml`.
- The repo-memory rule: **every Ingress declares
  exactly one entrypoint.**
- [study-guide-k8s-core.md](study-guide-k8s-core.md)
  Module 5 (Services + LoadBalancer) for the
  layer-below mental model.
- [study-guide-kustomize.md](study-guide-kustomize.md)
  Module 7 for the dev-LB / prod-IngressRoute pattern.

## How to use this guide

Same protocol as the other study guides. One
curriculum-level rule specific to this one: **every lab
that adds or modifies an Ingress predicts the exact URL
that should resolve to the new backend, then `curl -v`
confirms.** Ingress is the layer where wishful thinking
costs the most; the antidote is to write the prediction
before the change and check it after.

## Modules

### Module 1 — Layers of routing: Service / Ingress / LoadBalancer / Gateway API (F1)

#### Concept

Four concepts, three layers. The operator must keep
them clearly separated:

| Layer | Resource | What it answers | Who owns it |
|---|---|---|---|
| L4 (in cluster) | `Service` (ClusterIP) | "How does pod A reach pod B by a stable name?" | App team |
| L4 (cluster edge) | `Service` (LoadBalancer / NodePort) | "How does anything outside the cluster reach this Service?" | Platform team (k3s gives you `klipper-lb` for free) |
| L7 (cluster edge) | `Ingress` / `IngressRoute` / `Gateway` + `HTTPRoute` | "Which HTTP host / path / header should land on which Service?" | Mixed — platform owns the controller, app owns the route |
| L7 control plane | `IngressClass` / `GatewayClass` | "Which controller implements these rules?" | Platform team |

The trap: confusing L4 and L7. A `Service` of `type:
LoadBalancer` answers "I want a stable IP and port the
outside world can hit," not "I want HTTP routing." If
you find yourself adding host headers and path rules to
a Service, you wanted an Ingress. If you find yourself
defining HTTP rules but nothing actually routes them,
you forgot the IngressClass or the controller isn't
installed.

The mental model that makes the rest of this guide
make sense:

```
                  [internet / LAN]
                         |
                  ┌──────▼──────┐
                  │ LoadBalancer│  ← L4 (klipper-lb on k3s, cloud LB in cloud)
                  │  Service    │
                  └──────┬──────┘
                         |
                  ┌──────▼──────┐
                  │   Ingress   │  ← L7 (Traefik on k3s)
                  │  Controller │     reads Ingress / IngressRoute / Gateway+HTTPRoute
                  └──────┬──────┘
                         |
            ┌────────────┼────────────┐
            ▼            ▼            ▼
       ┌─────────┐  ┌─────────┐  ┌─────────┐
       │ Service │  │ Service │  │ Service │  ← L4 (ClusterIP)
       │  app1   │  │  app2   │  │  app3   │
       └─────────┘  └─────────┘  └─────────┘
            │            │            │
          [pods]       [pods]       [pods]
```

Three layers; each one has one job. The Gateway API is
a fourth name for the L7 layer that aims to replace
`Ingress` (see Module 6).

#### Example

In movies-bartr, all four layers are visible:

| Resource | Layer | Purpose |
|---|---|---|
| `Service movies/movies-api` (ClusterIP) | L4 in-cluster | webv pod reaches movies-api by DNS |
| Traefik's `Service kube-system/traefik` (LoadBalancer) | L4 edge | k3s's klipper-lb publishes Traefik on host ports |
| `Ingress movies/movies-api` | L7 edge | HTTP rule: `host=localhost` on entrypoint `web` → Service `movies-api` |
| `IngressClass traefik` (installed by k3s) | L7 control plane | Tells the cluster which controller handles Ingresses |

Remove any one of the four and routing breaks in a
specific way.

#### Lab

1. In movies-bartr, run `kubectl get svc -A` and label
   each one as ClusterIP / NodePort / LoadBalancer.
   For each LoadBalancer, find the host port it's
   published on.
2. Run `kubectl get ingress -A` and `kubectl get
   ingressclass`. Confirm every Ingress's
   `ingressClassName` matches an installed
   IngressClass.
3. Delete the IngressClass (in a scratch cluster, not
   the real one!). Predict what happens to existing
   Ingresses. Confirm.
4. Reinstall it, recreate the Ingress. Confirm
   routing returns.

#### Knowledge check

- A teammate writes a `Service` with `type:
  LoadBalancer` and adds rules for `host: api.com`
  paths `/v1` and `/v2`. What did they actually want?
- What happens to an Ingress with `ingressClassName:
  traefik` if the Traefik controller is uninstalled?
  Is the Ingress deleted? Is it inert?
- Why does the curriculum keep `Service` and `Ingress`
  in two different mental boxes even though the
  Ingress controller *uses* a Service to expose
  itself?

---

### Module 2 — Traefik as the k3s default (F2)

> **Don't fight the distro.** k3s ships Traefik
> bundled and reconciled by the helm-controller. If
> you uninstall Traefik to install ingress-nginx,
> you have just signed up to fight the distro's
> reconciler forever. Either accept Traefik or
> install a different distro.

#### Concept

k3s ships with Traefik installed as the default
ingress controller. This is *not* an unfortunate
default to work around — Traefik is a competent L7
proxy with a more expressive native CRD model than
vanilla `Ingress`, a useful dashboard, and tight
integration with the k3s helm-controller. The
curriculum's opinion is to learn it.

**Two ways to drive Traefik:**

1. **Vanilla `networking.k8s.io/v1 Ingress`** —
   portable across all ingress controllers, limited
   to whatever the controller can express via
   annotations. Use this when you might ever switch
   controllers, or when the routing rules are
   simple.
2. **Traefik CRDs (`IngressRoute`, `Middleware`,
   `TLSStore`, `ServersTransport`, ...)** —
   Traefik-specific, more expressive, supports
   route priorities, matcher syntax, middleware
   chaining via references rather than annotations.
   Use this when vanilla `Ingress` makes you fight
   the controller, or for shared middleware (rate
   limit, auth) used by many routes.

The curriculum's spec-floor choice: **vanilla
`Ingress`** for application routes (portability
beats expressiveness for spec-bar workloads),
**`IngressRoute` + `Middleware`** for cluster-level
concerns (rate limits, TLS policies, shared auth).
movies-bartr uses vanilla `Ingress` throughout
because the routing is simple.

**Traefik's dashboard** — `traefik.io/p/d/` lives at
the Traefik admin port (9000 by default). It shows
every Router, Service, and Middleware Traefik knows
about, including the ones synthesized from your
Ingresses. **The single most useful debug tool** when
"why isn't this route working" — if your route isn't
in the dashboard, Traefik never saw it, and the
problem is between you and the controller (bad
`ingressClassName`, bad annotation, etc.).

#### Example

The same routing rule, expressed two ways.

**Vanilla `Ingress` (movies-bartr's current shape):**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: movies-api
  namespace: movies
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  ingressClassName: traefik
  rules:
    - host: localhost
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: movies-api
                port:
                  number: 8080
```

**Traefik `IngressRoute` (the equivalent in CRD form):**

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: movies-api
  namespace: movies
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`localhost`)
      kind: Rule
      services:
        - name: movies-api
          port: 8080
```

Same behavior. The IngressRoute makes the entrypoint
a first-class field instead of an annotation, and the
matcher expression is more powerful (`Host(...) &&
PathPrefix(...) && Header(...)`). It's also locked
into Traefik — every controller migration becomes a
manual rewrite.

#### Lab

1. Find Traefik's dashboard in the movies-bartr
   cluster:
   ```sh
   $ kubectl get svc -n kube-system traefik
   # Find the dashboard port; expose it if needed:
   $ kubectl port-forward -n kube-system svc/traefik 9000:9000
   ```
   (This is one of the few legitimate uses of
   `port-forward` — admin UIs that don't earn a
   permanent LB port.)
2. Open `http://localhost:9000/dashboard/`. Find the
   `movies-api@kubernetes` router. Confirm its
   entrypoint, rule, and backend match the YAML.
3. Convert the movies-api Ingress to an IngressRoute
   (in a scratch overlay). Apply. Confirm the
   dashboard now shows
   `movies-api@kubernetescrd` instead of
   `@kubernetes`. Notice the priority and matcher
   syntax are now first-class.
4. Revert. Decide whether IngressRoute is worth the
   portability cost for this workload — and write a
   one-paragraph answer.

#### Knowledge check

- Why does k3s ship Traefik instead of ingress-nginx?
  (It's a deliberate choice.)
- An Ingress doesn't show up in the Traefik
  dashboard. Three causes — what are they?
- A teammate proposes converting every Ingress in
  the cluster to IngressRoute "because it's more
  powerful." What's the question to ask first?
- What's the difference between `traefik` (an
  IngressClass) and the `traefik` ServiceAccount
  the controller runs as?

---

### Module 3 — Entrypoints and port discipline (F2 anchor)

> The curriculum's strongest opinion in domain F.
> Every Ingress declares exactly one entrypoint. The
> agent's defaults will violate this and silently
> collide your routes.

#### Concept

A Traefik **entrypoint** is a listener on a specific
port. The bundled k3s chart ships two:

- `web` on port 80.
- `websecure` on port 443.

That's enough if you only ever serve one
application on host port 80 and one on 443. The
moment you want to serve more than one thing per
host (movies-api on 80, Prometheus on 9090, Grafana
on 3000, vLLM on 8000, ...), you need additional
entrypoints — and you need to pin each Ingress to
exactly one of them.

**The trap the agent's defaults will spring:** if
you create three Ingresses with `host: localhost`
and no entrypoint pin, Traefik attaches each
router to **every** entrypoint. The result:
`http://localhost:9090/` resolves to the
movies-api home page instead of Prometheus,
because the movies-api router answered first on
the 9090 entrypoint as well.

This isn't a bug. It's exactly what the Ingress
spec says should happen when you don't constrain
the binding. But it's not what anyone wants. The
fix is the annotation:

```yaml
metadata:
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: prometheus
```

That pins the Ingress's router to the named
entrypoint — and **only** that entrypoint. No
collisions, no host header gymnastics required.

**The curriculum rule:** every Ingress in a
multi-entrypoint cluster declares exactly one
entrypoint annotation. If you forget it, the agent
forgets it, and the next operator wastes an hour
chasing a routing collision.

#### Example

movies-bartr's `deploy/traefik/base/entrypoints.yaml`
publishes five additional entrypoints via
`HelmChartConfig` — the k3s mechanism for patching
a bundled HelmChart in-place:

```yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: traefik
  namespace: kube-system
spec:
  failurePolicy: reinstall
  valuesContent: |-
    ports:
      vllm:        { port: 8001, exposedPort: 8000, protocol: TCP, expose: { default: true } }
      cllm:        { port: 8089, exposedPort: 8088, protocol: TCP, expose: { default: true } }
      ask:         { port: 8009, exposedPort: 8008, protocol: TCP, expose: { default: true } }
      prometheus:  { port: 9091, exposedPort: 9090, protocol: TCP, expose: { default: true } }
      grafana:     { port: 3001, exposedPort: 3000, protocol: TCP, expose: { default: true } }
```

| Entrypoint | Service port (Traefik) | Host port (klipper-lb publishes) | Used by |
|---|---|---|---|
| `web` (default) | 8000 | 80 | movies-api on `localhost:80` |
| `websecure` (default) | 8443 | 443 | TLS workloads (not in this stack) |
| `prometheus` | 9091 | 9090 | `deploy/prometheus` Ingress |
| `grafana` | 3001 | 3000 | `deploy/grafana` Ingress |
| `vllm` / `cllm` / `ask` | 8001 / 8089 / 8009 | 8000 / 8088 / 8008 | future workloads |

And each Ingress pins itself:

```yaml
# movies/ingress.yaml
annotations:
  traefik.ingress.kubernetes.io/router.entrypoints: web

# prometheus/ingress.yaml
annotations:
  traefik.ingress.kubernetes.io/router.entrypoints: prometheus

# grafana/ingress.yaml
annotations:
  traefik.ingress.kubernetes.io/router.entrypoints: grafana
```

Result: clean separation. `http://localhost/` →
movies-api. `http://localhost:9090/` → Prometheus.
`http://localhost:3000/` → Grafana. No collisions,
no host-header tricks, no `port-forward`. And every
service in the stack reachable on a real, stable LB
port — the K8s-core M5 / Kustomize M7 LB-not-
port-forward opinion holds end-to-end.

**Note on the helm-controller subtlety:** the
`HelmChartConfig` is informational on a fresh node —
k3s only re-renders the chart when the
HelmChartConfig itself changes. The Kustomize
ownership in `deploy/traefik/base/` is the durable
source of truth for the entrypoint table, but the
chart itself is reconciled by the k3s
helm-controller from `kube-system/HelmChart
traefik`. This is the cleanest way to live with a
distro-bundled controller: own the *config*, not
the chart.

#### Lab

1. In a scratch k3s cluster, create two Ingresses
   on `host: localhost` with no entrypoint
   annotation, pointing at two different Services.
   Predict what happens. Confirm with `curl`.
2. Add the `router.entrypoints: web` annotation to
   one of them. Confirm the collision resolves.
3. Add a new entrypoint to the k3s Traefik chart
   via `HelmChartConfig`. Apply. Wait for the
   helm-controller to reconcile (`kubectl get pods
   -n kube-system -w` to watch). Confirm the new
   port appears on `netstat -ln` on the host.
4. Create a new Ingress pinned to the new
   entrypoint. Curl. Confirm.
5. Try the same thing on a cluster where you've
   deliberately uninstalled Traefik and installed
   ingress-nginx. Notice how much more work the
   "extra entrypoint" pattern becomes when you've
   left the distro's defaults.

#### Knowledge check

- An Ingress is missing the entrypoint annotation
  on a five-entrypoint cluster. What's the worst-
  case behavior?
- Why is the `expose: { default: true }` in the
  HelmChartConfig values load-bearing?
- A teammate adds `Host(localhost) &&
  Path(/prometheus)` to the movies-api Ingress
  "to put Prometheus behind a path instead of a
  port." Two reasons this is worse than the
  entrypoint pattern.
- The chart re-renders only when the
  `HelmChartConfig` changes. What does that imply
  about how you bootstrap a fresh cluster vs how
  you update an existing one?

---

### Module 4 — Middleware patterns (F2 depth)

> Rate limit, auth, headers, redirects, strip
> prefix. The Traefik `Middleware` CRD is the
> right primitive; the annotation form is the
> portable fallback.

#### Concept

A middleware is a request-modifying function the
ingress controller runs between "match the route"
and "forward to the backend." Common ones, in
rough order of how often you'll need them:

| Middleware | What it does | Use when |
|---|---|---|
| `rate-limit` | Caps requests per second per source IP | Public endpoints with abuse risk; pre-prod cost-control |
| `headers` | Add / remove / modify request or response headers | Security headers (HSTS, CSP, X-Frame-Options); CORS; injecting `X-Forwarded-*` |
| `redirect-scheme` | Force HTTP → HTTPS | Almost always, on production websecure entrypoint |
| `strip-prefix` | Remove a URL prefix before forwarding | Multi-service routing where each backend expects to be at `/` |
| `basic-auth` / `forward-auth` | Require credentials or delegate to an auth service | Dashboard / admin UI gates; pre-prod login walls |
| `ip-allow-list` (formerly `ip-whitelist`) | Allow only specified source IPs | Admin endpoints; pre-prod environments; CI runners |
| `compress` | gzip / brotli response | Public APIs serving JSON over the public internet |
| `circuit-breaker` | Stop sending to a backend that's failing | Cascading-failure prevention |
| `retry` | Retry failed requests | Idempotent endpoints only — never on POST/PATCH |

Two ways to attach middleware to an Ingress:

**Annotation form (works with vanilla `Ingress`):**

```yaml
annotations:
  traefik.ingress.kubernetes.io/router.middlewares: monitoring-rate-limit@kubernetescrd
```

The middleware itself is still a `Middleware`
CRD; the annotation just references it. The
suffix `@kubernetescrd` tells Traefik which
provider the middleware lives in.

**IngressRoute form (Traefik-native):**

```yaml
spec:
  routes:
    - match: Host(`localhost`)
      kind: Rule
      middlewares:
        - name: rate-limit
      services:
        - name: movies-api
          port: 8080
```

Middleware is composable: you reference it by
name, and the same middleware can be attached to
many routes. **One `Middleware` CRD per concern,
referenced by many routes**, not "copy the rate-
limit config into every Ingress."

#### Example

A `rate-limit` middleware and its attachment:

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: rate-limit
  namespace: movies
spec:
  rateLimit:
    average: 100        # 100 req/s sustained per source IP
    burst: 200          # absorb bursts up to 200
    period: 1s
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: movies-api
  namespace: movies
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
    traefik.ingress.kubernetes.io/router.middlewares: movies-rate-limit@kubernetescrd
spec:
  ingressClassName: traefik
  rules:
    - host: localhost
      http:
        paths:
          - { path: /, pathType: Prefix, backend: { service: { name: movies-api, port: { number: 8080 } } } }
```

And a `redirect-scheme` middleware for the
HTTP → HTTPS pattern:

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: redirect-https
  namespace: traefik-system
spec:
  redirectScheme:
    scheme: https
    permanent: true
```

This middleware is attached to a *catch-all
Ingress* on the `web` entrypoint that has no
routing rules — its only job is to bounce every
HTTP request to HTTPS before the user-facing
controller sees it.

#### Lab

1. Create a `rate-limit` Middleware in
   movies-bartr at 10 req/s (deliberately
   aggressive for the lab). Attach it to the
   movies-api Ingress. Predict: webv's 584 RPS
   baseline will hit 429s. Confirm by watching
   the dashboard.
2. Remove. Confirm RPS returns to baseline.
3. Add a `headers` middleware that sets
   `Strict-Transport-Security:
   max-age=31536000` on every response. Curl
   `-v` to confirm.
4. Add an `ip-allow-list` middleware that allows
   only your laptop's IP. Curl from your laptop:
   should succeed. Try from any other source
   (a different VM): should 403. Predict the
   exact status code first.
5. Compose two middlewares on one route: `ip-
   allow-list` + `rate-limit`. Confirm order
   doesn't matter for these two (both are
   filters); contrast with `strip-prefix` +
   `rate-limit` where order *would* matter
   (the rate limit needs to see the stripped or
   unstripped URL?).

#### Knowledge check

- A teammate copies the same `rateLimit` config
  into every Ingress's annotations. Why is the
  `Middleware` CRD pattern strictly better?
- What's the difference between `headers`,
  `customRequestHeaders`, and
  `customResponseHeaders` in Traefik's
  middleware syntax?
- Middleware order matters. Given a chain `[ip-
  allow-list, rate-limit, basic-auth]`, what
  happens to a request from a denied IP? From
  an allowed IP without basic-auth credentials?
- A request hits the rate-limit. What HTTP
  status does Traefik return, and what header
  tells the client when to retry?

---

### Module 5 — TLS termination + cert-manager + Let's Encrypt (F4)

> Enterprise table stakes. Self-signed for dev,
> Let's Encrypt for prod, end-to-end TLS when you
> need it. The mechanism is the same; the issuer
> changes.

#### Concept

**TLS termination** means: the ingress controller
decrypts the inbound HTTPS request and forwards
plain HTTP to the backend Service in the cluster.
This is the default pattern and the right one for
most workloads — TLS terminates at the cluster
edge, internal traffic is plaintext (or mTLS via a
service mesh, but that's a separate layer).

**End-to-end TLS** (re-encryption from ingress to
backend) is needed when the backend requires HTTPS
(some legacy apps) or when intra-cluster traffic
must be encrypted by compliance mandate without a
service mesh. It's more complex; default to TLS
termination unless you have a reason.

**The certificate question — three answers:**

1. **Self-signed for dev/local.** Traefik
   generates one on startup. Browsers complain;
   `curl -k` works. Fine for `localhost` and
   internal-only services. movies-bartr does
   not use TLS in its dev setup at all — the
   `web` entrypoint on port 80 is enough for a
   local demo.

2. **Let's Encrypt via `cert-manager` + ACME.**
   The standard for any public-facing service.
   cert-manager watches `Ingress` resources with
   a `cert-manager.io/cluster-issuer` annotation,
   solves the ACME challenge, stores the cert in
   a `Secret`, and renews automatically before
   expiry. Two challenge types:
   - **HTTP-01:** Let's Encrypt hits
     `http://yourdomain/.well-known/acme-challenge/...`
     to prove you control the domain. Needs port
     80 reachable from the public internet.
     Simpler; works for single-domain certs.
   - **DNS-01:** cert-manager writes a TXT record
     to your DNS provider. Needed for wildcard
     certs (`*.example.com`) or when port 80
     isn't publicly reachable. More setup; needs
     credentials for your DNS provider.

3. **Internal CA / corporate PKI.** Enterprise
   environments often have an internal CA
   cert-manager can integrate with via the
   `Venafi` or `Vault` issuer types. Same
   mechanism, different issuer.

**Staging vs production issuer.** Let's Encrypt
rate-limits the prod issuer hard (50 certs per
domain per week). Every cert-manager setup uses
two ClusterIssuers — `letsencrypt-staging`
(unlimited, untrusted) and `letsencrypt-prod`
(rate-limited, trusted). Always test on staging
first; flip the annotation to prod when staging
is happy.

#### Example

A vanilla `Ingress` with cert-manager + Let's
Encrypt:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: movies-api-public
  namespace: movies
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
spec:
  ingressClassName: traefik
  tls:
    - hosts:
        - movies.example.com
      secretName: movies-api-tls    # cert-manager creates this
  rules:
    - host: movies.example.com
      http:
        paths:
          - { path: /, pathType: Prefix, backend: { service: { name: movies-api, port: { number: 8080 } } } }
```

The flow:

1. Apply this Ingress.
2. cert-manager sees the `cluster-issuer`
   annotation, creates a `Certificate` resource.
3. cert-manager creates a `CertificateRequest`,
   `Order`, and `Challenge`.
4. ACME completes the challenge (HTTP-01 hits
   port 80, or DNS-01 writes a TXT record).
5. Let's Encrypt issues the cert; cert-manager
   stores it in `Secret movies-api-tls`.
6. Traefik picks up the Secret and serves
   `https://movies.example.com/`.
7. cert-manager renews automatically at ~60 days.

The `letsencrypt-prod` ClusterIssuer (one-time
cluster setup):

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: ops@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      - http01:
          ingress:
            ingressClassName: traefik
```

For dev, swap the server URL for
`https://acme-staging-v02.api.letsencrypt.org/directory`.

#### Lab

*(This lab assumes a cluster with a public DNS
name; for a movies-bartr local cluster, run it
against a public-DNS dev cluster on a cloud
droplet.)*

1. Install cert-manager:
   ```sh
   $ kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
   ```
2. Create the staging ClusterIssuer. Apply.
3. Create an Ingress on a real domain with the
   staging annotation. Watch:
   ```sh
   $ kubectl describe certificate movies-api-tls
   $ kubectl describe challenge -A
   ```
4. Once `kubectl get cert` shows `READY=True`,
   `curl -v https://yourdomain/`. The cert is
   from staging — `curl` warns about the
   untrusted issuer. That's expected.
5. Flip the annotation to `letsencrypt-prod`.
   Delete the existing Secret to force re-
   issuance. Watch the renewal. Confirm `curl`
   no longer warns.
6. Find when the cert expires (`kubectl get
   cert -o yaml`). Note: cert-manager will
   renew it at ~60 days; you don't have to.

#### Knowledge check

- Why does every cert-manager setup keep two
  ClusterIssuers (staging + prod)?
- A teammate's cert is stuck in `Pending` for an
  hour. Three causes — what are they, and which
  `kubectl describe` shows each?
- HTTP-01 vs DNS-01: when is each right? What
  blocks each one in a typical enterprise
  network?
- The Secret `movies-api-tls` is deleted by
  accident. What happens to traffic? How long
  until it's restored?

---

### Module 6 — Ingress vs Gateway API: the migration question (F5)

> Ingress is sufficient. Gateway API is the
> future. The honest answer to "should we
> migrate?" is "not yet for most teams, yes
> eventually for everyone."

#### Concept

**Ingress** (`networking.k8s.io/v1`) was
graduated to v1 in Kubernetes 1.19 (2020) and
hasn't materially changed since. It works for
~90% of HTTP routing needs. It has known
limitations — vendor-specific annotations,
weak multi-tenancy story, no native support for
non-HTTP protocols.

**Gateway API** (`gateway.networking.k8s.io`) is
the successor. v1 (`v1.0`) graduated in October
2023 (~2.5 years ago as of mid-2026). What it
gets right:

1. **Role separation.** Three resource kinds:
   `GatewayClass` (cluster ops), `Gateway`
   (platform team), `HTTPRoute` / `TCPRoute` /
   `GRPCRoute` (app team). Each role binds to
   a clear resource; no more "the app team
   reaches into the platform team's
   annotations."
2. **Multi-protocol.** Native support for
   HTTP, HTTPS, TCP, UDP, gRPC, and TLS
   passthrough — `Ingress` is HTTP-only.
3. **Expressive routing.** Header matching,
   query-parameter matching, weighted
   traffic splitting (the canary primitive),
   request mirroring — all without vendor-
   specific annotations.
4. **Cross-namespace references.** A
   Gateway in one namespace can be referenced
   by an HTTPRoute in another, with explicit
   permission. Real multi-tenancy.

**When Ingress is still right:**

- Single team, single namespace, simple HTTP
  routing.
- Local-cluster work where Traefik's `Ingress`
  + annotation patterns are already fluent.
- Anything where portability across ingress
  controllers matters (Ingress is fully
  portable; HTTPRoute is portable but
  controllers vary on which features they
  support).

**When Gateway API earns its complexity:**

- Multi-team clusters where role separation
  matters.
- Multi-protocol workloads (gRPC, TCP, UDP
  alongside HTTP).
- Sophisticated traffic management — canary
  rollouts, request mirroring, header-based
  routing for A/B testing.
- Cross-namespace routing (Gateway in
  `infrastructure`, HTTPRoute in `apps`).

**The curriculum's position:** **learn
`Ingress` first, deeply.** It's the lingua
franca; every team you'll work with has it;
the patterns transfer. **Move to Gateway API
when you hit a real limitation of Ingress
on a real workload, not pre-emptively.** Most
teams will not hit that limitation; the
ones that do will know it.

#### Example

The same routing rule in Ingress and Gateway API:

**Ingress (Traefik):**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: movies-api
  namespace: movies
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  ingressClassName: traefik
  rules:
    - host: localhost
      http:
        paths:
          - { path: /, pathType: Prefix, backend: { service: { name: movies-api, port: { number: 8080 } } } }
```

**Gateway API (Traefik supports it as of v3):**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: traefik
  namespace: kube-system
spec:
  gatewayClassName: traefik
  listeners:
    - name: web
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: All
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: movies-api
  namespace: movies
spec:
  parentRefs:
    - name: traefik
      namespace: kube-system
  hostnames:
    - localhost
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: movies-api
          port: 8080
```

Twice as much YAML for the same behavior — but
the second form has clean role separation
(platform owns the Gateway in `kube-system`,
app owns the HTTPRoute in `movies`),
explicit cross-namespace binding, and no
controller-specific annotations.

For movies-bartr, the simpler Ingress is right.
For a multi-team cluster with five teams each
needing their own HTTP rules onto a shared LB,
Gateway API earns its keep.

#### Lab

1. Read the Gateway API conformance report for
   your controller: https://gateway-api.sigs.k8s.io
   → "Implementations." Note which features
   Traefik fully supports, partially supports,
   or skips.
2. Install the Gateway API CRDs (if not
   already present): `kubectl apply -f
   https://github.com/kubernetes-sigs/gateway-api/releases/latest/download/standard-install.yaml`
3. Create a `GatewayClass`, `Gateway`, and
   `HTTPRoute` for the movies-api service
   alongside the existing Ingress. Confirm both
   route to the same backend.
4. Demonstrate a feature that Ingress can't do
   natively: weighted backendRefs (e.g. 90/10
   split between two Services). Confirm webv's
   traffic distribution matches.
5. Decide for movies-bartr: keep Ingress, move
   to Gateway API, or run both side by side.
   Write the one-paragraph reasoning.

#### Knowledge check

- A Kubernetes consultant says "Ingress is
  deprecated, you must migrate to Gateway API."
  Why is this misleading?
- The Gateway API splits one Ingress resource
  into three (GatewayClass / Gateway /
  HTTPRoute). When is this *a feature* and
  when is it *bureaucratic overhead*?
- A teammate proposes Gateway API for the
  movies-bartr local cluster. Two reasons it's
  the wrong move. One reason it might be the
  right move anyway.
- What feature of Gateway API is the most
  compelling for *partner enablement* scenarios
  (multiple teams sharing one platform)?

---

### Module 7 — Capstone: reading an ingress baseline cold

> The 15-minute walkthrough that produces a
> one-page ingress report from an unfamiliar
> cluster. Same shape as the K8s-core M12
> capstone; specific to L7 routing.

#### Concept

You inherit a cluster (or join a partner's
cluster) and need to answer six questions about
its ingress posture in 15 minutes:

1. **Which controller is in charge?** (Traefik,
   ingress-nginx, Contour, Istio, ALB
   controller, ...) Is there exactly one, or
   are there multiple competing for the same
   class?
2. **What ports are publicly reachable, and
   what's on each?** (The entrypoint table,
   essentially.)
3. **What hostnames route to what backends?**
   The Ingress / HTTPRoute table.
4. **What middleware is in play?** Rate limits,
   auth, redirects, headers — anything that
   modifies requests before they hit the
   backend.
5. **What's the TLS posture?** Self-signed,
   Let's Encrypt, internal CA, end-to-end?
   When do the certs expire?
6. **Where are the silent failure modes?**
   Ingresses pointing at non-existent
   Services, Services with no endpoints,
   stale Secrets, Middleware references that
   404 in the controller log.

The deliverable: a one-page markdown report
that any operator can read in 60 seconds and
know what they're inheriting.

#### Example

A runnable script that answers all six:

```sh
#!/bin/sh
set -e
echo "## Ingress controllers"
kubectl get pods -A -l 'app.kubernetes.io/name in (traefik,ingress-nginx,contour,istiod)' -o wide
echo ""
echo "## IngressClasses"
kubectl get ingressclass -o wide
echo ""
echo "## Gateway API (if installed)"
kubectl get gatewayclass,gateway -A 2>/dev/null || echo "(Gateway API not installed)"
echo ""
echo "## Published ports (Traefik LB Service)"
kubectl get svc -n kube-system traefik -o json 2>/dev/null | \
  jq -r '.spec.ports[] | "\(.name): port=\(.port) targetPort=\(.targetPort) protocol=\(.protocol)"'
echo ""
echo "## All Ingresses, with backend, entrypoint, and TLS"
kubectl get ingress -A -o json | jq -r '
  .items[] |
  [.metadata.namespace, .metadata.name,
   (.spec.rules // [] | .[0].host // "<no host>"),
   (.spec.rules // [] | .[0].http.paths[0].backend.service.name // "<no svc>"),
   (.metadata.annotations["traefik.ingress.kubernetes.io/router.entrypoints"] // "web"),
   (if .spec.tls then "TLS" else "plain" end)] | @tsv' | \
  column -t
echo ""
echo "## All HTTPRoutes (Gateway API)"
kubectl get httproute -A 2>/dev/null -o wide
echo ""
echo "## Middleware (Traefik CRDs)"
kubectl get middlewares.traefik.io -A 2>/dev/null -o wide
echo ""
echo "## TLS Secrets + cert-manager Certificates"
kubectl get secret -A -o json | jq -r '
  .items[] |
  select(.type == "kubernetes.io/tls") |
  [.metadata.namespace, .metadata.name] | @tsv' | column -t
kubectl get certificate -A 2>/dev/null
echo ""
echo "## Silent-failure check: Ingresses pointing at non-existent Services"
# (Cross-reference each Ingress backend.service.name against `kubectl get svc -n <ns>`.
#  Left as an exercise — the most common silent failure on inherited clusters.)
echo ""
echo "## Recent ingress-controller errors"
kubectl logs -n kube-system -l app.kubernetes.io/name=traefik --tail=200 2>/dev/null | \
  grep -iE 'error|warn' | tail -20 || echo "(no traefik logs)"
```

#### Report template

```markdown
# Ingress posture: <cluster-name>

## Controllers
- <traefik 3.x / nginx-ingress 1.x / etc.>, single replica / HA?

## Entrypoints
| Entrypoint | Host port | Used by |
|---|---|---|
| web | 80 | movies-api |
| websecure | 443 | <none> |
| prometheus | 9090 | monitoring/prometheus |

## Routes (Ingresses + HTTPRoutes)
| Namespace | Name | Host | Path | Backend | Entrypoint | TLS |
|---|---|---|---|---|---|---|
| movies | movies-api | localhost | / | movies-api:8080 | web | no |
| monitoring | grafana | * | / | grafana:3000 | grafana | no |
| monitoring | prometheus | * | / | prometheus:9090 | prometheus | no |

## Middleware in use
- <list `Middleware` CRDs and which routes reference each>

## TLS posture
- <self-signed / Let's Encrypt staging / Let's Encrypt prod / internal CA>
- Certs expire: <date range>
- cert-manager installed: <yes/no>

## Silent failures detected
- <Ingress X points at Service Y which doesn't exist>
- <Cert Z expired 5 days ago>
- <none>

## Risk verdict
- ❌ Block: <or absent>
- ⚠ Fix soon: <or absent>
- ✅ Ship-ready: <or absent>
```

#### Lab

1. Run the script above on the movies-bartr
   cluster. Produce the report.
2. Deliberately introduce a silent failure: edit
   an Ingress's backend to point at a non-
   existent Service (`movies-api-typo`). Re-run.
   Confirm the report flags it. Notice that
   `kubectl get ingress` shows no error — the
   silent failure is exactly that, silent.
3. Run the script on a different cluster (cllm,
   a colleague's cluster, a fresh k3s). Note how
   the report shape transfers.
4. Time the full walkthrough. Target: 15 minutes
   from `kubectl config use-context` to a
   completed report. If it takes 30, the
   script needs tightening or the cluster's
   ingress is gnarlier than usual — both are
   findings.

#### Knowledge check

- Why is "Ingress pointing at non-existent
  Service" the most common silent failure on
  inherited clusters?
- The cluster has two IngressClasses, both
  named `default`. What does that mean for
  routing?
- An Ingress is in the cluster, the dashboard
  shows it, but `curl` returns "connection
  refused." What's the first place to look?
- This capstone takes 15 minutes against a
  3-namespace lab. What scales linearly with
  cluster size, and what stays constant?

---

## Per-release review

Per the curriculum-wide template in
[study-guide-observability.md](study-guide-observability.md):
every release runs the cold-cluster read (K8s-core
M12), the security capstone (security M7), the
image-size budget check (containers per-release),
the inner-loop signal check (dev-loop per-release),
the baseline-still-signaling check (testing per-
release), and **this guide's residual: the
entrypoint-pin audit.** Every Ingress in the
cluster has a `router.entrypoints` annotation
matching exactly one published entrypoint; no two
Ingresses share an entrypoint without an
intentional host-or-path discriminator. The audit
is one line:

```sh
$ kubectl get ingress -A -o json | jq -r '
    .items[] |
    [.metadata.namespace, .metadata.name,
     (.metadata.annotations["traefik.ingress.kubernetes.io/router.entrypoints"] // "MISSING")] | @tsv'
```

Anything with `MISSING` in column 3 is the next
silent collision waiting to happen.

## What this guide is

- The integration module for L7 routing — what an
  Ingress is, how Traefik implements it, how to
  drive it without colliding routes, how to add
  middleware and TLS, when to consider Gateway
  API, and how to read an ingress baseline cold.
- Anchored in movies-bartr's three-Ingress / five-
  entrypoint setup, which exists *because* the
  curriculum's "every service gets a real LB port"
  opinion (K8s-core M5) forced the entrypoint
  discipline into the open.
- The L7 counterpart to K8s-core's L4 coverage.

## What this guide is not

- Not a Traefik reference — the canonical one is
  at https://doc.traefik.io/traefik/.
- Not a complete cert-manager tutorial — covers
  the patterns the curriculum needs; deep ACME
  troubleshooting is its own domain.
- Not opinionated on service mesh (Istio, Linkerd,
  Cilium). Service mesh sits *alongside* ingress;
  the trade-offs deserve their own guide if the
  curriculum ever needs one.

## Open questions

1. Is Module 6's "learn Ingress first, deeply"
   position still right two years from now?
   Gateway API will keep maturing; controllers
   will improve coverage; the inflection point
   where Gateway API becomes the floor is
   foreseeable but not here yet. Re-check
   annually.
2. Module 4's middleware list omits OAuth /
   OIDC integration via `forward-auth` to an
   external identity provider. Should that be
   a sub-module for enterprise scenarios?
3. Module 5 sidesteps service-mesh mTLS as
   "alongside, not part of ingress." That's the
   right framing for the spec floor; an
   enterprise-bar variant would address it.

## Status

DRAFT. Not promoted to `methodology/` until at
least one operator runs the Module 7 capstone
against an unfamiliar (i.e. not their own) cluster
and produces a usable report in under 15 minutes.
That's the gate — if the capstone doesn't
generalize, the rest of the guide is
under-validated.
