# Study Guide — GitOps in the Claude Era (domain E)

> **DRAFT — NOT FOR PUBLICATION.** Fourth instance of the study-guide
> format established by
> [study-guide-observability.md](study-guide-observability.md),
> [study-guide-kustomize.md](study-guide-kustomize.md), and
> [study-guide-go.md](study-guide-go.md). Scoped from domain E of
> [skills-inventory.md](skills-inventory.md). Builds on
> [study-guide-kustomize.md](study-guide-kustomize.md) Module 0
> (repo-per-env is the curriculum default).

## Why this exists

Movies-spec doesn't require GitOps. Enterprise does. The gap is real:
the moment you have more than one cluster, or more than one
environment, or any kind of audit obligation, you need a
*single source of truth* that is *not* the cluster. That's GitOps.

Two things have changed since the last serious wave of GitOps
adoption (2019–2023):

1. **Claude can read your directory structure and reason about it.**
   The "we adopted Flux because tracking manifests by hand was
   infeasible" rationale is much weaker than it was. A standardized
   directory layout + an LLM that understands it covers a surprising
   amount of what tools like Argo Image Updater were built for.
2. **Ring-based promotion is now table-stakes** for any
   multi-cluster fleet. Promotion forward (Ring 0 → 1 → 2 → 3) and
   `git revert` for rollback is the canonical contract, regardless
   of which controller you run underneath.

This guide teaches both — the *contract* (what GitOps actually
guarantees), and the *post-Claude tooling reality* (you have more
choices than "Flux or Argo," and the directory shape matters more
than the controller).

**What this guide is anchored in:**

- Spec §8 (Kustomize-only, no Helm) and §13 (no secrets in repos).
- [study-guide-kustomize.md](study-guide-kustomize.md) Module 0
  (repo topology — repo-per-env as default).
- The cllm repo's `clusters/z01/` tree — a Bart-owned MIT working
  example of a Flux-managed cluster with `GitRepository` +
  `Kustomization` objects.

## How to use this guide

Same protocol as the other guides:

- Pick the module mapping to upcoming work; don't read front-to-back.
- Run the lab with your own hands.
- At each tag, run the per-release review.

**One curriculum-level rule for this guide specifically:** the labs
involve `git revert`, `git push`, and reconciliation against a real
cluster. Practice these on a *scratch* fork before touching anything
you care about. The whole point of GitOps is that `git revert` is
the rollback button. If you've never pulled that lever in practice,
you don't actually have rollback.

## Modules

### Module 1 — Why GitOps changed in 2024

#### Concept

Pre-2024, "we use GitOps" practically meant "we run Flux" or "we run
Argo." The reason was operational: keeping the cluster's actual
state in sync with the repo's declared state was hard, and tools
like Flux did the hard parts (watch a repo, reconcile drift, manage
the order of operations across many resources). Tracking what
version was running where, across dozens of clusters, was infeasible
by hand.

Post-Claude, two things are different:

1. **Reading the repo is free.** "What version of `movies-api` is
   pinned in `ring-1`?" used to require a shell pipeline. Now it's
   a one-line question to Claude. "What changed between `ring-0`
   and `ring-1` in the last week?" used to require git archeology;
   now it's a prompt.
2. **Generating the right commits is free.** "Promote
   `movies-api:0.9.1` from `ring-0` to `ring-1`" used to require
   either careful manual edits or a controller (Argo Image Updater
   etc.) with its own DSL. Now Claude can read your directory
   shape and generate the PR.

What this *doesn't* change:

- **The contract.** The repo is still the source of truth. The
  cluster still reconciles to it. Drift is still measured against
  the repo, not against tribal knowledge.
- **The need for reconciliation.** If you have 1K clusters and no
  humans in the loop, you still need a controller continuously
  reconciling drift. Claude doesn't run on a timer.
- **Security boundaries.** Repo permissions, secrets management,
  audit trails — all unchanged.

What it changes:

- **The tool choice has more options.** Flux and Argo are still
  right at certain scales; below those scales, a 200-line CLI that
  matches your team's workflow is often better than adopting
  someone else's.
- **The directory shape matters more than the controller.** If your
  layout is consistent and discoverable, Claude (or a small
  custom tool, or Flux, or Argo) can all operate on it. If your
  layout is a mess, no controller saves you.

#### Example

The pre-2024 promotion sequence for "deploy `movies-api:0.9.1` to
ring-0":

1. Look up what `ring-0` currently runs (`grep -r` across the fleet
   repo).
