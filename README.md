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
  cllm/                        — the first experiment (bartr/vllm)
    summary.md                 — what was built, how sessions mapped
    session-log.md             — 10-session intentional practice log
```

## The First Experiment

**cLLM** — a synthetic GPU / vLLM simulator built in 4 days by one engineer using GitHub Copilot and an AI-native engineering workflow.

- 125 commits in 12 days · 99 in the first 5
- ~19.5K lines of Go · ~40% test ratio · race-clean
- 3-layer observability shipped day one
- GitOps-deployed on K3s / Flux

The session model emerged from that project accidentally. This repo exists to make it intentional — and to replicate it across more engineers, domains, and experience levels.

Full project: [github.com/bartr/vllm](https://github.com/bartr/vllm)

## The Bigger Claim

AI-native engineering amplifies engineering judgment. Sessions give that judgment a repeatable structure. Together they produce a compounding effect that neither delivers alone.

The cLLM experiment was accidental. The next ones will not be.

## Status

Early. One experiment. One engineer. The methodology is sound but needs more data points — different engineers, different domains, different experience levels. That is what the `experiments/` folder is for.

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
