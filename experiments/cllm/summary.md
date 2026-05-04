# Experiment 001: cLLM

> The first experiment. Accidental session model. Real evidence.

## Project

**cLLM** — a synthetic GPU / vLLM simulator with a Chat Completions–compatible proxy, benchmark client, MCP server, and full 3-layer observability (Grafana / Prometheus), GitOps-deployed on K3s / Flux.

Repo: [github.com/bartr/vllm](https://github.com/bartr/vllm)

## Engineer Profile

- **Level**: Partner FDE (Microsoft)
- **Domain experience**: 5 years compounding K8s / Prometheus / Grafana / GitOps; first time on vLLM and GPUs-on-K8s
- **Reuse base**: ~1 year of K8s-on-the-edge + GitOps work from a Domino's project carried forward as the starting stack

## Outcome

Comparable engineering bar to Helium (2020) — a 26-week, 4-person Kubernetes platform — shipped in **4 days** by **one engineer** using GitHub Copilot and an AI-native engineering workflow.

| Axis | Helium (2020) | cLLM (2026) |
|---|---|---|
| Time to MVP | 26 weeks | 4 days |
| Team | 2 Principal FDE + 1 Senior FDE + 1 Principal PM | 1 Partner FDE |
| **People-weeks** | **~104** | **~0.8** |
| Tooling | Hand-written, docs, Stack Overflow | GitHub Copilot + AI-native workflow |

**~130× less effort. Same engineering bar.**

## Evidence

Real numbers from the repo, April 18 → May 2, 2026:

- **125 commits** in 12 days · **99 commits in the first 5 days**
- **~19.5K lines of Go** · **~7.8K lines of Go tests** (≈40% test ratio)
- **~1.6K lines of Python** (MCP server)
- **~8.3K lines of K8s YAML** (Flux-managed)
- **~6.2K lines of Markdown** — design docs, ADRs, evidence reports
- **5 design docs** in `docs/`, co-written with Copilot
- **5 scenarios + 5 evidence reports** in `benchmark/`
- **`go test -race ./...`** — green
- **3-layer observability** (cLLM control plane · vLLM serving · GPU hardware) shipped on day one

## How Sessions Mapped to the Work

The session model was not designed up front. It emerged from how the work actually flowed:

- Most days produced one or more **release-able sessions** ending in `make deploy` + smoke + tag
- The **dot-release rhythm** matched the session rhythm almost 1:1 — 60–120 minutes of focused work, ending with tests + tag + repo memory update
- **Repo memory** (`/memories/repo/*.md`) became the cross-session compounding mechanism
- Mid-project, the team switched from squash-merge to **FF-merge** so the per-session arc would survive on `main` — itself a session-shaped decision

## What Surprised Me

1. **The session boundary emerged naturally.** I did not set out to work in 60–120 minute units. That just happened to be the bound where a coherent diff + test pass + design note actually fit together.
2. **Repo memory was the unlock.** Without it, every session started cold. With it, Copilot picked up where the last session ended — and the multiplier compounded.
3. **The close ritual mattered more than the work.** Sessions that ended cleanly produced 2× the value of sessions that did not — because the *next* session started fast.
4. **GPUs-on-K8s and vLLM were both new to me.** The multiplier held even without prior domain expertise. That was the strongest evidence the workflow is real.

## What I Would Do Differently

- **Frame each session in writing before starting.** I did not. Some sessions drifted because the goal was implicit. The best sessions had a one-paragraph frame; the weakest did not.
- **Treat every project as an experiment from day one.** I lost some Git history and chat transcripts to squash-merge and inattention. The replayable property of sessions only works if you preserve the artifacts.
- **Update repo memory more aggressively.** I caught up at the end. Doing it at every session close would have produced a tighter compounding curve.

## Caveats

Four honest caveats that explain the multiplier without removing it:

1. I am one of the most senior ICs in my org. AI-native engineering rewards experience.
2. I have built Helium-shaped projects before. Repeat domain helps.
3. I reused a year of K8s/GitOps work from a Domino's edge project as the starting stack.
4. This is one project, not a fleet. More data points needed.

A junior engineer might see 3–10× rather than 130×. The multiplier scales with judgment.

## Tooling

- **GitHub Copilot** — primary AI assistant, in the IDE
- **Claude / Claude Code** — design conversations, methodology drafting
- **Repo memory** (`/memories/repo/*.md`) — cross-session continuity
- **GitHub Actions / Flux** — CI/CD and GitOps
- **Marp** — presentations as markdown

## Session Log

See [session-log.md](session-log.md). The cLLM sessions were largely accidental; the next 10 sessions on this repo (context-first) are the first intentional run of the methodology, and are logged there.