2. Find the right file to edit (`clusters/ring-0/movies-api/kustomization.yaml`).
3. Edit the `images:` block, careful not to break YAML.
4. Open a PR.
5. Wait for CI to render the manifest and post the diff.
6. Get review approval.
7. Merge.
8. Wait for Flux to reconcile (or `flux reconcile kustomization`).

The post-Claude equivalent:

> *"Promote `movies-api:0.9.1` from ring-0 only. Don't touch any
> other ring. Open a PR with the diff."*

Claude reads the repo, edits the right file, opens the PR. Steps 5–8
unchanged.

#### Lab

1. Pick a real fleet repo you work in (or fork the cllm repo as a
   scratch).
2. Ask Claude: *"What version of `<service>` is running in each
   cluster? Give me a table."* Compare to what you'd have written by
   hand.
3. Ask Claude: *"What changed in `<cluster-path>` between
   `<tag-A>` and `<tag-B>`?"* Compare to `git log --oneline
   <tag-A>..<tag-B> -- <cluster-path>`.
4. Ask Claude: *"Here's a CHANGELOG entry. Generate the fleet PR
   that promotes the version change to ring-0 only."* Read the
   generated PR. Don't merge — just read.

#### Knowledge check

1. What stays the same about GitOps in the Claude era? Name three
   things.
2. What changes? Name three things.
3. "Adopting a controller is no longer the only path to GitOps."
   Defend or attack.
4. If Claude can read the repo and generate the right commit, why
   would you ever still need Flux or Argo?
5. The directory shape matters more than the controller. Why?

---

### Module 2 — The GitOps contract

> **The thing every GitOps tool you'll ever use agrees on.** If you
> can articulate the contract, you can evaluate any tool against
> it.

#### Concept

GitOps is a four-part contract:

1. **Declarative.** The desired state of the cluster is described
   declaratively (Kubernetes manifests, Kustomize overlays, Helm
   charts — anything that says *what* should exist, not *how* to
   create it).
2. **Versioned and immutable.** The desired state lives in git.
   Every change has an author, a timestamp, a diff, and a SHA.
3. **Pulled automatically.** A controller (or in the post-Claude
   world, sometimes a CLI on a cron, or sometimes nothing
   automatic at all) reconciles the cluster's actual state to the
   repo's declared state. Humans don't `kubectl apply` to
   production.
4. **Continuously reconciled.** Drift between repo and cluster is
   detected and corrected (or at minimum, flagged).

What this gives you:

- **Audit trail.** Every change to prod has a SHA, an author, a PR.
- **Rollback.** `git revert <sha> && git push`. The controller
  reconciles back. *This is the actual rollback button.*
- **Disaster recovery.** Lose the cluster, point the controller at
  the repo, get the cluster back.
- **Drift detection.** Someone `kubectl edit`s in prod? The
  controller reverts (or alerts).

What this requires:

- **Repo as source of truth.** Nobody applies to the cluster
  out-of-band. *Ever.* The moment one person does, the contract is
  broken.
- **A reconciliation loop.** Either a controller, a scheduled CLI,
  or (at minimum) an explicit step in your dev loop that pulls the
  current repo state and applies it.

Anti-patterns that break the contract:

- "Hotfix in prod, then PR" — breaks the audit trail.
- `kubectl apply` in CI without a corresponding repo commit — the
  cluster's state isn't reproducible from the repo alone.
- `kubectl edit` "just this once" — breaks drift detection.
- Helm `--set` overrides in CI — the rendered manifest isn't in the
  repo.

#### Example

A minimal Flux `GitRepository` + `Kustomization`, from
`repos/cllm/clusters/z01/flux-system/`:

```yaml
# source.yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 1m0s
  ref:
    branch: bartr
  url: https://github.com/bartr/vllm.git
```

```yaml
# listeners/vllm.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: vllm
  namespace: flux-system
spec:
  interval: 1m
  path: ./clusters/z01/vllm
  prune: true
  wait: true
  dependsOn:
    - name: nvidia-plugin
  sourceRef:
    kind: GitRepository
    name: flux-system
