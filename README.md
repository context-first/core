# Context-first

> The planning methodology for AI-native engineering teams.
>
> **Frame it. Build it. Ship it. Repeat.**

## The Problem

**The bottleneck was never capacity. It was always context.**

Agile was designed for a world where shipping software took weeks per feature. AI-native engineering changes that to hours. But most teams are still planning in points, running weekly ceremonies, and measuring velocity in sprints — a cadence built for a delivery speed that no longer exists.

The overhead has not scaled down with the delivery time. It has stayed the same, or grown. That is the problem context-first is trying to solve.

## Two Levels, One Idea

**Context** is the *why* — the bounded cognitive unit, the attention window, the thing AI-native work lives inside. It applies to engineers, PMs, managers, and ceremonies equally.

**Sessions** are the *how* — the engineering-specific instantiation of a context. A session is the unit of attention you plan, build, and ship inside.

A PM framing an arc is being context-first. A retro that closes in 60 minutes with one committed output is context-first. An engineer shipping a tagged, tested artifact in 90 minutes is context-first — and that engineering pattern is what we call a session.

## The Core Idea

Replace the story with the session.

| Old primitive: **story** | New primitive: **session** |
|---|---|
| Sized in points | Sized in focus minutes |
| Bounded by sprint calendar | Bounded by cognitive context |
| Accepted via demo | Accepted via tests + tag + repo memory |
| Not replayable | Replayable — chat, git, memory |
| Backlog grows | Compounds — each session paves the next |

A session is what one engineer + an AI coding assistant can design, build, test, and ship before context drift sets in. Typically 60–120 minutes. Always ends with a coherent artifact: green tests, FF-merge, tag, repo memory updated.

That is not a smaller story. It is a different primitive — one that matches how AI-native work actually flows.

## What Is In This Repo

```
methodology/
  sessions-not-stories.md      — the session primitive (engineering chapter)
  ceremonies-as-sessions.md    — remapping standups, retros, planning, demos
  drafts/
    ai-native-for-business.md  — the PM and stakeholder angle (draft)

experiments/
  session-log-template.md      — run your own experiment
  cllm/                        — Experiment 001 — accidental, with reuse
  fit-check/                   — Experiment 002 — methodology-shaping
  movies-bartr/                — Experiment 003 — intentional, no reuse
```

## The Experiments

Three runs so far, each at a stricter condition than the last.

| | Helium MVP (2020) | [cLLM](experiments/cllm/summary.md) (2026) | [movies-bartr](experiments/movies-bartr/summary.md) (2026) |
|---|---|---|---|
| Time to MVP | 26 weeks | 4 days | **~5 focus hours** |
| Team | 4 (3 FDE + 1 PM) | 1 FDE | 1 FDE |
| Reuse base | n/a (the new project) | ~1 yr K8s/GitOps from prior project | **none — greenfield** |
| Methodology | none | sessions (accidental) | sessions (intentional) |
| Effort | ~4,160 person-hours | ~0.8 person-weeks | **~5 person-hours** |
| Headline ratio | — | ~130× | **~830×** |

