# Study Guide — Kustomize and Manifest Packaging (movies-spec §8 + §9.1)

> **DRAFT — NOT FOR PUBLICATION.** Second instance of the study-guide
> format established by
> [study-guide-observability.md](study-guide-observability.md).
> Scoped from row D of
> [skills-inventory.md](skills-inventory.md). Canonical examples use
> the Go-on-distroless stack per domain M of the inventory.

## Why this exists

Matt-v1 shipped 1.0.0 with a `k8s/` tree that has one base and one
overlay (`dev/`). It works. It also will not scale past a second
environment without rewriting, and that gap is invisible until you
try to add `bench/`, `prod/`, `prod-eu/`, `edge-store-0042/`. The
spec doesn't require enterprise-scale Kustomize; the **methodology**
requires that the operator know the difference between "passes the
spec" and "scales to a 1K-cluster fleet" — and can defend the call.

This guide closes the gap that Matt-v1 made visible: the agent will
generate a single-environment overlay because that's what the spec
asks for; the operator who can only read that pattern can't push back
when the agent's output is going to bite the team in six months.

**What this guide is anchored in:**
- Spec §8 — Kustomize mandatory, Helm forbidden, base + overlays
  structure.
- Spec §9 — single multi-stage Dockerfile, distroless/Alpine runtime,
  non-root.
- Spec §12 — the inner-loop dev process that the manifest layout has
  to support.
- The cllm repo's `clusters/z01/` tree — a working example of the
  pattern this guide pushes toward, in a *Bart-owned* repo, MIT.

## How to use this guide

Same protocol as
[study-guide-observability.md](study-guide-observability.md):

- Pick the module that maps to upcoming work; don't read front-to-back.
- Run the lab with your own hands when the agent reaches for the tool.
- Append a one-paragraph "what I learned" to the session's RETRO entry.
- At each tag, run the per-release review (template at the end of
  the observability guide; same template applies here).

## Modules

### Module 0 — Repo topology: where do dev, staging, prod actually live?

> **Read this before Module 2.** The base + overlay shape you'll
> learn in Modules 2–4 lives *inside* one repo. Which repo? That's
> a separate question, and it's the one the rest of the
> curriculum depends on getting right.

#### Concept

Kustomize teaches you to express *transformations*. It says nothing
about *where the transformed manifests are stored, who can push to
them, or who reviews the change.* Those are the
things real ops teams actually argue about — and they collapse to
one decision: **how many git repos do my environments live in?**

Four shapes you'll see in the wild:

1. **Single repo, branch-as-environment.** `main` is prod, `staging`
   and `dev` are branches. Most common starter shape. **Fights
   GitOps reconciliation** — controllers want to watch a *ref*, not
   a moving branch tip, and "which branch is truth" becomes a
   recurring fight. *Curriculum status: don't use.*
2. **Single repo, directory-as-environment.** `overlays/dev/`,
   `overlays/prod/` siblings (what Modules 2–4 below use as a
   teaching device). Works for a personal project or a spec-bar
   demo. **Does not survive contact with access control on GitHub.**
   See the security note below. *Curriculum status: the starter
   shape; not a real-environment answer.*
3. **Repo per environment.** `movies-dev`, `movies-staging`,
   `movies-prod` — each repo has its own `base/` + `overlays/`, its
   own CODEOWNERS, its own branch-protection rules, its own list
   of who has push access. **Curriculum default.** Simple, scales
   predictably, secures cleanly. Bart's KISS rule applies: keep it
   simple so it scales.
4. **App-config + fleet-config split.** Two-tier: an `app-config`
   repo with the shared base, and one `fleet-config` repo per
   security boundary with overlays + per-cluster pinning. The
   shape big platform teams converge on eventually. **Advanced;
   usually not worth the effort.** Don't reach for it until you
   have N services × M environments *and* proven copy-paste pain.

#### Why GitHub's permission model forces shape #3

GitHub's access controls are **per-repo, not per-branch or
per-directory.** `CODEOWNERS` plus branch protection can *require*
review from a specific team — they cannot *deny read or write* to
specific paths. If your prod-ops team has push to the repo, they
have push to dev; if your dev team has push to the repo, they have
push to prod. The "review gate" is real; the "security boundary"
is not.

For anything sensitive — production secrets, customer data,
compliance scope — you **must** split repos. Branch-protection-as-
security is a control-theater pattern, not a real boundary.

This is the load-bearing reason the curriculum picks repo-per-env
as the default rather than the (Kustomize-correct but operationally
wrong) single-repo + sibling-overlays shape Modules 2–4 use to
teach the *transforms*.

#### Why Modules 2–4 still use single-repo overlays as the example

Kustomize's *primitives* are the same regardless of repo topology.
A base is a base; an overlay is an overlay; `kustomize build` works
the same. Teaching the transforms inside one repo keeps the labs
bounded — you can `cd repos/movies-bartr/deploy/movies` and run
things without coordinating across three repos.