```

Reads as: "Every minute, look at branch `bartr` of `github.com/bartr/vllm`.
Apply whatever is in `./clusters/z01/vllm/`. Prune things that
disappear from the repo. Wait for nvidia-plugin to be ready first."

Every word in those manifests maps onto one part of the contract:

- `path:` → declarative.
- `ref:` + `interval:` → versioned + pulled automatically.
- `prune: true` → continuously reconciled (drift in the
  "extra-stuff" direction).
- `wait: true` + `dependsOn:` → reconciliation order matters.

#### Lab

1. In your scratch fleet repo, find every place anyone applies
   manifests to prod *not* via the GitOps controller. Count them.
   For each one, identify whether it's a contract violation or a
   legitimate bootstrap concern.
2. Pick one workload in the cllm repo's `clusters/z01/`. Trace what
   would happen if you:
   - `git revert` the last commit that touched its `kustomization.yaml`,
   - then pushed.
   How long until the cluster reverts? What command would you run
   to force it faster?
3. Try the drift-detection test: `kubectl edit deployment` something
   minor (`replicas: 1` → `2`). Wait for Flux's interval. Watch the
   change get reverted. *Confirm it actually got reverted.* If it
   didn't, you don't have GitOps; you have a deployment script.

#### Knowledge check

1. Name the four parts of the GitOps contract.
2. The rollback button is `git revert <sha> && git push`. Why is
   that *better* than `kubectl rollout undo`?
3. `kubectl edit` "just this once" is an anti-pattern. What does it
   break and how would you discover the breakage later?
4. `prune: true` is a small thing with a big consequence. What
   happens if it's `false`?
5. The bootstrap problem: the cluster doesn't exist yet, so the
   GitOps controller can't reconcile it from nothing. How is that
   solved in practice? (Hint: cllm has `scripts/init-flux.sh`.)

---

### Module 3 — Ring-based promotion: the canonical GitOps example

> **The centerpiece of this guide.** Every other module supports
> this one. If you only learn one thing from this guide, learn how
> rings work and why `git revert` is the rollback button.

#### Concept

A **ring** is a set of clusters / regions / tenants that all run
*the same version* and get promoted together. The canonical
sequence:

- **Ring 0** — canary. 1–5 clusters. You watch dashboards in real
  time. If it breaks, it broke on a tiny blast radius.
- **Ring 1** — early adopters. ~10% of the fleet. The first
  detection of "works in canary, breaks at moderate scale" lives
  here.
- **Ring 2** — broad. ~40% of the fleet. By this ring, the version
  is "validated under real load."
- **Ring 3** — everyone else. ~50% of the fleet. Stable for
  customers who never want to be on the new thing first.

(Numbers and ring count are illustrative — different orgs use 2–6
rings. The principle doesn't change.)

**Promotion** is forward, deliberate, and observable:

1. `heartbeat:0.2.0` is built and tested.
2. Commit to fleet repo: bump ring-0's pin from `0.1.0` to `0.2.0`.
3. Reconcile. Watch ring-0 dashboards for the bake period (1 hour?
   1 day? depends on the service).
4. If green: commit again, bump ring-1's pin to `0.2.0`. Bake.
5. Repeat for ring-2, ring-3.

**Rollback** is the inverse commit:

1. Ring-1 starts erroring out 20 minutes after promotion.
2. `git revert <the-promotion-commit-for-ring-1>` (and `ring-0` if
   it's also affected). Push.
3. Reconcile. Ring-1 (and ring-0) goes back to `0.1.0` automatically.
4. No `kubectl rollout undo`. No SSH. No "what did Jane do at
   2am?" The revert SHA is the audit trail.

The thing that makes this work is **per-ring version pinning** in
the fleet repo. Each ring has its own directory with its own
overlay; the overlay sets that ring's image tag explicitly.

#### Example

Directory shape for a 4-ring fleet (one repo per environment per
[Module 0 of the Kustomize guide](study-guide-kustomize.md#module-0--repo-topology-where-do-dev-staging-prod-actually-live);
this example is the `<service>-prod` repo):

```
movies-prod/                            # repo: write = sre-prod only
├── base/
│   └── heartbeat/
│       ├── deployment.yaml             # image: heartbeat (no tag)
│       ├── service.yaml
│       └── kustomization.yaml
└── clusters/
    ├── ring-0/                         # canary
    │   ├── kustomization.yaml          # ../../base/heartbeat
    │   └── version.yaml                # images.newTag: 0.2.0
    ├── ring-1/                         # early adopters
    │   ├── kustomization.yaml
    │   └── version.yaml                # images.newTag: 0.1.0
    ├── ring-2/                         # broad
    │   ├── kustomization.yaml
    │   └── version.yaml                # images.newTag: 0.1.0
    └── ring-3/                         # everyone else
        ├── kustomization.yaml
        └── version.yaml                # images.newTag: 0.1.0
```

Each ring's `kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base/heartbeat
patches:
  - path: version.yaml