**cLLM** — synthetic GPU / vLLM simulator. 125 commits in 12 days, ~19.5K lines of Go (~40% test ratio, race-clean), 3-layer observability day one, GitOps on K3s/Flux. Public: [github.com/bartr/cllm](https://github.com/bartr/cllm). The session model emerged here by accident.

**movies-bartr** — direct replication of the Helium MVP scope (same API, same Swagger, same 100-movie IMDb dataset, read-only, no auth). Stack substitutions only: Go instead of C#, k3s instead of App Services, Prometheus/Grafana instead of Azure Monitor, local JSON instead of Cosmos DB. 10 sessions, 9 tags, p95 50–500× under spec, all acceptance criteria green on a freshly-wiped cluster. Honest [retro](experiments/movies-bartr/RETRO.md) included.

This run is also **participant 1 of a reusable experiment harness** at [github.com/bartr/movies](https://github.com/bartr/movies). The spec is stack-agnostic (Go/Rust/Python/TS/.NET runs are all expected to land at the bar), the ground rules are codified in `EXPERIMENT.md`, and the submissions tracker is open. The methodology becomes falsifiable across operators when N > 1.

### On the Ratios

The interesting finding is not the size of either number. It is the **direction of change** when a confound was removed.

cLLM had two confounds: a senior operator and ~1 year of reusable scaffolding. movies-bartr was designed to strip the second — greenfield repo, no carryover, spec written before code. If the multiplier had been mostly scaffolding-reuse, the result should have collapsed. It did not. It expanded.

That is the signal worth attending to. The 130× and the 830× are the same claim under different conditions; the second condition is stricter, and the result held. The remaining confound — operator seniority — is what the next experiment (a different FDE running the same spec) is built to test.

Two honest framings of the numbers:

- **Conservative:** cLLM's defensible floor is **HVE-only ~65×** (see `repos/cllm/docs/helium.md` "The Counterfactual"). movies-bartr's ~830× is sum-of-focus-hours, not wall-clock; the wall-clock figure was ~12 hours, which is still ~350×.
- **Falsifiable:** the prediction now is that a different operator on the same movies spec will land in the same order of magnitude. If they don't, the methodology claim weakens. That is the experiment in flight.

## The Bigger Claim

AI-native engineering amplifies engineering judgment. Sessions give that judgment a repeatable structure. Together they produce a compounding effect that neither delivers alone.

The cLLM experiment was accidental. The next ones will not be.

## Status

Early but accelerating. Three experiments. One operator on two of them; a different operator on the third (in flight). The methodology is sound and survived its first stricter condition. It needs more data points — different engineers, different domains, different experience levels. That is what the `experiments/` folder is for.

## License & Ownership

This repo, the methodology, and the experiments are bartr's personal work, **MIT licensed**. Not Microsoft IP. The RPI inner-loop pattern referenced in the methodology is from Microsoft's [HVE Core](https://github.com/microsoft/hve-core) (also MIT) — credit Microsoft when referencing RPI.

If you are running AI-native engineering and want to contribute an experiment, see [contributing.md](contributing.md).

## Who This Is For

- **Engineers** adopting AI-native workflows who want a repeatable session structure
- **PMs** looking for a lighter-weight planning primitive
- **Engineering managers** whose teams are outgrowing Scrum overhead
- **Anyone** who has felt ceremony cost exceed ceremony value

## Related Work

Context-first did not emerge from a vacuum. These are the bodies of work that overlap, inform, or inspired it — and where it parts ways with each.

### HVE Core — Microsoft

The closest *complementary* work — not the closest competitor. HVE Core ([microsoft/hve-core](https://github.com/microsoft/hve-core)) is a GitHub Copilot tooling library: specialized agents, auto-applied coding instructions, reusable prompts, and validated skills, distributed as VS Code extensions. Its signature pattern is **RPI** — Research → Plan → Implement → Review, with `/clear` between phases so each agent runs on a clean context.

HVE Core operates at the **IDE / Copilot layer** — it answers *"how do I get high-quality output from Copilot on one task?"* Context-first operates at the **planning / methodology layer** — it answers *"what is the unit of work, and how do the ceremonies wrap it?"*

The two fit together cleanly: an RPI cycle typically runs *inside* a single context-first session. HVE makes the inner loop at the keyboard fast; context-first is the outer-loop discipline (sessions, arcs, ceremonies-as-sessions, repo memory) that lets the speed actually reach delivery. For the practical "how do these compose?" guide, see [Sessions and RPI](methodology/sessions-and-rpi.md) — including a worked example you can follow without the HVE Core extension installed.

The consistent feedback from teams adopting HVE: the tools are fast, the agile process gets in the way. **Context-first is the process catching up to the tools.** Use both.

### Shape Up — Basecamp / 37signals

The closest existing framework. Shape Up replaces backlogs with "bets," estimates with "appetites," and sprints with six-week cycles that end with a hard stop — no carryover, no extensions.

The circuit breaker concept (work stops at the time box, full stop) is very session-like, and the critique of story-point estimation is the same critique context-first makes.

Where it stops: Shape Up operates at the feature and project level. It does not prescribe a daily work primitive for the individual engineer. Context-first picks up where Shape Up leaves off — and adds the AI tooling layer that did not exist when Shape Up was written.

If you know Shape Up, think of context-first as Shape Up at the engineer level, with a release discipline and a compounding mechanism built in.

### #NoEstimates — Woody Zuill, Vasco Duarte

A movement, not a framework. The core claim: estimation is waste, and teams should replace it with flow metrics and conversation.

Directionally aligned with context-first — and the critique of points-based estimation is shared. But #NoEstimates leaves open the question it raises: "if not stories and points, then what?"

The session is an answer to that question. The session is the planning primitive that #NoEstimates never quite named.

### Deep Work — Cal Newport

The cognitive science foundation for why sessions work. Newport's argument: the ability to focus without distraction is the most valuable and most rare skill in the modern economy, and it is destroyed by fragmented work culture.

Context-first is essentially Deep Work applied to AI-native engineering — with a release discipline, a close ritual, and a compounding mechanism on top. If you have not read Deep Work, it is the best single background read for understanding why the session model produces the results it does.

### Team Topologies — Skelton, Pais

Focuses on reducing cognitive load and handoff friction at the team level. The "stream-aligned team" concept and the emphasis on fast flow are directionally consistent with the session model.

Operates at a higher altitude than context-first — org structure and team interaction modes rather than individual work primitives. The two are complementary, not competing.

### Pomodoro Technique — Francesco Cirillo

Superficially similar: time-boxed work units, regular breaks, protection of focus time. A useful personal productivity tool.

The difference is depth. Pomodoro has no artifact discipline, no compounding mechanism, no release ritual, and no planning implications. A session is a richer primitive — it produces something that outlasts the timer.

### The Honest Summary

None of these frameworks combine:

- A session as the **planning primitive** (not just a productivity tool)
- A **close ritual** as the compounding mechanism
- **Repo memory** as the cross-session continuity layer
- A **ceremony remapping** that makes standups, retros, and planning all session-shaped

That combination is what context-first is trying to formalize. The prior art is real and worth knowing. The gap is also real.

## Origins

Born from a working conversation while preparing an internal AI-native engineering presentation, May 2026. The thinking is exploratory, not finished. Pick it up, challenge it, run your own experiment, and add to it.

> The methodology gets better when more engineers stress-test it.
> That is the point of making it a repo.
