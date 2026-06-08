# Skills Inventory — what an SE on a movies-spec-class project should know

> **DRAFT — NOT FOR PUBLICATION.** Parent document for the per-domain
> study guides (first instance:
> [study-guide-observability.md](study-guide-observability.md)).
> The inventory exists so the study guides have a defensible scope —
> we are *not* writing a Kubernetes textbook; we are filling in the
> skills the agent would otherwise silently substitute for the
> operator (see
> [sessions-and-skill-compounding.md](../methodology/drafts/sessions-and-skill-compounding.md)).

## Level convention

Borrowed from the Microsoft 100–400 training convention:

| Level | What it means | Test |
|---|---|---|
| **100** | Awareness. Knows the term, can read a doc with it in. | "What is X and why does the spec need it?" |
| **200** | Practitioner. Can use it day-to-day with reference material. | "Walk me through doing X on this cluster." |
| **300** | Expert. Can debug deeply, explain trade-offs, choose between alternatives. | "Y is broken — find it." / "Why this and not that?" |
| **400** | Principal. Can teach, can design at scale, can defend the choice in front of a skeptical room. | "Build the org's standard for X." / "Take it to 1K clusters." |

The spec's bar for a one-person, single-cluster run is **200 across
the board, 300 in observability and dev loop.** The enterprise bar
(GitOps, multi-cluster, multi-tenant) pushes several of these to 300
or 400. Each row below names both.

## Inventory

### A. Container fundamentals

| # | Skill | Spec bar | Enterprise bar | Notes / opinion |
|---|---|---|---|---|
| A1 | **What a container actually is** — process + namespaces + cgroups, *not* a VM; shares host kernel; PID 1 is your binary | 100 | 200 | Most "Docker is slow" complaints trace to not knowing this. See [study-guide-containers.md](study-guide-containers.md) Module 1. |
| A2 | Dockerfile authoring — `FROM`, `COPY`, `RUN`, `ENV`, `ARG`, `USER`, `ENTRYPOINT` (exec form) vs `CMD`, `.dockerignore`, syntax directive | 200 | 300 | Exec-form `ENTRYPOINT` matters for SIGTERM forwarding. See [study-guide-containers.md](study-guide-containers.md) Module 2. |
| A3 | **Multi-stage builds** — build stage (full toolchain) → runtime stage (minimal); `COPY --from`; multi-binary patterns; test stage | 200 | 300 | Spec §9 requires it. movies-bartr ships two binaries from one tree. See [study-guide-containers.md](study-guide-containers.md) Module 3. |
| A4 | **Layer caching + ordering** — deps above source; cascade rule; BuildKit `--mount=type=cache` and `--mount=type=secret` | 200 | 300 | The skill that turns 5-minute rebuilds into 5-second ones. See [study-guide-containers.md](study-guide-containers.md) Module 4. |
| A5 | **Distroless / Alpine minimal runtime, non-root, read-only-FS-compatible** — image-side + K8s-side completion; `automountServiceAccountToken: false` | 200 | 300 | Spec §9 + §13 require all three. Cross-reads with [study-guide-security.md](study-guide-security.md) Module 3. See [study-guide-containers.md](study-guide-containers.md) Module 5. |
| A6 | **Tagging + versioning discipline** — semver + git SHA tags, never `:latest` in deploy, digest pinning, OCI annotations, `VERSION` → ldflags → `/version` chain | 200 | 300 | Spec §12 inner loop depends on this. See [study-guide-containers.md](study-guide-containers.md) Module 6. |
| A7 | **`docker buildx`, multi-arch** — manifest lists; `--push` requirement; `BUILDPLATFORM` vs `TARGETPLATFORM`; cross-compile vs emulation | 100 | 300 | Enterprise needs arm64 + amd64 (Graviton, Apple Silicon). See [study-guide-containers.md](study-guide-containers.md) Module 7. |
| A8 | **Reading images** — `docker history`, `docker image inspect`, `dive`; wasted-space pattern; size-budget discipline | 200 | 300 | If the operator can't read the artifact, they ship what they don't understand. See [study-guide-containers.md](study-guide-containers.md) Module 8. |
| A9 | Image vulnerability scanning (trivy, grype, language auditor) | 100 | 300 | Spec §13 names "dependency scanning." Lives in [study-guide-security.md](study-guide-security.md) Module 5, not this guide. |

### B. Local platform — Docker, WSL, the "easy" mode that gets in the way