```

Each ring's `version.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
images:
  - name: heartbeat
    newName: ghcr.io/<org>/heartbeat
    newTag: 0.2.0          # ring-0
    # newTag: 0.1.0        # ring-1, ring-2, ring-3
```

Per-cluster Flux `Kustomization` watches its own ring's path:

```yaml
# cluster: ring-0-cluster-1
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
spec:
  interval: 1m
  path: ./clusters/ring-0
  sourceRef:
    kind: GitRepository
    name: movies-prod
```

When you bump `clusters/ring-0/version.yaml`, every ring-0 cluster
picks it up at next reconciliation. Ring-1+ keep running `0.1.0`
because their `version.yaml` is unchanged.

#### Promotion in commits

Promote 0.2.0 from ring-0 to ring-1:

```bash
# in movies-prod repo
sed -i 's/newTag: 0.1.0/newTag: 0.2.0/' clusters/ring-1/version.yaml
git add clusters/ring-1/version.yaml
git commit -m "promote heartbeat 0.2.0 to ring-1"
gh pr create --fill
```

Rollback ring-1 after a bad bake:

```bash
git revert <promotion-commit-sha> -m 1
git push
# ring-1 reverts to 0.1.0 automatically at next reconcile
```

#### Lab

In a scratch fork of the cllm repo (or a similar fleet repo):

1. Add a `clusters/ring-{0,1,2,3}/` shape for one workload. Each
   gets its own `version.yaml` with the image tag pinned. Three
   rings pin `0.1.0`; ring-0 pins `0.2.0`.
2. `kustomize build clusters/ring-0` and `kustomize build
   clusters/ring-1`. Confirm the only difference in the rendered
   output is the image tag.
3. Commit and push. Wait for Flux to reconcile (or `flux reconcile
   kustomization`).
4. **The rollback drill.** Promote `0.2.0` to ring-1 with a commit.
   Wait for reconciliation. Now `git revert` that commit and push.
   Watch ring-1 go back to `0.1.0`. Time it.
5. Read the git log on `clusters/ring-1/`. The promotion + revert
   pair is the audit trail. **This is what GitOps is for.**

#### Knowledge check

1. Why is per-ring version pinning the load-bearing piece? What
   breaks if all rings share one `images.newTag` instead?
2. Promotion is "forward only." Why isn't promotion just "set the
   tag in `base/` and let it propagate to everything at once"?
3. `git revert` is the rollback button. Compare it to `kubectl
   rollout undo` on every cluster in ring-1. What's the difference
   in audit trail, in time-to-rollback, in confidence that the
   rollback completed?
4. What's the right *bake time* for ring-0 before promoting to
   ring-1? What signals do you watch during the bake?
5. A bug only shows up at 1000+ RPS. Ring-0 has 5 clusters at 10
   RPS each. Where does the bug first appear, and what's the
   methodology fix?
6. If ring-0 has been on `0.2.0` for a week and ring-1 is still on
   `0.1.0`, what's wrong? (Hint: rings aren't supposed to drift
   indefinitely.)

---

### Module 4 — Flux primitives (when you actually need a controller)

> **The "use Flux" answer is correct at certain scales.** This
> module covers the smallest useful slice of Flux so you can recognize
> when it's the right answer (Module 5 covers when it's not).

#### Concept

Flux is a set of Kubernetes controllers that watch a `GitRepository`
and apply `Kustomization` / `HelmRelease` resources from it. Five
objects cover ~95% of what you'll see:

- **`GitRepository`** — "watch this repo, this branch, on this
  interval."
- **`Kustomization`** — "apply the manifests at this path in the
  watched repo." `prune: true` deletes things removed from git;
  `wait: true` blocks dependent kustomizations.
- **`HelmRelease`** — Helm chart deployments (which the movies
  spec forbids; included here for fleet-level recognition).
- **`ImageRepository`** + **`ImagePolicy`** + **`ImageUpdateAutomation`** —
  the image-automation triad. Watch a registry, pick versions
  matching a policy, commit the bump back to git.
- **Notification** objects — `Alert`, `Provider`, `Receiver`. Slack
  on failed reconciliation, etc.

Two things Flux gives you that a CLI doesn't:

1. **Continuous reconciliation.** Every minute (or whatever
   `interval:` says), Flux re-applies the manifests. Drift in the
   cluster gets corrected automatically. A CLI on a cron does the
   same thing but with worse failure modes.
2. **Dependency ordering across many objects.** `dependsOn:` lets
   you say "don't reconcile cllm until vLLM is ready," and Flux
   respects it without you wiring up the order in shell. cllm's
   `clusters/z01/flux-system/listeners/` is a working example.

Two things Flux does *not* solve for you:

- **Choosing the right directory shape.** Flux watches a path; if
  the path is a mess, Flux applies a mess.
- **Deciding *when* to promote between rings.** That's a human
  decision (or a Claude-assisted one — Module 5).

#### Example

The cllm bootstrap, from `scripts/init-flux.sh`:

```bash
kubectl apply -f components.yaml      # the Flux controllers themselves
kubectl apply -f source.yaml          # the GitRepository

