# Experiment 003: movies-bartr

> A direct, intentional replication of the Helium MVP. Same scope. Single operator. Methodology-driven.

## Project

**movies-bartr** — a Kubernetes-native read-only HTTP API serving a 100-movie/actors catalog (the same IMDb-derived test data used for Helium load tests in 2020), with Swagger, structured logs, Prometheus metrics, a Grafana dashboard, and a load-generation client (`webv`). GitOps-deployable on local k3s.

Local clone (gitignored): `repos/movies/`
Implementation notes: [IMPL-README.md](IMPL-README.md) · Performance: [PERFORMANCE.md](PERFORMANCE.md) · Retro: [RETRO.md](RETRO.md)

## Engineer Profile

- **Level:** Partner FDE (Microsoft) — same operator as cLLM
- **Domain experience:** authored the original Helium MVP scope; deep K8s/observability background
- **Reuse base:** **none.** Greenfield repo. No code or scaffolding carried forward from cLLM or Helium.

## Why This Experiment

cLLM produced **~130×** vs Helium, but with two confounds the seniority-aware caveats in [cllm/summary.md](../cllm/summary.md) explicitly flag:

1. Senior operator
2. ~1 year of K8s/GitOps scaffolding reused from a prior project

movies-bartr was designed to **strip the second confound** (the first persists — a different operator is the next experiment, [repos/cllm/README.md](../../repos/cllm/README.md), already handed off). Greenfield repo. Spec was written before any code. Stack chosen at session 1.

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

## Outcome

Helium-MVP-equivalent artifact shipped in **~5 focus hours of work** by **one engineer** with no scaffolding reuse.

| Axis | Helium MVP (2020) | movies-bartr (2026) |
|---|---|---|
| Time to MVP | 26 weeks | **~5 focus hours** (~12 hours wall-clock) |
| Team | 2 Principal FDE + 1 Senior FDE + 1 Principal PM | 1 Partner FDE |
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

- [cllm](../cllm/summary.md) — accidental, ~130×, with reuse base. *Discovered the methodology.*
- [fit-check](../fit-check/summary.md) — methodology-shaping, no implementation. *Sharpened the methodology.*
- **movies-bartr** — intentional, ~830×, no reuse, single operator. *Stress-tested the methodology under stricter conditions.*
- `repos/cllm` (handoff) — different operator, in flight. *Tests the seniority confound.*

## Session Log

See [session-log.md](session-log.md). Ten sessions, nine tags, one retroactive stub (session 5), one duration reconciliation (session 10). The honest log is part of the evidence.