The move from "sibling overlays in one repo" to "one overlay per
repo, three repos" is mechanical. The `base/` either gets copied
into each environment repo, vendored via
`resources: - git::https://...//base?ref=v1.0.0`, or (advanced)
lifted into a separate app-config repo. The *contents* of the
overlays don't change.

#### Example

The repo-per-env default for movies:

```
• github.com/<org>/movies-dev          # write: dev team
  └─ base/        + overlays/dev/

• github.com/<org>/movies-staging      # write: dev team + sre-staging
  └─ base/        + overlays/staging/

• github.com/<org>/movies-prod         # write: sre-prod only
  └─ base/        + overlays/prod/
```

Each repo has its own CODEOWNERS, its own branch protection,
its own audit log. The dev team can ship dev without an
SRE-prod approval; the SRE-prod team can lock prod without
blocking dev velocity.

#### Lab

1. List the repos you currently work in. For each one that holds
   k8s manifests, identify the *shape* (one of the four above).
2. For any single-repo shape, write down the answer to: "who has
   push to this repo, and is that the same set of people who
   should be allowed to deploy to *each* environment in it?" If
   the answers differ, you have a control-theater pattern.
3. Sketch what a repo-per-env split would look like for one of
   your services. Three repo names, three CODEOWNERS files.
4. *Don't actually do the split as part of this lab.* Recognizing
   it is the lab.

#### Knowledge check

1. Why does GitHub's permission model force the repo-per-env
   shape for anything that needs a real security boundary?
2. Branch protection requires reviewers but doesn't deny pushes.
   Why is that a review gate but not a security boundary?
3. The app-config + fleet-config split is the "next next" pattern.
   What specific pain motivates the move from repo-per-env to a
   two-tier split?
4. Modules 2–4 use sibling overlays in one repo to teach the
   *transforms*. What's the move from there to the real shape?
   Why is the move mechanical, not architectural?
5. "KISS — keep it simple so it scales." Defend or attack this
   slogan in the context of repo topology choices.

#### Forward pointer

Repo-per-env is also the foundation for **ring-based deployments**
— the canonical GitOps story (Ring 0 → Ring 1 → … → Ring N
promotion, `git revert` as rollback). Rings live one layer up from
the single-repo Kustomize shape Modules 2–4 cover and are the
centerpiece of the forthcoming GitOps guide (domain E of
[skills-inventory.md](skills-inventory.md)). Don't try to bolt
rings onto a single-repo shape; the directory layout has to support
them from day one, and that conversation belongs in the GitOps
guide.

---

### Module 1 — Why Kustomize, why not Helm (spec §8)

#### Concept

Spec §8 is explicit: **Kustomize only, Helm forbidden.** The reason is
not religious; it's about the mental model.

- **Helm** is a *templating* engine. A chart is YAML with `{{ .Values }}`
  holes; you render at install time with a values file. The rendered
  manifest is not the source of truth; the chart + values is.
- **Kustomize** is a *transformation* engine. There is no template
  language. Every input is already a valid Kubernetes manifest; an
  overlay is a set of mechanical edits (`namePrefix`, `images`, JSON
  patches) applied on top. The rendered manifest is the source of
  truth; the base + overlay produces it deterministically.

The Helm choice creates a class of bugs that GitOps reconciliation
can't see — your repo says `{{ .Values.image.tag }}`, the cluster says
`v0.3.7`, and "drift" is undefined because the chart isn't an object
the cluster understands. Kustomize keeps everything as real manifests
end-to-end, which is the contract Flux / Argo / `kubectl apply` all
already speak.

Spec-level: pick Kustomize because the spec says so. Enterprise-level:
**be able to defend it in a room with a Helm shop.**

#### Example

The same Deployment, expressed two ways.

Helm (forbidden by spec):
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.name }}
spec:
  replicas: {{ .Values.replicas }}
  template:
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repo }}:{{ .Values.image.tag }}"
```

Kustomize (spec-compliant):
```yaml
# base/deployment.yaml — a real, valid manifest with sensible defaults
apiVersion: apps/v1
kind: Deployment
metadata:
  name: movies-api
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: movies-api
          image: movies-api:dev
```
```yaml
# overlays/prod/kustomization.yaml — transforms, not templates
resources:
  - ../../base
replicas:
  - name: movies-api
    count: 3
images:
  - name: movies-api
    newName: ghcr.io/bartr/movies-api
    newTag: 1.0.0