kubectl wait --namespace flux-system --for=condition=Ready pod --all --timeout=5m

# Bring up workloads in dependency order
kubectl apply -f listeners/nvidia-plugin.yaml
flux reconcile kustomization nvidia-plugin

kubectl apply -f listeners/vllm.yaml
flux reconcile kustomization vllm

kubectl apply -f listeners/traefik.yaml
flux reconcile kustomization traefik
# ... etc
```

Each listener YAML is a Flux `Kustomization` object pointed at a
path in the repo. After bootstrap, the human's job is *commit to
the repo* — never `kubectl apply` again.

#### Lab

1. In a scratch k3d cluster, bootstrap Flux pointing at a fork of
   the cllm repo. Use `scripts/init-flux.sh` as the reference, but
   modify the source to point at your fork.
2. After bootstrap, `flux get kustomizations -A`. Read every
   resource. Note the `Ready` status, `Last Applied Revision`,
   `Last Reconciled`.
3. Make a change in the repo (bump a replica count). Push. **Don't
   run `flux reconcile`.** Wait for the next interval. Watch the
   change propagate.
4. Make a change *in the cluster* via `kubectl edit`. Wait for the
   next interval. Watch Flux revert it. **This is the drift-
   detection contract you read about in Module 2.**
5. Break the source: push a YAML syntax error. Watch Flux's
   `Kustomization` status go to `Failed`. Read the error message.
   Revert. Confirm recovery.

#### Knowledge check

1. Name the five most common Flux object types and what each one
   does.
2. `interval: 1m` vs `interval: 10m` — what changes operationally?
   What's the failure mode of each?
3. `prune: true` vs `prune: false` — name a real situation where
   each is the correct choice.
4. The bootstrap problem: how does Flux get *into* a cluster that
   doesn't yet have Flux? Trace cllm's bootstrap.
5. `dependsOn:` is per-Kustomization. What happens when a
   dependency fails? What happens when a dependency is *flaky*?
6. Flux watches a branch (`ref: branch: bartr` in the cllm
   example). What's the trade-off vs `ref: tag: v1.0.0`?

---

### Module 5 — Roll your own: Claude + CLI + a standardized directory shape

> **The post-2024 alternative to "adopt Flux/Argo."** Not always the
> right answer, but often a *better* answer at small-to-medium
> scale. Module 7 covers when to *not* do this.

#### Concept

The realization: most of what GitOps controllers do is watch a
repo, render manifests, apply them in order, detect drift, and
expose status. Each of those is a few dozen lines of code
*if your directory shape is standardized*.

A standardized fleet directory:

```
movies-prod/
├── base/
│   └── <service>/
└── clusters/
    └── ring-<n>/
        ├── kustomization.yaml
        └── version.yaml
```

Becomes operable by a 200-line CLI:

- `mfleet rings` — list rings and their pinned versions.
- `mfleet diff` — `kustomize build` each ring and diff against
  what's deployed.
- `mfleet promote <service> <version> --ring <n>` — generate the
  commit that bumps a ring's pin.
- `mfleet rollback <ring>` — generate the revert commit.
- `mfleet apply --ring <n>` — render and apply (the
  reconciliation step).
- `mfleet status` — render everything, compare to cluster.

The CLI doesn't have to be smart. It has to be **consistent with the
directory shape your team already uses.** If your directory shape
is right, the CLI is mechanical; if it's wrong, no controller saves
you.

**Where Claude fits:**

- **Generating the CLI.** Give Claude the directory shape and the
  list of operations you want. The first cut is usually
  ~80% correct.
- **Operating the CLI.** "Promote 0.9.1 to ring-0," "What versions
  are pinned across the fleet," "Generate the rollback for ring-1."
- **Reasoning about state.** "What's the difference between ring-0
  and ring-1 right now?" → Claude reads the repo, runs the CLI,
  answers.

**Where MCP fits:** wrap the CLI as an MCP server, and Claude
operates the fleet through tool calls. The team's workflow becomes
the tool's API instead of forcing the team to adopt someone else's
DSL. cllm's MCP layer (`repos/cllm/mcp/`) is a working Bart-owned
example of this pattern for a different domain (inference); the
same shape applies to fleet ops.

#### Example

A skeleton `mfleet` CLI (Go, since the canonical curriculum
language per [domain M of the inventory](skills-inventory.md#m-language--go-canonical)):

```go
// cmd/mfleet/main.go
package main

