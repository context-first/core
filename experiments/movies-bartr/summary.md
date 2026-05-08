# Experiment 003: movies-bartr

> A direct, intentional replication of the Helium MVP. Same scope. Single operator. Methodology-driven.
>
> **This is also the first run through a reusable experiment harness** — future participants will run the same spec independently, and their runs will be comparable evidence by design.

## Two Repos, One Experiment

This experiment is structured as **harness + participant runs**, not a single project:

| Repo | Role | Public URL |
|---|---|---|
| **harness / template** | The spec, methodology guide, experiment framing, session-log template, and 100-movie dataset. Other FDEs `Use this template` to start their own run. | [github.com/bartr/movies](https://github.com/bartr/movies) |
| **movies-bartr (this run)** | Participant 1's run through the harness. Implementation, RETRO, PERFORMANCE, filled-in session log, 21 RPI artifacts. | [github.com/bartr/movies-bartr](https://github.com/bartr/movies-bartr) |

This is the structural difference from cLLM. cLLM was one project that produced a methodology by accident. movies is a **falsifiability mechanism** — a harness designed so that participant N can produce comparable evidence to participant 1, and the methodology claim either holds across operators or it doesn't. movies-bartr is the first run; the next FDE clones the template and contributes their run as a sibling.

## Project (Run Scope)

**movies-bartr** — a Kubernetes-native read-only HTTP API serving a 100-movie/actors catalog (the same IMDb-derived test data used for Helium load tests in 2020), with Swagger, structured logs, Prometheus metrics, a Grafana dashboard, and a load-generation client (`webv`). GitOps-deployable on local k3s.

Local clones: `~/movies` (template) · `~/bartr-movies` (this run). Mirrored into core as this folder.
Implementation notes: [IMPL-README.md](IMPL-README.md) · Performance: [PERFORMANCE.md](PERFORMANCE.md) · Retro: [RETRO.md](RETRO.md)

## Engineer Profile

- **Level:** Forward Deployed Engineer (Microsoft) — same operator as cLLM
- **Domain experience:** authored the original Helium MVP scope; deep K8s/observability background
- **Reuse base:** **none.** Greenfield repo. No code or scaffolding carried forward from cLLM or Helium.

## Why This Experiment

cLLM produced **~130×** vs Helium, but with three confounds the seniority-aware caveats in [cllm/summary.md](../cllm/summary.md) explicitly flag:

1. Senior operator
2. ~1 year of K8s/GitOps scaffolding reused from a prior project
3. Stack pre-chosen by the operator (Go, K3s, Prometheus — a stack the operator was already deep in)

The harness was designed to **strip confounds 2 and 3 by construction:**

- The template is greenfield. No reusable code; participants get spec + data + methodology, nothing else.
- The spec is **stack-agnostic** — [docs/EXPERIMENT.md](https://github.com/bartr/movies/blob/main/docs/EXPERIMENT.md) is explicit that Go, Rust, Python, TypeScript, and .NET runs are all expected to land at the bar. Picking the stack is part of session 1.
- Ground rules are codified: solo only, no copy-paste from other projects, tag every session, honest retros, fit check is mandatory.

Confound 1 (operator seniority) is what *cross-participant* runs are designed to test. movies-bartr is participant 1; participant 2+ are the falsification surface.

## Scope vs Helium MVP

The scope is **functionally equivalent** to the Helium 2020 MVP, with stack substitutions called out honestly:

| Surface | Helium MVP (2020) | movies-bartr (2026) |
|---|---|---|
| API | read-only movies/actors catalog | **same** |
| Test data | 100-movie IMDb-derived JSON | **same dataset** |
| Auth | none | **same — none** |
| Swagger / OpenAPI | yes | **yes** |
| Load-test client | `webv` (C#) | `webv` (Go, rewritten) |
| Service language | C# | Go |
| Datastore | Azure Cosmos DB | local JSON files baked into image |
| Hosting | Azure App Services | k3s + Kustomize |
| Logs / dashboards | Azure Monitor | structured `slog` + Prometheus + Grafana |
| Metrics at MVP | none (added later) | Prometheus from session 1 |

Differences are **stack substitutions, not scope reductions.** The acceptance bar (service · structured logs · Prometheus · Grafana · load-test client · all §14 boxes green on a freshly-wiped cluster) is the Helium-MVP bar.

## Outcome (Participant 1)

Helium-MVP-equivalent artifact shipped in **~5 focus hours of work** by **one engineer** with no scaffolding reuse.

From the participant's [RETRO.md](RETRO.md):

> Sessions + RPI got me to 1.0.0 in ~12 hours wall-clock / ~5 hours focus, hitting spec p95 targets by 50–500× and the 500 RPS goal by ~17%. The fit check + frame-before-implement caught three premature implementations; thread reuse on context-heavy late sessions was the main methodology friction.

| Axis | Helium MVP (2020) | movies-bartr (2026) |
|---|---|---|
| Time to MVP | 26 weeks | **~5 focus hours** (~12 hours wall-clock) |
| Team | 2 Principal FDE + 1 Senior FDE + 1 Principal PM | 1 FDE |
| **People-hours** | **~4,160** | **~5** |
| Reuse base | n/a (was the new project) | none — greenfield |
| Methodology | none | sessions, intentional from session 1 |
| Tooling | Hand-written, docs, Stack Overflow | GitHub Copilot + AI-native workflow |

Raw ratio: **~830× less effort. Same engineering bar.**

That number is not the headline. The headline is: **stripping the scaffolding-reuse confound did not collapse the multiplier — it strengthened it.** That is the signal.

## Evidence

Real numbers from the repo, May 6 → May 7, 2026:

- **10 sessions** logged, **9 tags shipped** (`0.1.0` → `1.0.0`)
- **First commit → `1.0.0` tag:** ~12 hours wall-clock
- **Sum of session focus minutes:** ~5 hours (rest is breaks between sessions)
- All **7 §14 acceptance boxes green** on a freshly-wiped local k3s cluster
- **p95 list routes: 0.10–0.35 ms** (target < 50 ms) — 50–500× under spec
- **p95 detail routes: ~0.10 ms** (target < 10 ms)
- **Sustained 585 RPS, 0% 5xx, 0% 4xx** (target ≥ 500 RPS, < 1% errors)
- Honest [RETRO.md](RETRO.md) — including where the methodology got in the way

## Methodology Fingerprints in the Artifact

Things that are visible in the run repo because the methodology was followed, not because they were optional polish:

- **21 RPI artifacts in `.copilot-tracking/`** — five sessions × research/plan/changes/review files. The audit trail for *Research happened before Plan, Plan before Implement* is on disk, not asserted.
- **9 git tags as session boundaries** — the close ritual produced visible waypoints, not a single squashed commit. The session-10 reconciliation (where a post-hoc duration estimate was off by 4×) was only resolvable *because* the tag/commit timestamps existed.
- **The PERFORMANCE.md headroom argument** — the point isn't "Go is fast." The point is that 50–500× headroom means the operator can spend the budget elsewhere (richer middleware, slower disk, noisy neighbor) and the SLO still holds. That framing only happens when there's a session to write it down in.
- **Two specific instrumentation discoveries** worth keeping:
  - **Default Prometheus histogram buckets lie about sub-ms services.** `prometheus.DefBuckets` starts at 5 ms; every request landed in the smallest bucket and `histogram_quantile` interpolated ~4.75 ms across all routes. The dashboard was uniform-but-wrong for 8 sessions until session 9 caught it. Fix: custom sub-ms ladder starting at 100 µs.
  - **Go `time.Sleep` is quantized by the kernel timer to ~1 ms.** Single-thread RPS jumped in coarse steps (800 / 428 / 293) with no value near 500. Fix: two threads at `sleep=3ms` for ~584 RPS sustained.

## The Methodology Evolved From This Run

The template's most recent commit (`e882424 updated based on retro`) folded lessons from this run *back into the harness* before participant 2 starts:

- The fit check beat (originally added during the [fit-check experiment](../fit-check/summary.md)) is now load-bearing in the participant onboarding.
- The "log Start at frame, End at close ritual" rule — from [RETRO.md](RETRO.md) — is being adopted in the template's session-log to prevent post-hoc duration decay.
- The submissions tracking issue ([context-first/core#6](https://github.com/context-first/core/issues/6)) is the cross-participant evidence registry.

Provenance pattern: **rules earn their keep by surviving real sessions, not by being designed.** Same pattern as the fit-check addition. This is the methodology compounding.

## What This Replication Proves

1. **The scaffolding-reuse confound was not the multiplier.** With reuse stripped, the per-effort multiplier did not regress — it expanded. The methodology is doing more of the work than the reuse was.
2. **Sessions held under stricter conditions.** 10 framed sessions, 9 tags, fit-check applied per session. The model survived its own intentional run.
3. **The artifact is real, not a demo.** p95 50–500× under spec on a 500m-CPU pod. Acceptance criteria green on a wiped cluster.
4. **The methodology fails legibly.** The [RETRO](RETRO.md) records where it broke (thread reuse on context-heavy sessions, missing session-5 entry, post-hoc duration estimates that decayed by 4×). A methodology that makes its own failures visible is one that compounds.

## Honest Caveats

1. **Same operator as cLLM.** The seniority confound is not yet stripped. The [repos/cllm/README.md](../../repos/cllm/README.md) handoff to a different FDE is the experiment that addresses this.
2. **Focus hours, not wall-clock.** ~12 hours wall-clock to first `1.0.0` tag. The 5h figure is sum-of-session-frames; the rest is breaks, sleep, and unmeasured reading.
3. **Helium MVP people-hours estimate (~4,160) assumes 4 people × 26 weeks × 40 hrs.** Helium had real overhead (meetings, code review, design discussion) the 40 hrs doesn't capture; the actual person-hours were almost certainly higher. The ratio is therefore conservative.
4. **Tooling was Copilot, not Claude Code.** A re-run on Claude Code is the obvious next comparison.

## How This Connects to the Other Experiments

- [cllm](../cllm/summary.md) — accidental, ~130×, with reuse base, stack pre-chosen. *Discovered the methodology.*
- [fit-check](../fit-check/summary.md) — methodology-shaping, no implementation. *Sharpened the methodology.*
- **movies-bartr** — intentional, ~830×, no reuse, stack-agnostic spec, single operator. *Stress-tested the methodology under stricter conditions, and produced a reusable harness as a side effect.*
- **Participant 2+** ([github.com/bartr/movies](https://github.com/bartr/movies) submissions) — different operators, in flight. *Tests the seniority confound. The methodology becomes falsifiable when N > 1.*

## IP / Licensing

- This experiment, the harness, and the implementation are bartr's personal work, **MIT licensed**. Not Microsoft IP.
- The RPI inner-loop pattern referenced in the methodology is from Microsoft's [HVE Core](https://github.com/microsoft/hve-core) (also MIT). Credit Microsoft when referencing RPI.
- The 100-movie dataset is IMDb-derived public-domain test data, used in the original Helium load tests.

## Session Log

See [session-log.md](session-log.md). Ten sessions, nine tags, one retroactive stub (session 5), one duration reconciliation (session 10). The honest log is part of the evidence.