```

#### Lab

```bash
cd repos/movies-bartr/deploy/movies
kustomize build base | head -40
kustomize build overlays/dev | head -40
diff <(kustomize build base) <(kustomize build overlays/dev)
```

Then:

1. Add a second overlay `overlays/bench/` whose only change is
   `replicas: 2` and a different `images.newTag`. Use the existing
   `dev` overlay as the template.
2. `kustomize build overlays/bench`. Confirm two replicas and the
   new tag in the output.
3. Try the same change as a templating exercise on paper — what
   value-file syntax would Helm need, and where would the source of
   truth live?

#### Knowledge check

1. Why does the spec forbid Helm specifically? Give an answer that
   doesn't reduce to "Bart said so."
2. What's the difference between a *template* and a *transformation*?
3. If Flux is reconciling a Helm chart, what does "drift" mean? What
   does it mean when Flux is reconciling a Kustomize tree?
4. Name a real situation where Helm is the right answer despite the
   above (hint: charts you don't own).

---

### Module 2 — Base + overlay fundamentals (spec §8)

#### Concept

A `base/` is a set of *complete, sensible-default* manifests. It must
render cleanly on its own (`kustomize build base/` produces valid YAML
the cluster will accept). An `overlay/` consumes one or more bases via
`resources:` and applies transforms — `namespace`, `namePrefix`,
`commonLabels`, `replicas`, `images`, `patches`.

The base is **not** an abstract template with holes; it's the
simplest-valid-deployment. The overlays are the *deltas* between that
and what each environment actually needs. This is the inverse of
Helm's "everything is configurable, defaults live in values.yaml" —
in Kustomize, defaults live in the *base manifest itself*.

**Spec-bar overlay** (what the spec requires, ~ what movies-bartr ships
today):
```
deploy/movies/
├── base/
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── servicemonitor.yaml
│   └── networkpolicy.yaml
└── overlays/
    └── dev/
        └── kustomization.yaml   # ../../base, nothing else
```

This is enough for the spec. It is **not** enough for a second
environment. See Modules 3 and 4.

#### Example

Minimal valid base, taken from movies-bartr:

```yaml
# deploy/movies/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: movies

resources:
  - namespace.yaml
  - deployment.yaml
  - service.yaml
  - ingress.yaml
  - servicemonitor.yaml
  - networkpolicy.yaml

labels:
  - includeSelectors: false
    pairs:
      app.kubernetes.io/part-of: movies
```

Minimal overlay:

```yaml
# deploy/movies/overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base
```

The dev overlay is currently identical to base. Per the comment in
the real file: *"Kept as a seam for later sessions."* That seam is
what Module 3 fills in.

#### Lab

In `repos/movies-bartr/deploy/movies/`:

1. `kustomize build base/` and read every resource it emits. For each
   one, identify which file produced it and which transform (if any)
   the base's `kustomization.yaml` applied.
2. Notice the `app.kubernetes.io/part-of: movies` label on every
   resource. Find where it was set.
3. Remove the `namespace: movies` line from the base's
   `kustomization.yaml`. Re-build. Where does the namespace go now?
   What breaks?
4. Put it back.

#### Knowledge check

1. Why must the base render cleanly on its own?
2. What's the difference between `commonLabels` (deprecated) and
   `labels:` with `includeSelectors`? When does each one bite you?
3. What does `namespace: movies` at the kustomization level *actually*
   do? Does it edit your manifests or override them at apply time?
4. If the base sets a default image of `movies-api:dev`, what does
   "the dev overlay is identical to the base" tell you about how the
   tag is being managed today?

---

### Module 3 — Overlay design that scales (the Matt-v1 gap)

> **This is the module Matt-v1 missed.** The agent shipped what the
> spec required — one base, one overlay. The operator who can only
> read that pattern can't tell when it's about to fall over.
>
> *Topology reminder (Module 0):* the `overlays/dev` + `overlays/bench`
> + `overlays/prod` siblings used below are a *teaching device* for
> the Kustomize transforms. In a real org, each of those overlays
> would live in its own repo (`movies-dev`, `movies-bench`,
> `movies-prod`), each with its own CODEOWNERS and branch
> protection. The transforms are identical; the repo layout is
> different. Read Module 0 if you haven't.

#### Concept

This module covers the Kustomize-side question: given that you have
multiple environments (in this teaching example, sibling overlays;
in reality, one overlay per repo per Module 0), what transforms
belong where? The single-overlay shape stops working the moment you
have **two or more environments that differ by more than one
thing.** Real differences across environments:

- **Image tag and registry** — `dev` pulls from a local registry; `prod`
  pulls from `ghcr.io`. *Use `images:` transform.*
- **Replica count** — 1 in dev, 3 in prod, 10 in prod-eu. *Use
  `replicas:` transform.*
- **Resource requests/limits** — bigger in prod. *Use a patch.*
- **Per-environment ConfigMap values** — `MOVIES_LOG_LEVEL=debug` in
  dev, `info` in prod. *Use `configMapGenerator` + an overlay patch.*
- **Environment-specific resources** — `dev` has a Grafana admin
  Secret with a weak password; `prod` reads from External Secrets.
  *Add to `resources:` in the overlay.*
- **Different cluster shape** — `bench` has a NetworkPolicy that
  allows the load generator; `prod` doesn't. *Patch or component.*

When you have 2–5 environments, **per-environment overlays** are the
right answer. When you have 10–100 environments that share most of
their config, you need **components** (Module 4).

The shape you want at this level:

```
deploy/movies/
├── base/
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── servicemonitor.yaml
│   └── networkpolicy.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   ├── grafana-admin-secret.yaml
    │   └── patches/
    │       └── deployment-log-level.yaml
    ├── bench/
    │   ├── kustomization.yaml
    │   └── patches/
    │       ├── deployment-replicas.yaml
    │       └── networkpolicy-allow-loadgen.yaml
    └── prod/
        ├── kustomization.yaml
        └── patches/
            ├── deployment-resources.yaml
            └── deployment-replicas.yaml