| # | Skill | Spec bar | Enterprise bar | Notes / opinion |
|---|---|---|---|---|
| B1 | **Linux is the floor** \u2014 every container, every cluster node is Linux; macOS / Windows are convenience layers running a Linux VM. **K8s == bash:** the entire ecosystem writes examples in POSIX shell; PowerShell is for Windows admin, not cluster work. | 200 | 200 | The reference; what everything else is emulating. **Non-negotiable for Windows operators:** learn Linux, get comfortable in zsh/bash, stop translating. See [study-guide-local-platform.md](study-guide-local-platform.md) Module 1. |
| B2 | **WSL2 on Windows + the [bartr/wsl](https://github.com/bartr/wsl) baseline + Windows Terminal** \u2014 `install.sh`, snapshot via `wsl --export`; **Build 2026: WSL Containers** (public preview, native Linux containers without Docker Desktop), enterprise-management API, open-source project (Build 2025) | 200 | 300 | **Opinion (strong):** if you are on Windows, live in WSL2 with a snapshot-able baseline. **Windows Terminal is not optional** \u2014 copy-paste alone is worth the install. Same script works on DO droplets. WSL Containers materially changes the Docker Desktop conversation on Windows. See [study-guide-local-platform.md](study-guide-local-platform.md) Module 2. |
| B3 | **macOS container runtime** — Docker Desktop / OrbStack / Colima trade-offs; optional Multipass VM for full-Linux parity | 200 | 300 | macOS zsh default since 2019. Docker Desktop licensing matters at scale. See [study-guide-local-platform.md](study-guide-local-platform.md) Module 3. |
| B4 | **Cloud dev environments** — GitHub Codespaces, DO/EC2 droplets via VS Code Remote-SSH, Microsoft Dev Box, VS Code dev tunnels | 100 | 300 | The sidearm. "My laptop died" stops being a workday-killer. The bartr/wsl baseline runs unmodified on a DO droplet. See [study-guide-local-platform.md](study-guide-local-platform.md) Module 4. |
| B5 | **Dev containers as IaC for the dev machine** — `.devcontainer/devcontainer.json`, `features`, postCreate, same file → local Dev Containers / Codespaces / Remote-SSH / `devcontainer up` CLI | 200 | 300 | **Promoted from "encouraged" (spec §9) to curriculum standard.** The piece that makes "spec the dev environment" tractable. See [study-guide-local-platform.md](study-guide-local-platform.md) Module 5. |
| B6 | **Shell baseline: zsh + oh-my-zsh** — tab completion, kubectl/docker/git plugins, curriculum aliases (`k`, `kgp`, `kgs`, ...) | 200 | 200 | macOS default since 2019; bartr/wsl ships it on Linux/WSL. Tab completion + uniformity for screen-share are the wins. See [study-guide-local-platform.md](study-guide-local-platform.md) Module 6. |
| B7 | **Networking pitfalls** — `localhost` ambiguity (host / VM / container), `host.docker.internal`, `--network=host` on Mac/Win gotcha, WSL2's four address spaces | 200 | 300 | The trap that bites everyone. Curriculum answer: `type: LoadBalancer` via klipper-lb for any service you'd `port-forward`. See [study-guide-local-platform.md](study-guide-local-platform.md) Module 7. |
| B8 | **Snapshot + reproducibility discipline** — `wsl --export`/`--import`, cloud snapshots, dev-container rebuild, named-volume reset, documented recovery path in every repo | 200 | 300 | The capstone. A 5-minute reset is what makes fearless experimentation cheap. See [study-guide-local-platform.md](study-guide-local-platform.md) Module 8. |
| B9 | Volume mount perf on macOS / Windows (bind mounts are slow; named volumes are fast) | 100 | 200 | Covered as a falsification lab in [study-guide-local-platform.md](study-guide-local-platform.md) Module 1; could expand if it becomes a frequent gap. |
| B10 | Licensing reality of Docker Desktop in commercial settings | 100 | 200 | Org-policy concern, not a skill. Drives Module 3's runtime-choice trade-offs. **Build 2026 update:** on Windows, WSL Containers is now the Microsoft-supported alternative; on macOS, OrbStack / Colima remain the answers. |
| B11 | Alternatives ecosystem awareness — Colima, Rancher Desktop, Podman Desktop, OrbStack, UTM, Multipass, Parallels, Microsoft Dev Box | 100 | 200 | At enterprise scale the team picks one; the SE should not be surprised by any of them. |

### C. Kubernetes core — opinionated on the local distro

| # | Skill | Spec bar | Enterprise bar | Notes / opinion |
|---|---|---|---|---|
| C1 | **The cluster mental model** — control plane vs nodes, **API server as the only writer**, reconciliation loops | 200 | 300 | The mental model that makes the rest make sense. See [study-guide-k8s-core.md](study-guide-k8s-core.md) Module 1. |
| C2 | **Local distro choice: k3s or k3d** (not minikube, not kind, not Docker Desktop's k8s) | 200 | 200 | **Opinion (strong):** k3s for VM/bare metal, k3d for laptop. Light, fast, production-shaped. See [study-guide-k8s-core.md](study-guide-k8s-core.md) Module 2. |
| C3 | Why not Docker Desktop Kubernetes / minikube for this work | 100 | 200 | Slower, less production-like, fewer of the things-that-bite-you in prod show up locally. Covered in [study-guide-k8s-core.md](study-guide-k8s-core.md) Module 2. |
| C4 | `kubectl` fundamentals — `get / describe / logs / exec / apply / delete / rollout / scale / wait` | 200 | 300 | The 80%. See [study-guide-k8s-core.md](study-guide-k8s-core.md) Module 3. |
| C5 | **`k` alias and shell shortcuts** — `alias k=kubectl`, `kgp`, `kgs`, `kgd`, completion for `k` | 200 | 200 | If you are typing `kubectl` in full, every hour of work has a tax on it. Lives in both [study-guide-local-platform.md](study-guide-local-platform.md) Module 6 and [study-guide-k8s-core.md](study-guide-k8s-core.md) Module 3. |
| C6 | **Context / namespace switching with `kubectx` / `kubens`** + visible-in-prompt current context | 200 | 300 | Enterprise = many clusters, many namespaces; not knowing this is dangerous. The wrong cluster + the wrong namespace is how production gets deleted. See [study-guide-k8s-core.md](study-guide-k8s-core.md) Module 3. |
| C7 | **`k9s`** — terminal UI for fast cluster inspection | 200 | 200 | Inner-loop multiplier. Worth an hour to learn. See [study-guide-k8s-core.md](study-guide-k8s-core.md) Module 3. |
| C8 | **Pods, ReplicaSets, Deployments** — the four-layer ownership chain; rolling updates; rollout history/undo; StatefulSets / DaemonSets / Jobs awareness | 200 | 300 | Most "the deployment didn't update" stories trace to misunderstanding the chain. See [study-guide-k8s-core.md](study-guide-k8s-core.md) Module 4. |
| C9 | **Services + cluster DNS** — ClusterIP / NodePort / LoadBalancer / ExternalName / headless; `<svc>.<ns>.svc.cluster.local` | 200 | 300 | See [study-guide-k8s-core.md](study-guide-k8s-core.md) Module 5. |
| C10 | **Probes — liveness, readiness, startup** — what each does, why readiness ≠ liveness, why readiness must be local | 200 | 300 | Spec §8.1 requires liveness + readiness. The most-misconfigured controls in K8s. See [study-guide-k8s-core.md](study-guide-k8s-core.md) Module 6. |
| C11 | **Resources — requests vs limits, QoS classes, OOMKilled vs Evicted, invisible CPU throttling** | 200 | 300 | Spec §8.1 fixes the numbers. See [study-guide-k8s-core.md](study-guide-k8s-core.md) Module 7. |
| C12 | **ConfigMaps and Secrets** — base64 is not encryption; RBAC is the real boundary; etcd encryption-at-rest is opt-in on vanilla K8s | 200 | 300 | See [study-guide-k8s-core.md](study-guide-k8s-core.md) Module 8; runtime injection patterns in [study-guide-security.md](study-guide-security.md) Module 4. |
| C13 | **RBAC basics — ServiceAccount, Role, ClusterRole, RoleBinding, ClusterRoleBinding** — per-workload SAs, default-deny posture, `cluster-admin` for workloads is almost always wrong | 100 | 300 | Operators like Prometheus Operator need it. Spec-floor pattern for workloads: `automountServiceAccountToken: false`. See [study-guide-k8s-core.md](study-guide-k8s-core.md) Module 9. |
| C14 | **Custom Resources (CRDs) + operator pattern** — CRD defines schema, controller does work; `kubectl explain` works on CRDs; ServiceMonitor as the canonical example | 200 | 300 | Spec §8.1 mandates ServiceMonitor. See [study-guide-k8s-core.md](study-guide-k8s-core.md) Module 10. |
| C15 | NetworkPolicy — default-deny, allow-by-label; CNI must implement it (k3s flannel does not by default) | 200 | 300 | Spec §13 requires it for movies-api. Pointer module in [study-guide-k8s-core.md](study-guide-k8s-core.md) Module 11; depth in [study-guide-security.md](study-guide-security.md) Module 2. |
| C16 | **Local-cluster ingress strategy** — `type: LoadBalancer` services via k3s's built-in `klipper-lb` + Traefik; `kubectl port-forward` is a smell | 200 | 300 | **Opinion (strong):** every service the operator interacts with locally (movies-api `:8080`, Grafana `:3000`, Prometheus `:9090`) gets a real LB port. `port-forward` hides Service / Ingress correctness. See [study-guide-k8s-core.md](study-guide-k8s-core.md) Module 5 and [study-guide-kustomize.md](study-guide-kustomize.md) Module 7. |
| C17 | **Reading a cluster cold** — the 15-min walkthrough that produces a one-page status report from an unfamiliar cluster | 200 | 300 | The capstone for domain C. Scales from a 3-node lab to an 800-cluster fleet — same template, same skill. See [study-guide-k8s-core.md](study-guide-k8s-core.md) Module 12. |

### D. Manifest packaging — Kustomize, scalable

| # | Skill | Spec bar | Enterprise bar | Notes / opinion |
|---|---|---|---|---|
| D1 | Kustomize mental model — `base/` + `overlays/`, no templating | 200 | 300 | Spec §8 mandates Kustomize, forbids Helm. |
| D2 | Common transforms — `namePrefix`, `commonLabels`, `images`, `replicas`, `patches` (JSON6902 + strategic merge) | 200 | 300 | |
| D3 | **Overlay design inside one environment** — thin `base/`, per-environment overlay with the right mix of `patches` / `images` / `replicas` / per-env resources | 200 | 300 | This is the per-repo Kustomize lever. Cross-environment / cross-cluster scaling is D8 (repo topology) and the GitOps guide (rings), not sibling overlays in one tree. Matt-v1's implementation is one overlay over base — fine for spec, doesn't survive a second environment. |
| D4 | Kustomize components — reusable, mix-in pieces; the lever for the many-cluster path | 100 | 400 | Without this you end up with N overlays that are 90% copy-paste. Most useful inside one fleet-config repo or inside a per-environment repo with many tenants; combines with D8. |
| D5 | `kustomize build` vs `kubectl apply -k` vs `kubectl kustomize` | 200 | 300 | |
| D6 | Why **not** Helm (in this repo / spec) — strong opinion, defensible | 100 | 300 | Templating + values files + lifecycle hooks add a layer of "magic" that loses in a GitOps reconciliation model. The SE should be able to defend the call, not just repeat it. |
| D7 | Image tag substitution patterns for the inner loop — `kustomize edit set image` + semver tags; never `:latest` + `imagePullPolicy: Always` | 200 | 300 | Spec-bar tag management. Per-ring image automation and GitOps controllers belong in the GitOps guide, not here. |
| D8 | **Repo topology** — single-repo + branches / single-repo + directories / **repo per environment** (curriculum default) / app-config + fleet-config split | 200 | 300 | **Opinion (strong):** GitHub permissions are per-repo, not per-branch or per-directory. CODEOWNERS + branch protection is a review gate, not a security boundary. Repo-per-env is the simplest shape that secures cleanly and scales predictably; KISS — keep it simple so it scales. App+fleet split is advanced and usually not worth the effort until N services × M environments produces real copy-paste pain. See [study-guide-kustomize.md](study-guide-kustomize.md) Module 0. |

### E. GitOps — the path from "one cluster" to "1K clusters"

| # | Skill | Spec bar | Enterprise bar | Notes / opinion |
|---|---|---|---|---|
| E1 | The GitOps **contract** — declarative, versioned, pulled, continuously reconciled | 100 | 300 | Spec does not require GitOps. Enterprise does. The contract is tool-independent; if you can articulate it, you can evaluate any tool against it. See [study-guide-gitops.md](study-guide-gitops.md) Module 2. |
| E2 | **Ring-based promotion + `git revert` as rollback** | 100 | 400 | **The canonical GitOps example.** Per-ring version pinning in the fleet repo; promotion forward (Ring 0 → 1 → 2 → 3); `git revert` as the rollback button. See [study-guide-gitops.md](study-guide-gitops.md) Module 3. |
| E3 | **Flux primitives** — `GitRepository`, `Kustomization`, `HelmRelease`, `ImageUpdateAutomation`, dependency ordering | 100 | 400 | Bart's primary controller choice when one is needed; cllm uses it (`clusters/z01/`, `scripts/init-flux.sh`). |
| E4 | Roll-your-own GitOps — standardized directory shape + small CLI (Claude-generated, ~200 lines) | 100 | 300 | **Post-2024 alternative to adopting Flux/Argo.** Works at small-to-medium scale; the directory shape matters more than the controller. See [study-guide-gitops.md](study-guide-gitops.md) Module 5. |
| E5 | MCP for fleet ops — wrap the team's CLI as MCP so Claude operates via the team's verbs | 100 | 300 | The team-workflow-matches-the-tool inversion you don't get with Flux/Argo. cllm's `mcp/` is a working reference for a different domain. |
| E6 | ArgoCD — alternative controller; UI-first, sync waves, app-of-apps, Argo Rollouts | 100 | 300 | Should at least know it exists and the trade-off vs Flux. Argo Rollouts is a real gap when progressive delivery is in scope. |
| E7 | When to adopt a controller vs roll your own — the decision matrix | 100 | 400 | Hundreds of clusters, multi-team audit obligations, fleet-scale image automation → adopt. < 10 clusters, single team, git-log-is-enough audit → roll your own. See [study-guide-gitops.md](study-guide-gitops.md) Module 7. |
| E8 | Secrets in GitOps — SOPS, Sealed Secrets, External Secrets Operator | 100 | 300 | Spec §13 forbids secrets-in-repo. GitOps without a story here is a vulnerability. **Curriculum default: ESO for production**, SOPS for the in-between case. |
| E9 | Multi-cluster bootstrap — how a brand-new cluster becomes a Flux-managed cluster from one command | 100 | 400 | The actual "1K clusters" lever. cllm's `scripts/init-flux.sh` is the working reference. |
| E10 | Cluster fleet patterns — "cluster per tenant" vs "namespace per tenant" vs "control-plane-of-clusters" | 100 | 400 | Where the 1K-cluster opinion lives. |

### F. Ingress and edge

| # | Skill | Spec bar | Enterprise bar | Notes / opinion |
|---|---|---|---|---|
| F1 | **Layers of routing** — Service (L4) / Ingress (L7) / LoadBalancer (L4 edge) / Gateway API (L7, post-Ingress); IngressClass as the binding | 200 | 300 | The trap is conflating L4 and L7. See [study-guide-ingress.md](study-guide-ingress.md) Module 1. |
| F2 | **Traefik as the k3s default** — vanilla `Ingress` vs Traefik CRDs (`IngressRoute`, `Middleware`), the dashboard, "don't fight the distro" | 200 | 300 | **Opinion:** if you are on k3s/k3d, learn Traefik first. Spec-floor uses vanilla `Ingress` for portability; `IngressRoute` + `Middleware` for cluster-level concerns. See [study-guide-ingress.md](study-guide-ingress.md) Module 2. |
| F3 | **Entrypoints and port discipline** — every Ingress declares exactly one entrypoint; agent defaults will silently collide routes; k3s `HelmChartConfig` is the right mechanism for extra entrypoints | 200 | 300 | **Strongest opinion in domain F.** Anchored in movies-bartr's three-Ingress / five-entrypoint setup. The per-release audit catches the next silent collision before it ships. See [study-guide-ingress.md](study-guide-ingress.md) Module 3. |
| F4 | **Middleware patterns** — rate-limit, headers (HSTS/CORS/security), redirect-scheme, strip-prefix, basic-auth / forward-auth, ip-allow-list, compress, circuit-breaker, retry | 200 | 300 | One `Middleware` CRD per concern, referenced by many routes — not copy-paste into annotations. Order in the middleware chain matters. See [study-guide-ingress.md](study-guide-ingress.md) Module 4. |
| F5 | **TLS termination + cert-manager + Let's Encrypt** — self-signed for dev, ACME for public, HTTP-01 vs DNS-01, staging-then-prod issuer flip, end-to-end TLS as the exception not the default | 100 | 300 | Enterprise table stakes. **Always test on the staging ClusterIssuer first** — prod rate-limits at 50 certs/domain/week. See [study-guide-ingress.md](study-guide-ingress.md) Module 5. |
| F6 | **Ingress vs Gateway API — the migration question** — role separation, multi-protocol, expressive routing, cross-namespace; when each is right; the "learn Ingress first, deeply" curriculum position | 100 | 300 | Migrate when you hit a real Ingress limitation on a real workload, not pre-emptively. Most teams will not hit it; the ones that do will know. See [study-guide-ingress.md](study-guide-ingress.md) Module 6. |
| F7 | **Reading an ingress baseline cold** — 15-min walkthrough, 6 questions, one-page markdown report | 200 | 300 | Capstone for domain F; mirrors the K8s-core M12 cold-cluster-read pattern at L7. See [study-guide-ingress.md](study-guide-ingress.md) Module 7. |

### G. Observability — *covered separately*

Full breakdown in
[study-guide-observability.md](study-guide-observability.md).
Targets summarized:

| Module | Spec bar | Enterprise bar |
|---|---|---|
| Metrics fundamentals | 300 | 300 |
| Structured logging | 200 | 300 |
| Grafana dashboards + UI editing | 200 | 300 |
| PromQL essentials | 200 | 300 |
| Prometheus Operator + ServiceMonitor | 200 | 300 |
| Probes + `/version` + inner loop | 300 | 300 |

### H. Dev loop and tooling

| # | Skill | Spec bar | Enterprise bar | Notes / opinion |
|---|---|---|---|---|
| H1 | The §12 inner loop end-to-end — bump → build → deploy → verify → validate → inspect, with a "loop closed" signal at every step | 300 | 300 | Spec's headline workflow. Integration module. See [study-guide-dev-loop.md](study-guide-dev-loop.md) Module 1. |
| H2 | **`make` discipline** — composable / inspectable / idempotent; wrappers earn their keep, hiders don't | 200 | 200 | Spec §9 says Makefile is *optional*; that's deliberate. A bad Makefile is worse than no Makefile. See [study-guide-dev-loop.md](study-guide-dev-loop.md) Module 2. |
| H3 | Versioning discipline — semver, what bumps what, where the version lives in the repo | 200 | 300 | Covered in [study-guide-containers.md](study-guide-containers.md) Module 6 (VERSION → ldflags → `/version` chain). Not re-explained in H guide. |
| H4 | **Shell fluency** — `jq`, `yq`, `xargs`, `awk` — the four tools that cover ~95% of curriculum shell work | 200 | 300 | The cost of not knowing `jq` is paid every session. See [study-guide-dev-loop.md](study-guide-dev-loop.md) Module 3. |
| H5 | **`git` for the methodology** — rebase vs merge, FF-merge close ritual, tagging, `--no-pager`, `bisect`, `reflog`, never `--squash` | 200 | 300 | Methodology assumes FF-merge close ritual. See [study-guide-dev-loop.md](study-guide-dev-loop.md) Module 4. |
| H6 | **Commit granularity** — one reason / independently revertable / independently understandable; 8–20 commits per session | 200 | 300 | Named gap from Matt-v1 review. See [study-guide-dev-loop.md](study-guide-dev-loop.md) Module 5 and [sessions-and-skill-compounding.md](../methodology/drafts/sessions-and-skill-compounding.md). |
| H7 | Devcontainers — when worth the cost, when not | 100 | 200 | Covered in [study-guide-local-platform.md](study-guide-local-platform.md) Module 5. Not re-explained in H guide. |

### I. Testing and benchmarks

| # | Skill | Spec bar | Enterprise bar | Notes / opinion |
|---|---|---|---|---|
| I1 | Unit vs integration vs E2E vs contract vs load — what belongs where | 200 | 300 | Spec §10 splits them this way. See [study-guide-testing.md](study-guide-testing.md) Module 1. |
| I2 | Coverage as a guide-rail, not a goal (≥ 80% on data + HTTP layers per spec) | 200 | 300 | Coverage tells you what you haven't *thought* about testing, not whether tests are good. See [study-guide-testing.md](study-guide-testing.md) Module 2. |
| I3 | **Contract / validation suites in-cluster** — black-box assertions on the live service over the real LB (`statusCode`, `contentType`, `length`, JSON shape) | 200 | 300 | Spec §10.3 requires this. The validation features earn their keep on the "valid JSON, wrong fields" failure mode unit tests miss. See [study-guide-testing.md](study-guide-testing.md) Module 3. |
| I4 | **The webv evolution: CLI → validation → in-cluster baseline** — the Helium pattern, every validation feature traces to an outage | 300 | 400 | **Curriculum centerpiece for I.** Once the suite runs 24/7 in-cluster against the live service, the dashboard signature *is* the test result. See [study-guide-testing.md](study-guide-testing.md) Module 4. Web Validate is Microsoft / MIT (https://github.com/microsoft/webvalidate). |
| I5 | **Dual instrumentation — client-side Prometheus histograms + dual-format logs (TSV for humans, JSON for log platforms)** — same metric shape on both sides + same correlation-id in both logs; `client_duration − server_duration` is the latency *outside* the service; volatility explodes with topological distance | 300 | 400 | **The original reason webv existed at Helium.** Two consumers, two ergonomics: TSV is right at the laptop, JSON is right for Loki / ELK — same fields either way, switchable by `--json`. movies-bartr today has TSV but no `--json` flag and no `/metrics` endpoint; Module 5's lab is the implementation walkthrough. See [study-guide-testing.md](study-guide-testing.md) Module 5. |
| I6 | **Load generation tools — when each is right** — `webv` (sustained baseline), `vegeta` (rate-controlled sweeps), `k6` (scripted scenarios), `hey` (smoke shots) | 200 | 300 | Don't try to make one tool do all four jobs. See [study-guide-testing.md](study-guide-testing.md) Module 6. |
| I7 | Reading benchmarks — p50/p95/p99 vs mean, RPS, error rate, why mean lies | 200 | 300 | The single most important benchmark literacy: **don't quote the mean for latency.** See [study-guide-testing.md](study-guide-testing.md) Module 7. |
| I8 | Performance methodology — establish baseline → change exactly one thing → measure → record → decide | 100 | 300 | Anchored in movies-bartr's `docs/PERFORMANCE.md` math for `--threads=2 --sleep=3ms`. See [study-guide-testing.md](study-guide-testing.md) Module 8. |
| I9 | **Continuous testing in production** — the dashboard signature *is* the SLO; bake-time gates promotion; negative-path suite tests the alerting | 300 | 400 | **Curriculum capstone for I.** The 48-hour smoke test wasn't the lesson; the structural baseline-runs-forever was. Composes with the rings + git revert pattern from [study-guide-gitops.md](study-guide-gitops.md) Module 3. See [study-guide-testing.md](study-guide-testing.md) Module 9. |

### J. Security

| # | Skill | Spec bar | Enterprise bar | Notes / opinion |
|---|---|---|---|---|
| J1 | **Pod / container `securityContext`** — seven controls (runAsNonRoot, runAsUser, fsGroup, allowPrivilegeEscalation, readOnlyRootFilesystem, capabilities.drop ALL, seccompProfile) + `automountServiceAccountToken: false` | 200 | 300 | Spec §8.1 + §13. movies-bartr `deploy/movies/base/deployment.yaml` is the reference. See [study-guide-security.md](study-guide-security.md) Module 1. |
| J2 | **NetworkPolicy: default-deny + narrow-allow pattern** — namespace selectors that survive cluster moves, prove deny works | 200 | 300 | Spec §13. The control most teams set once and never test. movies-bartr `networkpolicy.yaml` is the reference. See [study-guide-security.md](study-guide-security.md) Module 2. |
| J3 | **Image security** — distroless base, multi-stage, non-root `USER`, scan (trivy/grype), SBOM (syft), signing (cosign) | 200 | 300 | Spec §9 + §13. Tied to [study-guide-kustomize.md](study-guide-kustomize.md) Module 6 and [study-guide-go.md](study-guide-go.md) Module 10. See [study-guide-security.md](study-guide-security.md) Module 3. |
| J4 | **Secrets at runtime** — `envFrom` vs `valueFrom` vs projected volume; leak-risk ordering; redaction in startup logs | 200 | 300 | Spec §13. Git-side covered in [study-guide-gitops.md](study-guide-gitops.md) Module 6. See [study-guide-security.md](study-guide-security.md) Module 4. |
| J5 | **Supply-chain hygiene** — three layers (deps audit / image scan / SBOM+signing), cadence + waiver workflow | 100 | 300 | Spec §13. Cadence matters as much as the tool; quarterly scans are theater. See [study-guide-security.md](study-guide-security.md) Module 5. |
| J6 | **OWASP top-10 for GET-only HTTP APIs** — A05 misconfig, A06 deps, A09 logging, A10 SSRF; `problem+json` discipline; rate limiting | 100 | 200 | Lightest module — spec is narrow. Grows to its own guide if write endpoints / auth land in spec. See [study-guide-security.md](study-guide-security.md) Module 6. |
| J7 | **Read a security baseline cold** — score Deployment + NetworkPolicy + Dockerfile in 15 minutes against modules 1–6; ship / fix-then-ship / do-not-ship verdict | 200 | 300 | The capstone. Belongs in the per-release review at curriculum level. See [study-guide-security.md](study-guide-security.md) Module 7. |
| J8 | Admission control — PodSecurity admission, Kyverno, OPA — catching misconfig at apply time, not review time | 100 | 300 | Out of scope for spec but called out as a gap in the guide's open questions. |
| J9 | Runtime security + service mesh mTLS — Falco, Linkerd / Istio mTLS | 100 | 300 | Enterprise-scale; out of scope for spec. |
| J6 | Supply-chain awareness — SBOM, signed images (cosign / sigstore) | 100 | 300 | Enterprise table stakes. |
| J7 | RBAC least-privilege for service accounts | 100 | 300 | |

### K. Linux and HTTP fundamentals (the floor)

> **Reduced-form guide.** Not a teach-from-scratch text — a
> self-assessment checklist + labs + canonical-reading
> pointers. See [study-guide-floor.md](study-guide-floor.md).

| # | Skill | Spec bar | Enterprise bar | Notes / opinion |
|---|---|---|---|---|
| K1 | Process model, signals (SIGTERM, graceful shutdown, `terminationGracePeriodSeconds`), PID 1 + exec-form ENTRYPOINT | 200 | 300 | The cause of 90% of "why did my pod die uncleanly" tickets. See [study-guide-floor.md](study-guide-floor.md) Module 1. |
| K2 | File descriptors, ulimit, `/proc/<pid>/limits`, `somaxconn`, the things that bite high-RPS services | 100 | 300 | The shell's ulimit ≠ what the process sees. See [study-guide-floor.md](study-guide-floor.md) Module 2. |
| K3 | DNS resolution inside a pod — `/etc/resolv.conf`, the `ndots: 5` trap, FQDN with trailing dot, CoreDNS, headless vs ExternalName Services | 100 | 300 | "DNS is slow in our cluster" almost always ends at `ndots: 5`. See [study-guide-floor.md](study-guide-floor.md) Module 3. |
| K4 | HTTP status codes — 4xx-vs-5xx as "whose fault"; idempotency of GET/PUT/DELETE vs POST/PATCH; RFC 7807 `application/problem+json`; 502 vs 503 from Traefik | 200 | 300 | Spec §6 fixes these; movies-bartr's `test.yaml` validates the 4xx envelope. See [study-guide-floor.md](study-guide-floor.md) Module 4. |
| K5 | HTTP keep-alive, connection reuse, HTTP/1.1 vs HTTP/2 vs HTTP/3, why benchmarks lie without keep-alive | 100 | 300 | The #1 "benchmark looks fast in dev, bad in prod" cause. See [study-guide-floor.md](study-guide-floor.md) Module 5. |
| K6 | TLS basics — TLS 1.3 handshake, SNI, chain of trust + intermediates, SAN vs deprecated CN, mTLS for service mesh / B2B, cert expiry as the #1 outage cause | 100 | 200 | Floor under ingress guide M5. Be comfortable reading a cert as a structured document. See [study-guide-floor.md](study-guide-floor.md) Module 6. |

### L. Methodology (this repo)

> **Reading-list form.** Annotated index into the six
> existing methodology docs in `methodology/` — not a
> re-teach. See [study-guide-methodology.md](study-guide-methodology.md).

| # | Skill | Spec bar | Enterprise bar | Notes / opinion |
|---|---|---|---|---|
| L1 | Sessions + RPI cycle | 200 | 300 | The unit of work. Canonical text: [`sessions-not-stories.md`](../methodology/sessions-not-stories.md) + [`sessions-and-rpi.md`](../methodology/sessions-and-rpi.md). Index: [study-guide-methodology.md](study-guide-methodology.md). |
| L2 | Frame → Plan → Fit-check → Implement → Review → Close | 200 | 300 | Canonical text: [`sessions-and-rpi.md`](../methodology/sessions-and-rpi.md) "How One Session Maps to One RPI Cycle". |
| L3 | RETRO honesty — name what didn't work, not just what shipped | 200 | 300 | Worked example: [`sessions-and-skill-compounding.md`](../methodology/drafts/sessions-and-skill-compounding.md) is RETRO honesty applied to the methodology itself. |
| L4 | **Asking the agent for coaching** — "How could I have done this better?" / "What did I miss?" | 200 | 300 | Named gap from Matt-v1. Free, available, underused. See [`sessions-and-skill-compounding.md`](../methodology/drafts/sessions-and-skill-compounding.md). |
| L5 | Per-release human + Claude review (this study-guide series) | 200 | 300 | The proposed remediation in [`sessions-and-skill-compounding.md`](../methodology/drafts/sessions-and-skill-compounding.md); operationalized across the 12 study guides via the per-release review template in [study-guide-observability.md](study-guide-observability.md). |

### M. Language — Go (canonical)

> **Opinion (strong):** the curriculum standardizes on **Go** for all
> canonical examples. Rationale below. Other languages get a one-paragraph
> "translation note" per module rather than a parallel curriculum.

**Why Go for the canonical examples (not Rust, Node, .NET, Python):**

1. **The image story matches the spec defaults.** Spec §9 requires
   distroless/Alpine + non-root + read-only FS. A static Go binary
   on `gcr.io/distroless/static` lands at ~15MB with no ceremony.
   Node ~150MB, .NET ~110MB, Rust ~20MB but with a longer build.
2. **Runtime model matches what Kubernetes assumes.** Goroutines +
   `context.Context` cancellation map directly onto SIGTERM /
   `terminationGracePeriodSeconds` / request-scoped timeouts. K1
   (graceful shutdown) is *easy* in Go and a footgun in most other
   stacks — that's pedagogy gold, not language preference.
3. **Idiomatic Go reads like pseudocode.** Critical when the agent
   writes most of the code and the operator must *read* it. A 20-line
   `listMovies` handler is followable cold; async-Rust middleware
   stacks and DI-heavy Spring controllers are not.
4. **Prometheus client library is first-class.** `promhttp.Handler()`
   plus `prometheus.NewHistogramVec` is two lines. Other ecosystems
   have working libraries; none are as obviously The Way.
5. **Build, test, format, lint are in the toolchain.** `go build`,
   `go test`, `gofmt`, `go vet`, `staticcheck`. No bundler, no
   package-manager war, no virtualenv conversation. Removes a class
   of accidental-complexity questions from every session.

**Why not Rust** (the obvious challenger): async Rust specifically —
`Pin`, `Send + Sync` bounds, `tokio` vs `async-std`, borrow-checker
fights in middleware — is accidental complexity that costs operator
attention you want spent on PromQL and ServiceMonitors. Rust wins
when binary size, predictable latency, and zero-GC pauses are the
constraint. Movies-spec p95 is 0.2ms in Go with no caching layer;
the constraint isn't biting.

**The translation-stub pattern.** Each Go module in a study guide
gets a short "translation notes" section naming the equivalent
primitive in axum (Rust), fastify (Node), minimal API (.NET),
FastAPI (Python). Operator who needs another language runs the
prompt: *"translate this module's example to <stack>, keep the spec
contract identical."* Agent does the mechanical work; the concept
transferred without rewriting the curriculum.

**Confound to name.** Matt-v1 was Go. Standardizing the curriculum
on Go does *not* let us strip "is Go just easy" from "did the
methodology generalize." Hedge: invite at least one non-Go run
before promoting any of this to `methodology/`. The Movies harness
stays language-agnostic by spec; only the curriculum is opinionated.

| # | Skill | Spec bar | Enterprise bar | Notes / opinion |
|---|---|---|---|---|
| M1 | Project layout — `cmd/`, `internal/`, `pkg/`, why `internal/` matters | 200 | 200 | Matches movies-bartr and cllm layout. |
| M2 | HTTP server idioms — `net/http`, `http.ServeMux` (1.22+) or `chi`, handler signatures, middleware as `func(http.Handler) http.Handler` | 200 | 300 | Stdlib mux is enough for movies-spec; chi for richer routing. Matt-v1 used stdlib mux; movies-bartr used chi. |
| M3 | **`context.Context` and cancellation** — request-scoped contexts, deadlines, propagating cancel into downstream work | 200 | 300 | The lever for K1 (graceful shutdown) and J/security timeouts. |
| M4 | **Graceful shutdown** — `signal.NotifyContext`, `http.Server.Shutdown`, drain pattern | 200 | 300 | Twenty lines that get K1 right. Matt-v1 had a partial version. |
| M5 | Error handling discipline — `errors.Is` / `errors.As`, wrapped errors, no panic-as-control-flow | 200 | 300 | |
| M6 | **Structured logging with `log/slog`** (stdlib, 1.21+) — JSON handler, levels, request-scoped attrs | 200 | 300 | Maps directly onto spec §7.2. |
| M7 | **Prometheus client lib** — `promhttp.Handler()`, `CounterVec` / `HistogramVec` / `GaugeVec`, per-router registry | 200 | 300 | Maps directly onto spec §7.1. movies-bartr uses per-router registry — note the trade-off. |
| M8 | Table-driven tests + `httptest.NewServer` for HTTP integration | 200 | 300 | Spec §10.1/§10.2. |
| M9 | Coverage — `go test -cover`, `-coverprofile`, `go tool cover -html` | 200 | 200 | Spec §10.1 requires ≥80%. |
| M10 | Benchmarks — `testing.B`, `b.ReportAllocs()`, when micro-benchmarks lie | 100 | 300 | |
| M11 | Build flags for tiny static binaries — `CGO_ENABLED=0`, `-ldflags="-s -w"`, `-trimpath` | 200 | 200 | The 15MB story depends on these. |
| M12 | Multi-stage Dockerfile for Go — builder stage with full toolchain, runtime on `distroless/static` or `scratch` | 200 | 300 | Spec §9 + §13. |
| M13 | `go.mod` / `go.sum` hygiene — minimum version selection, `go mod tidy`, vendoring vs not | 200 | 200 | |
| M14 | `gofmt` / `go vet` / `staticcheck` / `golangci-lint` as the default loop | 200 | 200 | Non-negotiable, free, fast. |
| M15 | Concurrency primitives — goroutines, channels, `sync.Mutex`, `errgroup` — and when *not* to reach for them | 100 | 300 | Movies-spec rarely needs them; production services often do. |
| M16 | Race detector — `go test -race`, what it catches and what it costs | 100 | 200 | |
| M17 | Awareness — "the equivalent of M2–M7 in your target language" (axum / fastify / minimal API / FastAPI) | 100 | 200 | The translation-stub row. Keeps the SE honest about which concepts are language-independent. |

## Things I think you may also be forgetting

Not claiming exhaustive; flagging the gaps I spotted that you didn't
list in the prompt.

1. **Signals and graceful shutdown.** Listed as K1. Genuinely common
   blind spot — pods get `SIGTERM`, get `terminationGracePeriodSeconds`
   to drain, then `SIGKILL`. Services that don't trap and drain leak
   in-flight requests under any rolling deploy.
2. **DNS inside a pod.** K3. The cluster-DNS rabbit hole is where
   "why does my service work in `curl` but not from the other pod" lives.
3. **NetworkPolicy as a real control.** J2. Most teams set it once
   and never test it. Easy to write one that *looks* restrictive and
   isn't.
4. **Secrets in GitOps.** E5. The spec forbids secrets-in-repo; GitOps
   without SOPS / Sealed Secrets / External Secrets means *someone*
   is `kubectl apply -f`-ing secrets out-of-band, and that's the
   real-world break of the GitOps contract.
5. **Image tag automation under GitOps.** D7 / E2. "How does a new
   image actually reach prod" is a different problem under GitOps
   than under `kubectl set image`. Flux's image-automation controllers
   are the answer; the SE should know they exist.
6. **Multi-arch images.** A7. Apple Silicon laptops + amd64 clusters
   is the most common surprise; the SE should know `buildx` is the
   fix and what `--platform` does.
7. **CRDs as first-class concepts.** C14. `ServiceMonitor` is one;
   Flux's `Kustomization` is one; Traefik's `IngressRoute` is one.
   "What's a CRD" is a 100-level question whose absence makes a lot
   of operator-driven systems feel like magic.
8. **Reading a flamegraph / profile.** Not on the list above because
   it's outside what movies-spec needs at the 0.9.0 perf bar. Worth
   noting that for *any* perf-sensitive service, this is a gap that
   compounds.
9. **The licensing / org-policy floor** for Docker Desktop, base
   images, AGPL deps, etc. (B6.) Not technical, often missed by ICs,
   blocks real adoption in enterprises.
10. **What "production-shaped local" actually means.** Why k3s/k3d
    beats minikube isn't just "lighter"; it's that the things that
    break in prod (CNI quirks, real LoadBalancer behavior via
    klipper-lb, Traefik IngressRoute CRDs, real ServiceMonitors) all
    *show up locally.* The methodology depends on local = prod-shaped.

## What is explicitly out of scope (for this inventory)

- **Languages other than Go, at curriculum depth.** Go is the canonical
  language for examples (see domain M). Other stacks are covered via
  the translation-stub pattern: short, per-module notes naming the
  equivalent primitive, leaning on the agent to do the mechanical
  rewrite. The Movies *harness* stays language-agnostic by spec; only
  the *curriculum* is opinionated.
- Cloud-vendor specifics (EKS / AKS / GKE / managed Prometheus).
  Enterprise relevant, but a separate inventory.
- Data engineering, ML serving, queueing systems. Different specs.
- Frontend / UI. movies-spec is API-only.

## How this inventory drives the study-guide series

- Each domain (A–M) that has a meaningful gap-at-spec-bar gets its
  own study guide in this format:
  [study-guide-observability.md](study-guide-observability.md).
- Domains where the spec bar is already 200 *and* the agent doesn't
  silently substitute (e.g. K. Linux/HTTP) get a one-page check-list
  rather than a full guide.
- Per-release review (the test) only covers modules the release
  touched. Inventory rows not touched in a release are not examined.
- New domains the inventory missed get added as they surface in
  RETROs and the inventory itself versions.
- **Domain M (Go) is cross-cutting** — every other domain's study
  guide uses Go for its canonical example and appends a short
  "translation notes" stub. M's own guide covers the language
  concepts that don't fit naturally inside another domain (project
  layout, error handling, table-driven tests, `gofmt`/`vet`/lint loop).

## Open questions for you

*Settled in this revision:* 200-floor is the right framing for a
single-app curriculum (your call). Language is **in** scope; Go is
the canonical choice; other languages get translation stubs (your
call). Remaining:

1. **The enterprise bar.** I called Flux + multi-cluster bootstrap
   400. Is that right for the SE you have in mind, or is the SE
   target the 300-line and 400 belongs to a Staff/Principal?
2. **Domains I missed.** The "I think you may also be forgetting"
   list is my pass. What's *your* pass?
3. **Order.** Which domain after observability? My instinct is **D.
   Kustomize / manifest packaging** next, because that's where
   Matt-v1's gap was most visible to you and the 1K-cluster story
   lives there. Then E (GitOps), then J (Security). M (Go) gets
   written in parallel because every other guide depends on its
   examples.
4. **Format for the lighter domains.** Full study guide for every
   domain is a lot of words. Is a single-page "checklist + labs
   reference" acceptable for the floor domains (K), or does
   everything get the six-module treatment?
5. **The Go-confound hedge.** When do we commit to running at least
   one non-Go participant before promoting the inventory + study
   guides to `methodology/`? Naming it now keeps us honest later.

## Status

- Inventory only. Not yet used to drive any session.
- Promotes once we've run at least one study guide through a release
  review and validated the inventory's level targets against actual
  operator performance.