import (
    "context"
    "fmt"
    "os"
    "os/exec"
    "path/filepath"
)

func main() {
    if len(os.Args) < 2 {
        usage()
        os.Exit(2)
    }
    switch os.Args[1] {
    case "rings":
        rings()
    case "diff":
        diff()
    case "promote":
        promote(os.Args[2:])
    case "rollback":
        rollback(os.Args[2:])
    case "apply":
        apply(os.Args[2:])
    case "status":
        status()
    default:
        usage()
        os.Exit(2)
    }
}

func rings() {
    // Walk clusters/ring-*/version.yaml, print each ring's pin.
    matches, _ := filepath.Glob("clusters/ring-*/version.yaml")
    for _, m := range matches {
        // ... parse, print
        fmt.Println(m)
    }
}

func promote(args []string) {
    // mfleet promote <service> <version> --ring <n>
    // 1. read clusters/ring-<n>/version.yaml
    // 2. update images.newTag for <service>
    // 3. git add && git commit
    // 4. (optional) gh pr create
}

func apply(args []string) {
    // mfleet apply --ring <n>
    cmd := exec.CommandContext(context.Background(),
        "kubectl", "apply", "-k", "clusters/ring-"+args[1])
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr
    _ = cmd.Run()
}
```

200 lines plus tests is achievable. The discipline is "if a
command isn't trivial in this shape, fix the directory shape, not
the CLI."

#### The MCP wrap

Expose the CLI as an MCP server. The tools you'd surface:

- `mfleet_rings` → returns JSON of `{ring, service, version}` rows.
- `mfleet_promote` → takes `{service, version, ring}`, returns
  the commit SHA.
- `mfleet_rollback` → takes `{ring}`, returns the revert SHA.
- `mfleet_status` → takes `{ring}`, returns
  `{deployed_version, declared_version, drift_detected}`.

Claude in the room can now operate the fleet using the team's own
verbs. This is the inversion you don't get with Flux — you adopt
the team's vocabulary, not the controller's.

#### Lab

In a single session (90–120 min):

1. Pick a directory shape (the Module 3 ring layout is fine).
2. Ask Claude to generate `mfleet` per the skeleton above. Have it
   target the language from your inventory (Go, per domain M).
3. Run the generated CLI against a scratch fleet repo. Don't apply
   to a real cluster; `mfleet apply --dry-run` is enough.
4. Add the rollback drill from Module 3 Lab #4 using `mfleet`
   instead of raw `git revert`. Time it. Compare.
5. **Stretch goal:** wrap the CLI as an MCP server. cllm's
   `mcp/server.py` is a Python reference for the same idea; doing
   it in Go is its own exercise.

#### Knowledge check

1. Why does a standardized directory shape matter more than the
   choice of controller?
2. The CLI is "not smart." What does that mean concretely, and why
   is it the design goal?
3. What does wrapping the CLI as MCP buy you that a CLI alone
   doesn't?
4. When does the "roll your own" path stop scaling? (Hint: see
   Module 7.)
5. Claude can read your fleet repo and answer questions about it
   without any CLI at all. Why bother with the CLI?

---

### Module 6 — Secrets in GitOps (spec §13)

#### Concept

Spec §13 is explicit: **no secrets in repos.** GitOps amplifies the
problem: if you put a secret in a repo, it's in every clone,
every fork, every CI cache, forever. Even if you `git rm` and
`force-push`, the SHA is still indexable by anyone who pulled before
the rewrite.

Three patterns solve this:

1. **SOPS** (Mozilla, now CNCF). Encrypts the secret values in YAML
   files using a KMS-backed key (AWS KMS, GCP KMS, age,
   gpg). The encrypted file *is* committed; only people with the
   key can decrypt. Flux has native SOPS support.
2. **Sealed Secrets** (Bitnami). A `SealedSecret` CRD is committed
   to the repo. The cluster-side `sealed-secrets-controller`
   decrypts using its own private key and creates the actual
   `Secret`. The committed `SealedSecret` is opaque — only that
   specific cluster's controller can decrypt it.
3. **External Secrets Operator** (ESO). The repo contains a
   *reference* (an `ExternalSecret` CRD) pointing at a secret in
   AWS Secrets Manager / GCP Secret Manager / Vault / etc. ESO
   pulls the value at reconciliation time. The actual secret never
   touches the repo at all.

Curriculum recommendation: **ESO for production**, SOPS for the
in-between case where you want declarative + git-managed but don't
want a separate vault. Sealed Secrets is fine but cluster-bound,
which makes the rings story harder.

**Anti-patterns:**

- "Just put it in a ConfigMap, it's not technically a Secret."
  ConfigMaps are world-readable in-cluster.
- "We `.gitignore` the secrets file and pass it out-of-band." This
  breaks the GitOps contract (the cluster's state isn't
  reproducible from the repo).
- "Sealed Secrets are encrypted, so we can put the unencrypted ones
  in a `.bak` file." No.

#### Example

ExternalSecret manifest (the GitOps-friendly pattern):

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: movies-api-secrets
  namespace: movies
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: movies-api-secrets       # the in-cluster Secret ESO creates
  data:
    - secretKey: grafana-admin-password
      remoteRef:
        key: prod/movies/grafana-admin-password
```