```

Notice: **`base/` does not change** when you add `bench/` or `prod/`.
Adding an overlay is additive; that's the property you're protecting.

#### Example

Strategic-merge patch for resource limits in prod:

```yaml
# overlays/prod/patches/deployment-resources.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: movies-api
spec:
  template:
    spec:
      containers:
        - name: movies-api
          resources:
            requests:
              cpu: 500m
              memory: 256Mi
            limits:
              cpu: 2000m
              memory: 1Gi
```

JSON6902 patch — surgical, addresses a specific path:

```yaml
# overlays/prod/kustomization.yaml
patches:
  - target:
      kind: Deployment
      name: movies-api
    patch: |-
      - op: replace
        path: /spec/template/spec/containers/0/env/0/value
        value: info
```

The trade-off: **strategic merge** reads like the manifest it patches
(easier to review); **JSON6902** is precise and survives upstream
schema changes better (harder to read).

#### Lab

Build a real two-overlay tree on top of movies-bartr:

```bash
cd repos/movies-bartr/deploy/movies
cp -r overlays/dev overlays/bench
```

1. Edit `overlays/bench/kustomization.yaml` so its `images.newTag` is
   different from `dev`.
2. Add a patch under `overlays/bench/patches/` that sets `replicas: 3`
   on the Deployment.
3. `kustomize build overlays/dev | grep -E 'image:|replicas:'`
4. `kustomize build overlays/bench | grep -E 'image:|replicas:'`
5. Confirm both build cleanly and the only differences between the
   outputs are the two things you changed.
6. Now break it: introduce a typo in the patch's `name:` field.
   Re-build. Read the error. Fix it.

#### Knowledge check

1. Name three things that differ between dev, bench, and prod for a
   well-behaved service. For each, name the Kustomize transform
   you'd reach for.
2. When would you reach for a strategic-merge patch vs a JSON6902
   patch? Give a real example of each.
3. Why does `base/` stop changing once you have multiple overlays?
   What does that property buy you?
4. Matt-v1 has one overlay that is identical to base. What's the
   smallest second-environment requirement that would force a
   restructure, and why?

---

### Module 4 — Components and the 1K-cluster path

#### Concept

Per-environment overlays stop working when you have 10+ environments
that share *most* of their config but mix-and-match small features.
Examples:

- 800 edge clusters, most identical, some with GPU node pools, some
  without, some with a regional Prometheus, some without.
- A multi-tenant SaaS where every tenant gets the same base, but a
  matrix of feature flags determines which add-ons they get.
- A retail chain where each store cluster needs the base service plus
  a per-region NetworkPolicy plus an optional in-store-printer
  sidecar.

Kustomize **components** are the lever. A component is a mix-in:
a directory with its own `kustomization.yaml` that uses
`kind: Component` instead of `Kustomization`, and an overlay can
include multiple components.

```
deploy/movies/
├── base/
├── components/
│   ├── gpu-affinity/
│   │   ├── kustomization.yaml         # kind: Component
│   │   └── patches/...
│   ├── regional-prometheus/
│   ├── external-secrets/
│   └── high-replicas/
└── overlays/
    ├── edge-store-default/
    │   └── kustomization.yaml         # uses 0 components
    ├── edge-store-gpu/
    │   └── kustomization.yaml         # components: [gpu-affinity]
    └── prod-region-eu/
        └── kustomization.yaml
        # components: [regional-prometheus, external-secrets, high-replicas]
```

Without components: 800 overlays, each a copy of the next with one
line different.

With components: 800 overlays that are 3 lines each, composed of
~10 reusable components.

This is the lever the *spec doesn't require* and the *enterprise
absolutely does*. The cllm repo's `clusters/z01/` is the working
example in a Bart-owned MIT repo — every cluster gets its own
directory, but the actual content composes from shared pieces.

#### Example

A component that adds external-secrets wiring:

```yaml
# components/external-secrets/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component

resources:
  - external-secret-grafana.yaml

patches:
  - target:
      kind: Deployment
      name: movies-api
    patch: |-
      - op: add
        path: /spec/template/spec/containers/0/envFrom
        value:
          - secretRef:
              name: movies-api-secrets
```

An overlay that uses two components:

```yaml
# overlays/prod-eu/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

components:
  - ../../components/external-secrets
  - ../../components/regional-prometheus

images:
  - name: movies-api
    newName: ghcr.io/bartr/movies-api
    newTag: 1.0.0
```

The overlay is ~10 lines. The complexity lives in components, each
of which is independently reviewable and version-controlled.

#### Lab

In `repos/movies-bartr/deploy/movies/`, restructure the dev overlay
into a components-based shape:

1. Create `components/grafana-admin-secret/` with a `kustomization.yaml`
   (`kind: Component`) that adds the secret as a `resources:` entry.
2. Refactor `overlays/dev` to consume the component instead of having
   the secret inline (if it currently does) or as a separate
   `resources:` entry.
3. Create a second overlay `overlays/dev-no-secret/` that does *not*
   use the component (simulates a teammate's machine where the
   secret comes from `.env`).
4. `kustomize build` both. Confirm only one has the secret.
5. **Read the diff** between `overlays/dev` and `overlays/dev-no-secret`
   on disk — should be a single `components:` line.

#### Knowledge check

1. What's the difference between a `Component` and a `Kustomization`?
   When would you reach for which?
2. Why does adding a component to an overlay *not* require changing
   the base?
3. The cllm `clusters/z01/` tree has one directory per cluster.
   Sketch how you'd evolve it for 100 clusters using components.
   What stays in the per-cluster directory? What moves to components?
4. Components have an apply-order. Why does that matter? When would
   it bite you?

---

### Module 5 — Image tag management for the inner loop and GitOps

#### Concept

There are three different image-tag stories, and the operator must
not conflate them:

1. **Inner loop (spec §12).** Bump the version, build the image,
   deploy. The tag changes every iteration. The inner loop wants
   the tag managed *in one place* — the `images:` transform in the
   active overlay — and **never** wants `imagePullPolicy: Always`
   with a `:latest` tag, because that defeats the "verify `/version`"
   step.
2. **CI/CD push.** A pipeline builds the image, tags it semver-style
   (`1.2.3` or `1.2.3-sha1234`), pushes to a registry, and updates
   the repo's overlay file (commit + PR or direct commit on a
   release branch).
3. **GitOps image automation.** A controller (Flux's `ImageRepository`
   + `ImageUpdateAutomation`, Argo Image Updater) watches the
   registry, finds a new tag matching a policy, and commits the new
   tag back to the repo. The cluster picks up the change at the
   next reconcile.

Spec bar: own story 1. Enterprise bar: have an opinion on 2 and 3,
and know that "1K clusters" only works with story 3.

**Anti-patterns to recognize:**

- `image: movies-api:latest` in any manifest. Breaks `/version`
  verification.
- `imagePullPolicy: Always` to "fix" a stale image. Masks a tag
  discipline problem.
- Tag bumps done by `sed` in CI rather than by `kustomize edit set
  image`. Works until it doesn't (e.g. matching `1.0.0` substring
  inside a different value).
- One overlay per image tag (`overlays/v1.0.0`, `overlays/v1.0.1`).
  Version is not an environment.

#### Example

`kustomize edit` from the command line (what CI should run):

```bash
cd overlays/prod
kustomize edit set image movies-api=ghcr.io/bartr/movies-api:1.2.3
# updates overlays/prod/kustomization.yaml deterministically
```

Resulting overlay:

```yaml
images:
  - name: movies-api
    newName: ghcr.io/bartr/movies-api
    newTag: 1.2.3
```

Flux image automation marker (declarative, the controller updates
the tag here):

```yaml
images:
  - name: movies-api
    newName: ghcr.io/bartr/movies-api
    newTag: 1.2.3 # {"$imagepolicy": "flux-system:movies-api"}