The repo contains the *reference* (`key: prod/movies/...`); the
*value* lives in AWS Secrets Manager and never touches git.

#### Lab

1. In your scratch fleet repo, find every existing `Secret`. For
   each one, identify how it got there. If any are committed
   plaintext, you have an audit incident, not a lab.
2. Pick one secret. Convert it to an `ExternalSecret`
   (use a scratch external store — Vault dev mode, or a `kind:
   FakeStore` in ESO for the lab). Commit. Reconcile. Confirm the
   in-cluster `Secret` exists with the right value.
3. Rotate the secret in the external store. Wait for
   `refreshInterval`. Confirm the in-cluster `Secret` updates.
4. **The audit drill.** Have Claude grep your fleet repo for
   anything that looks like a base64-encoded secret. Read the
   findings. *Don't* just trust your gut that you've never
   committed one.

#### Knowledge check

1. Why is "we'll just `.gitignore` the secrets" a GitOps anti-pattern?
2. SOPS vs Sealed Secrets vs ESO — when does each one win? Give a
   real scenario for each.
3. The rings story (Module 3) wants the same `ExternalSecret`
   reference in every ring. Which of the three patterns makes that
   trivial? Which makes it painful?
4. A secret leaks into git despite all this. What's the recovery
   procedure? (Hint: rotation, not rewrite.)
5. Why does ESO refresh on an interval rather than on-change? What
   are the failure modes of each?

---

### Module 7 — When NOT to roll your own (the honest case for Flux/Argo)

> **Counterweight to Module 5.** Roll-your-own is right at small-to-
> medium scale. At a certain point, the cost flips. Knowing where
> the flip is matters as much as knowing how to build the CLI.

#### Concept

Flux and Argo solve real problems that get worse as you scale.
The honest list of when adopting one beats building one:

1. **Hundreds of clusters with no human in the loop.** Every
   minute, every cluster reconciles. Your CLI on a cron either runs
   centrally (and is a single point of failure) or runs on every
   cluster (and is now a distributed system you're maintaining).
   This is what Flux/Argo controllers are *for*.
2. **Multi-team, multi-tenant, with real audit obligations.** When
   the audit question is "show me every reconciliation event in
   prod for the last 90 days, with the SHA and the time," you want
   structured controller events feeding a real audit system, not
   `journalctl` from a cron.
3. **Drift detection at scale.** A CLI run once an hour can detect
   drift; a controller that watches the K8s API for changes
   detects drift in seconds. At small scale this doesn't matter.
   At 1000 clusters it does.
4. **Cross-team conventions.** When five teams in your org need to
   ship via the same shape, adopting Flux/Argo means everyone
   reads the same docs. Custom tooling means everyone reads *your*
   docs.
5. **Image automation policies at fleet scale.** Argo Image
   Updater + Flux's `ImagePolicy` express "semver, ≥0.9 and <1.0,
   prefer latest, only in ring-0" declaratively. Building the
   equivalent in your CLI is doable; debugging it at 3am is
   another matter.
6. **GitOps-native tooling ecosystem.** Argo Notifications, Argo
   Rollouts (canary + traffic shifting), Flux Notifications — you
   get these for free with the controllers. Building them yourself
   means building them yourself.

The honest list of when the controllers *don't* pay back:

1. **Fewer than 10 clusters.** The reconciliation engine's value
   is marginal; the operational complexity is real.
2. **One team, one product.** No cross-team convention to enforce.
3. **You're going to fight the controller's opinions.** If your
   directory shape doesn't fit Flux's worldview (or Argo's), and
   you don't want to change your shape, the controller becomes the
   tax.
4. **Your audit story is satisfied by the git log.** You don't
   need a separate event stream.

**The decision is not religious.** Use Flux when Flux pays back;
use your own CLI when it doesn't. The curriculum's bet is that the
flip-point has moved up significantly in the last two years —
many teams that "needed Flux" in 2022 don't need it in 2026.

#### Example

A decision matrix:

| Signal | Roll your own | Flux/Argo |
|---|---|---|
| Clusters | < 10 | 10+ |
| Teams shipping via this repo | 1 | 2+ |
| Audit obligations | "show me the git log" | "show me reconciliation events with SLAs" |
| Drift detection latency tolerance | minutes | seconds |
| Cross-team convention enforcement | not required | required |
| Image automation complexity | hand-written promotions | declarative policies |
| Multi-cluster bootstrap | one-time scripts | `Bootstrap` / `ApplicationSet` |

3+ signals on either side = strong indicator. Mixed signals = the
right answer is probably "Flux/Argo for the controller, plus a
CLI/MCP for the team's day-to-day verbs."

#### Lab

1. Score your real fleet against the decision matrix. Write down
   the score honestly.
2. If your score says "Flux/Argo" and you're rolling your own:
   list the specific pains you're absorbing. Is the absorbed pain
   greater or less than the adoption cost?
3. If your score says "roll your own" and you're running Flux:
   list the Flux features you're actually using. If most are
   unused, the controller is a tax you're paying for features you
   don't need.
4. Either way: read the matrix back to Claude and ask *"what am I
   missing?"* — the honest case-against your current choice is
   the lab.

#### Knowledge check

1. Name three signals that point toward adopting Flux/Argo.
2. Name three signals that point toward rolling your own.
3. The flip-point has "moved up" in the last two years. What
   specifically changed?
4. Argo Rollouts does canary + traffic shifting. Building the
   equivalent yourself is doable. Why is it usually not worth it?
5. The hybrid pattern is "Flux for reconciliation + custom CLI for
   day-to-day verbs." Why does that combination work? What does
   it cost?

---

## Per-release review

Same template as the observability guide; see the
[Per-release review template section](study-guide-observability.md#per-release-review-template).
GitOps-specific addition: when a release touches Module 3 (rings)
or Module 6 (secrets), the hands-on check **must** include a
deliberate `git revert` drill against the touched ring. "I know
how rings work in theory" is a much weaker signal than "I have
pulled the rollback lever this week."

## What this guide is and is not

- **Is:** the GitOps curriculum, post-Claude. Anchored in
  repo-per-env (from
  [study-guide-kustomize.md](study-guide-kustomize.md) Module 0),
  centered on rings, honest about when to adopt a controller
  vs roll your own.
- **Is not:** a Flux tutorial. Module 4 covers what you need to
  *recognize* and operate Flux; the official Flux docs are the
  reference.
- **Is not:** an Argo tutorial. Argo is named where it's relevant
  for trade-offs; the curriculum picks Flux as the canonical
  controller when one is needed because cllm uses it and so the
  repo has a working Bart-owned MIT reference.
- **Is not:** an opinion on cloud-vendor managed GitOps (AWS
  CodeCommit + Flux on EKS, GKE Config Sync, Azure Arc GitOps).
  Those are real options; the curriculum stays vendor-neutral
  because the spec is.

## Open questions

- **Module 5 vs Module 7 placement.** Module 5 ("roll your own") is
  before Module 7 ("when not to") deliberately — the bet is that
  most operators reading this guide are over-tooled, not under-
  tooled. If the audience profile is different, swap them.
- **The MCP slice (Module 5 stretch goal) might deserve its own
  module.** Building an MCP for fleet ops is a sizeable exercise
  and arguably its own domain. Current call: keep it as a stretch
  goal here; promote if the inventory grows an MCP-for-ops row.
- **Argo Rollouts is a real gap.** Canary + traffic shifting +
  analysis templates is a thing the curriculum doesn't cover.
  Should be its own module if/when the spec grows progressive-
  delivery requirements.
- **The decision matrix in Module 7 is opinionated and untested.**
  Real data from a few teams would calibrate it; current numbers
  are educated guesses.

## Status

- Not yet run end-to-end with any operator.
- Anchored in the cllm repo's working Flux setup; every controller-
  side example cites a file or pattern that exists today.
- Unlocks for promotion to `methodology/` once at least one full
  run has used it and reported honestly on whether the post-Claude
  framing holds up against a real fleet.