```

#### Lab

```bash
cd repos/movies-bartr/deploy/movies/overlays/dev
kustomize edit set image movies-api=movies-api:0.9.9
git diff kustomization.yaml
kustomize build . | grep image:
# put it back
kustomize edit set image movies-api=movies-api:dev
```

Then:

1. Add `imagePullPolicy: Always` to the base Deployment. Re-deploy.
   Inspect what changed in pull behavior.
2. Revert. Set the tag to `:latest` in the overlay's `images:`.
   `kustomize build .` and confirm what the rendered manifest now
   says. Note how `/version` would behave after a rebuild without a
   tag change.
3. Revert. This is the failure mode you want to recognize cold.

#### Knowledge check

1. Why is `:latest` + `imagePullPolicy: Always` a spec-bar violation
   even though no rule literally forbids it?
2. What's the difference between `kustomize edit set image` and
   `sed -i s/0.9.0/0.9.1/g`? Name a case where they produce
   different results.
3. In a Flux-managed cluster, who is allowed to commit a new image
   tag to the repo — a human, CI, the image automation controller,
   or all three? What's the operational consequence of each choice?
4. If you see `overlays/v1.0.0/`, `overlays/v1.0.1/`, `overlays/v1.0.2/`
   in someone's repo, what's wrong and what's the fix?

---

### Module 6 — The single multi-stage Dockerfile (spec §9)

#### Concept

Spec §9 requires **one** multi-stage `Dockerfile` producing a minimal
runtime image, running as non-root. For Go (domain M of the inventory),
the target is a static binary on `distroless/static` or `scratch`,
landing at ~15MB.

The multi-stage pattern:

1. **Builder stage** — full Go toolchain. `go mod download` first
   (cacheable layer), then `go build` with the right flags.
2. **Runtime stage** — minimal base, `COPY --from=builder` the binary,
   set `USER`, set `ENTRYPOINT`.

Anti-patterns to recognize:

- Single-stage build → 800MB image with the entire Go toolchain.
- `FROM golang:1.22` as the runtime → same problem.
- `USER root` (or no `USER` at all) → spec §13 violation.
- `COPY . .` before `go mod download` → cache busted on every source
  change.
- Building inside CI without `--build-arg` for the version → version
  metadata baked wrong.

The Dockerfile is also a Kustomize concern because the image *tag*
the Dockerfile produces is what the overlay's `images:` references.
A bad Dockerfile breaks the inner loop just as surely as a bad
manifest.

#### Example

Canonical Go Dockerfile, distroless, non-root, ~15MB output:

```dockerfile
# syntax=docker/dockerfile:1.7

# ---- builder ----
FROM golang:1.22-alpine AS builder
WORKDIR /src

# cache mod download as its own layer
COPY go.mod go.sum ./
RUN go mod download

COPY . .

ARG VERSION=dev
RUN CGO_ENABLED=0 GOOS=linux go build \
      -trimpath \
      -ldflags="-s -w -X main.version=${VERSION}" \
      -o /out/movies-api ./cmd/movies-api

# ---- runtime ----
FROM gcr.io/distroless/static:nonroot
WORKDIR /app
COPY --from=builder /out/movies-api /app/movies-api
USER 65532:65532
EXPOSE 8080
ENTRYPOINT ["/app/movies-api"]
```

Build:

```bash
docker build --build-arg VERSION=0.9.0 -t movies-api:0.9.0 .
docker images movies-api:0.9.0
# expect ~15-20 MB
```

#### Lab

In `repos/movies-bartr/src/`:

1. Read the existing Dockerfile end to end. For each line, identify
   which spec requirement it satisfies (§9 non-root, §13 read-only
   FS, etc.).
2. Build the image. Run `docker images` and confirm the size.
3. Comment out the `-ldflags="-s -w"` and rebuild. Compare sizes.
4. Change the runtime base from `distroless/static` to `golang:1.22-alpine`
   (just for the lab — don't commit). Rebuild. Compare sizes. Note
   the multiplier.
5. Put everything back. Confirm a clean build still produces the
   small image.

#### Knowledge check

1. What does `-ldflags="-s -w"` do? What's the trade-off when you set
   it? (Hint: stack traces.)
2. Why is `COPY go.mod go.sum ./ && RUN go mod download` *before*
   `COPY . .`? What breaks if you reverse them?
3. `distroless/static:nonroot` runs as UID 65532. How does this
   interact with the `securityContext` you set in the Deployment?
   What happens if they disagree?
4. The spec also requires a **read-only root filesystem.** What
   changes in the Dockerfile (if anything) and what changes in the
   Deployment to make that work?

---

### Module 7 — Local-cluster ingress: LoadBalancer in dev, IngressRoute in prod

> **Standard for this curriculum.** Every lab in every study guide
> assumes services are reachable at `http://localhost:<port>` without
> `kubectl port-forward`. This module is how that gets wired, and how
> the same overlay shape switches to a production-correct
> `IngressRoute` in `overlays/prod`.

#### Concept

There are three local-cluster patterns for reaching a service
from the host browser or `curl`:

1. **`kubectl port-forward`** — a `kubectl`-only tunnel. *Discouraged.*
   Two-terminal tax, hides Service / Ingress correctness, doesn't
   exist in production. The K8s inventory (row C16) calls it a smell.
2. **`Service: type: NodePort`** — the cluster exposes a high port
   (30000–32767) on every node. Works but the port numbers are
   ugly and you stop being able to use `:8080`, `:3000`, `:9090`.
3. **`Service: type: LoadBalancer` on k3s/k3d** — k3s ships
   `klipper-lb`, a real LoadBalancer implementation that binds the
   Service's port on the node IP. k3d extends this with
   `--port 8080:8080@loadbalancer` at cluster create time. **This
   is the standard.**

The production pattern is different: in a real cluster, `type: LoadBalancer`
provisions a cloud LB (real money, real DNS, real cert). Public entry
is a Traefik `IngressRoute` (or Ingress) terminating TLS, with
internal services as plain `ClusterIP`. That's a Kustomize
overlay difference — same base, different Service type and an extra
`IngressRoute` resource in `overlays/prod`.

#### Example

**k3d cluster bring-up** — once per workstation:

```bash
k3d cluster create movies \
  --port 8080:8080@loadbalancer \
  --port 3000:3000@loadbalancer \
  --port 9090:9090@loadbalancer \
  --k3s-arg "--disable=traefik@server:0" # only if you install your own
  # (omit the --disable line to use the bundled Traefik)
```

**Base Service** — ClusterIP by default, production-safe:

```yaml
# base/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: movies-api
spec:
  type: ClusterIP
  selector:
    app: movies-api
  ports:
    - name: http
      port: 8080
      targetPort: 8080
```

**Dev overlay** — promote to LoadBalancer for local convenience:

```yaml
# overlays/dev/patches/service-loadbalancer.yaml
apiVersion: v1
kind: Service
metadata:
  name: movies-api
spec:
  type: LoadBalancer
```
```yaml
# overlays/dev/kustomization.yaml
resources:
  - ../../base
patches:
  - path: patches/service-loadbalancer.yaml
```

**Prod overlay** — keep ClusterIP, add an IngressRoute:

```yaml
# overlays/prod/ingressroute.yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: movies-api
spec:
  entryPoints: [websecure]
  routes:
    - match: Host(`movies.example.com`)
      kind: Rule
      services:
        - name: movies-api
          port: 8080
  tls:
    secretName: movies-api-tls
```
```yaml
# overlays/prod/kustomization.yaml
resources:
  - ../../base
  - ingressroute.yaml
# no service-type patch — base ClusterIP is correct for prod
```

Grafana and Prometheus get the same treatment in their own bases /
overlays — ClusterIP in base, LoadBalancer patch in dev on ports
3000 and 9090 respectively.

#### Lab

1. If your cluster is already running without the LB port maps, tear
   it down and recreate with the bring-up command above.
2. In `repos/movies-bartr/deploy/movies/`, add the
   `service-loadbalancer.yaml` patch to `overlays/dev/` and reference
   it from the overlay's `kustomization.yaml`.
3. `kubectl apply -k overlays/dev`. Then
   `kubectl get svc -n movies movies-api` — confirm `TYPE` is
   `LoadBalancer` and `EXTERNAL-IP` is populated.
4. `curl -s http://localhost:8080/healthz` — must succeed with no
   port-forward.
5. Build a stub `overlays/prod/` with the IngressRoute from the
   example and an `images:` override. `kustomize build overlays/prod
   | grep -E '^kind:|type:'` — confirm the Service is `ClusterIP`
   and an `IngressRoute` was added.
6. Anti-pattern recognition: run `kubectl port-forward -n movies
   svc/movies-api 8080:8080` in a second terminal. Notice that *it
   still works* even with the LB in place. The lab passes either way;
   that's exactly why the habit persists. Kill it.

#### Knowledge check

1. Why does the base Service stay `ClusterIP`? What would change if
   you put `type: LoadBalancer` in the base instead of patching it
   into dev?
2. On a managed cluster (EKS / AKS / GKE), what happens when you
   `kubectl apply` a Service with `type: LoadBalancer`? Why is that
   not free in prod?
3. The k3d bring-up command maps host port 8080 to LB port 8080.
   What happens if you have two services that both want port 8080
   in dev? How would you resolve it?
4. The prod overlay uses `IngressRoute` (Traefik CRD), not
   `Ingress`. Why might you prefer the CRD? When would you regret it?
5. `port-forward` works *even when the Service is misconfigured.*
   Name a specific Service bug that `port-forward` would mask but
   `LoadBalancer` would surface immediately.

---

### Module 8 — Tying it together: spec §12 inner loop end-to-end

#### Concept

Modules 1–7 are pieces. The §12 inner loop is the test that says
they fit:

1. Make a change (source, manifest, or data).
2. Bump the version.
3. Build the image (Module 6).
4. Deploy the new version (Modules 2–5 — overlay + image tag).
5. Verify the version is live (`/version`) over the dev LB (Module 7).
6. Run validation tests.
7. Inspect metrics on the Grafana dashboard
   ([study-guide-observability.md](study-guide-observability.md)
   Module 6).
8. Iterate.

The Kustomize-and-Docker half of the loop is steps 2–5. If those
steps don't run cleanly from a fresh clone, the inner loop is broken
even if the code is correct. **That's the bar.**

#### Example

The full sequence, copy-paste ready, from movies-bartr (assumes the
cluster is up per Module 7):

```bash
# 1. change something (e.g. a handler)
# 2. bump version
make bump-patch         # or edit cmd/movies-api/main.go's version const

# 3. build (tag matches the bump)
docker build --build-arg VERSION=0.9.1 -t movies-api:0.9.1 .

# 4. point the overlay at the new tag and apply
cd deploy/movies/overlays/dev
kustomize edit set image movies-api=movies-api:0.9.1
kubectl apply -k .

# 5. verify (no port-forward — dev LB on :8080 from Module 7)
kubectl rollout status -n movies deploy/movies-api
curl -s http://localhost:8080/version    # expect 0.9.1

# 6. validate
make test-e2e

# 7. inspect (see observability guide — http://localhost:3000)
```

#### Lab

End-to-end, on a fresh clone, in one session:

1. `git clone` your fork of movies-bartr into a scratch directory.
2. Bring the cluster up with the Module 7 port map.
3. Follow only the implementation README — no other source. Reach
   a working step 7 (Grafana panel shows live traffic at
   <http://localhost:3000>).
4. Now break it deliberately: change the overlay to reference a tag
   you didn't build. Re-apply. **Read the failure mode.** Is it
   `ImagePullBackOff`, `ErrImagePull`, something else? Where do you
   look first?
5. Fix it. Confirm `/version` returns the right semver again over
   the LB.

#### Knowledge check

1. The README is the spec's contract for "fresh clone works." Where
   would Kustomize step 4 fail silently if the README skipped
   `kustomize edit set image`?
2. What's the difference between `ImagePullBackOff` and `ErrImagePull`?
   What signals does each give you?
3. The inner loop says "verify `/version`." What three things have
   to be right for `/version` to return the *new* semver?
4. If you skipped step 7 (Grafana inspection), you'd still ship a
   working artifact. What does Module 7 of the observability guide
   say you'd be missing?
5. The Module 7 LB binding lets you `curl http://localhost:8080`
   directly. What's the single failure mode this rules out vs the
   `port-forward` version of the same step?

---

## Per-release review

Same template as the observability guide; see the
[Per-release review template section](study-guide-observability.md#per-release-review-template).
The only Kustomize-specific addition: when a release touches Module 3
or Module 4, the hands-on check **must** include `kustomize build
overlays/<env>` on the touched overlay and a read-through of the
output. The most common failure mode is "the YAML looks right and the
rendered manifest is wrong"; you only catch it by reading what was
actually generated.

## What this guide is and is not

- **Is:** a Kustomize curriculum bounded by what movies-spec
  requires plus the next-step enterprise pattern (components +
  GitOps-friendly tag automation) that the spec doesn't require but
  every team eventually needs.
- **Is not:** a Helm reference. Helm is forbidden by spec; the
  operator should know enough about it to defend the choice, not
  enough to ship one.
- **Is not:** a Flux deep-dive. Image automation is mentioned in
  Module 5 because the tag story spans both; the full GitOps
  curriculum is a separate guide (domain E of
  [skills-inventory.md](skills-inventory.md)).

## Translation notes (per inventory domain M)

This guide uses Go for the Dockerfile in Module 6. The
Kustomize content (Modules 1–5, 7) is language-agnostic — the
manifests don't care what's inside the container. For other stacks,
the only thing that changes is Module 6's Dockerfile:

- **Rust + axum** — multi-stage with `rust:1-alpine` builder,
  runtime on `distroless/cc-debian12` (musl + static) or `scratch`.
  Comparable image size (~20MB), longer build time.
- **Node + fastify** — builder with `node:20-alpine`, runtime on
  `gcr.io/distroless/nodejs20-debian12`. ~150MB; trade-off is
  ecosystem reach.
- **.NET minimal API** — builder with `mcr.microsoft.com/dotnet/sdk:8.0`,
  runtime on `mcr.microsoft.com/dotnet/runtime-deps:8.0-jammy-chiseled`.
  ~110MB; "chiseled" is the .NET version of distroless.
- **Python + FastAPI** — builder with `python:3.12-slim`, runtime on
  `gcr.io/distroless/python3-debian12`. ~80MB; watch out for
  C-extension wheels.

The operator running a non-Go stack should run the prompt:
*"Translate Module 6's Dockerfile to <stack>. Keep the spec contract
identical: multi-stage, non-root, minimal runtime, version baked at
build time."*

## Open questions

- Module 4 (components) is the most opinionated module here.
  Should it be its own guide instead, alongside the GitOps guide
  (domain E)? Trade-off: smaller guides are easier to consume; one
  larger guide keeps the Kustomize mental model intact.
- The Module 5 image-automation content overlaps with the future
  GitOps guide. Cleanest cut is probably "spec-bar tag management
  stays here; GitOps automation moves to E." Worth deciding before
  E is drafted.
- The translation notes section is new (didn't exist in the
  observability guide because §7 didn't have a Dockerfile module).
  If it stays useful, it should be backfilled into
  [study-guide-observability.md](study-guide-observability.md)'s
  modules 1 and 2 (metrics client lib, structured logging) where
  the language *does* matter.

## Status

- Not yet run end-to-end with any operator.
- Not yet validated against the
  [sessions-and-skill-compounding](../methodology/drafts/sessions-and-skill-compounding.md)
  hypothesis.
- Unlocks for promotion to `methodology/` once at least one full
  run has used it and reported honestly on whether it closed the
  Kustomize-shaped piece of the learning gap without inflating
  focus time beyond the artifact's value.
