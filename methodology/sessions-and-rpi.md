# Sessions and RPI — How They Compose

> A practical guide for using [context-first sessions](sessions-not-stories.md) (the outer loop) together with HVE Core's [RPI workflow](https://github.com/microsoft/hve-core/blob/main/docs/rpi/README.md) (the inner loop) — without needing the HVE Core extension installed.

The two methodologies were designed independently and converge cleanly. Sessions answer *"what is the unit of work, and how do ceremonies wrap it?"* RPI answers *"how do I get high-quality output from an AI assistant on one task?"* They sit at different altitudes. This page explains how to run both at once.

## The Two Altitudes

```
┌─ Session (outer loop) ─────────────────────────────────────────┐
│  Frame                                                         │
│  ┌─ RPI cycle (inner loop) ───────────────────────────────┐    │
│  │  Research  →  Plan  →  fit check  →  Implement  →  Review │
│  └────────────────────────────────────────────────────────┘    │
│  Close (green tests · FF-merge · tag · repo memory · starter)  │
└────────────────────────────────────────────────────────────────┘
```

- **Session** is bounded by **cognitive context** (~90–120 focus minutes) and exits with a tagged dot-release.
- **RPI** is bounded by **task verification** and exits with reviewed code.
- One session ≈ one RPI cycle. If you need more than one cycle, your session is probably two sessions.

## RPI in 90 Seconds

You do not need the [hve-core](https://github.com/microsoft/hve-core) extension to run RPI. The agents and slash commands are sugar; the workflow is the substance. Anyone with any AI assistant can do this by convention.

The core insight: **AI assistants cannot tell the difference between investigating and implementing.** Asked for code, they write code — including code for APIs that do not exist and patterns that do not match your codebase. RPI fixes this by running each phase under a constraint that *prevents* the next one. When the AI knows it cannot implement, it stops optimizing for plausible code and starts optimizing for verified truth.

The four phases:

| Phase | Constraint | Output |
|---|---|---|
| **Research** | Cannot plan or write code. Must investigate codebase and external sources, cite findings with file paths and line numbers, and recommend one approach. | `YYYY-MM-DD-<topic>-research.md` |
| **Plan** | Cannot research or write code. Must turn the research into an ordered, checkbox-able task list with line-number-precise references. | `YYYY-MM-DD-<topic>-plan.md` |
| **Implement** | Cannot replan. Executes the plan task by task, recording every change. | Working code + `YYYY-MM-DD-<topic>-changes.md` |
| **Review** | Cannot modify code. Validates the implementation against research and plan; runs lint/build/test; identifies follow-up work. | `YYYY-MM-DD-<topic>-review.md` |

**The non-obvious rule: `/clear` (or start a new chat) between every phase.** Findings live in files, not in chat history. A clean context per phase is what makes the constraints stick.

Artifacts conventionally land in `.copilot-tracking/` at the repo root. Use any folder you like — what matters is that the next phase can open the previous phase's file.

For the full treatment, see HVE Core's [RPI guide](https://github.com/microsoft/hve-core/blob/main/docs/rpi/README.md) and the per-phase pages ([Researcher](https://github.com/microsoft/hve-core/blob/main/docs/rpi/task-researcher.md), [Planner](https://github.com/microsoft/hve-core/blob/main/docs/rpi/task-planner.md), [Implementor](https://github.com/microsoft/hve-core/blob/main/docs/rpi/task-implementor.md), [Reviewer](https://github.com/microsoft/hve-core/blob/main/docs/rpi/task-reviewer.md)).

## How One Session Maps to One RPI Cycle

| Session beat | What you do | RPI phase |
|---|---|---|
| **Frame** (2 min, before) | Goal · out-of-scope · failure condition. See [sessions-not-stories.md](sessions-not-stories.md) §Making It Intentional. | — (precedes RPI) |
| **Research** | One concise prompt to your AI: "Research only — investigate X, cite findings, recommend one approach. Do not plan or implement." Save output to `.copilot-tracking/`. | Research |
| **`/clear`** | New chat. Open the research file in your editor so the next phase can see it. | — |
| **Plan** | "Plan only — read the research file, produce an ordered task list with line refs and exit criteria. Do not implement." | Plan |
| **Fit check** (2 min) | Compare plan to frame: *will this fit in 90–120 min?* If no, name the smallest cut. Decide: proceed, cut, or re-frame. Record the decision in the session log. | — (the seam between Plan and Implement) |
| **`/clear`** | New chat. Open the plan file. | — |
| **Implement** | "Implement only — execute the plan task by task. Verify after each. Record changes." | Implement |
| **`/clear`** | New chat. Open plan + changes log. | — |
| **Review** | "Review only — validate the implementation against the research and plan. Run lint/build/test. Identify follow-ups." | Review |
| **Close** (5 min, after) | Green tests · FF-merge · tag · repo memory updated · one-paragraph summary · next-session starter. | — (succeeds RPI) |

**The fit check** is the only beat that does not exist in either methodology on its own. It emerged from running them together — see [sessions-not-stories.md §Making It Intentional](sessions-not-stories.md) for why it earns its keep.

## A Worked Example (One Session)

Concrete shape from a real session in the [fit-check experiment](../experiments/fit-check/) — drafting a stack-pick session for a greenfield Go service.

**Frame** (written into the session log before starting)

- Goal: Service end-to-end on localhost — data loaded, all `GET` endpoints with validation, integration tests, ≥80% coverage.
- Out of scope: `/healthz`, `/metrics`, container, k8s manifests.
- Failure condition: schema invented rather than inferred from the data files; any validation rule missing a negative test.

**Research** → `repos/<repo>/.copilot-tracking/2026-05-06-stack-research.md`

A short doc comparing Go/Rust/Python/TS for a small read-only API. Each column cited (image size, cold start, Prometheus client maturity, OpenAPI story). One recommendation with explicit "what we are NOT taking" — chi, viper, zap. Decision criteria traceable to the spec.

**Plan** → `repos/<repo>/.copilot-tracking/2026-05-06-service-mvp-plan.md`

Numbered task list: skeleton → inferred types → store → router → handlers → tests → smoke. Each task has a checkbox, file targets, and exit criteria. Parking lot for decisions deferred to Implement (envelope shape, default page size, 404 body shape).

**Fit check**

Plan walked through honestly: tests + coverage gate alone are 25–30 min; ambiguous response shapes need a 2-min "freeze" beat at the top. Stretch item (`/swagger` UI + OpenAPI doc) is 45–60 min on its own. Decision recorded in the session log: **proceed without stretch**; defer `/swagger` to next session as a carry-in. This decision saves the session.

**Implement**

Ten focused minutes of "freeze ambiguities," then mechanical execution against the plan. AI writes most of the code from the plan + data files. Negative tests are table-driven, one row per validation rule.

**Review**

Coverage report green. One follow-up logged: a year-bound assumption that should be re-verified once data grows. Closes cleanly.

**Close ritual**

`go test -race ./...` green · PR opened, self-reviewed, `gh pr merge --rebase --delete-branch` · `git tag 0.1.0 && git push origin 0.1.0` · repo memory updated with the inferred schema decisions · one paragraph in the session log + next-session starter naming `/swagger` as the carry-in for Session 2.

## You Do Not Need the Extension

The HVE Core VS Code extension provides custom agents and `/task-research` · `/task-plan` · `/task-implement` · `/task-review` slash commands that pre-load the right system prompts. They are excellent if you can install them.

If you cannot — different IDE, different AI, locked-down environment — RPI still works. The four phases are a *workflow*, not a tool. A short prompt at the start of each phase ("Research only. Do not plan or implement. Output a file at the path below.") substitutes for the agent. The `/clear`-between-phases rule and the per-phase artifacts are what carry the load.

This is also why context-first does not require HVE Core. They compose; neither depends on the other.

## What Each Methodology Adds

|  | Without RPI | Without sessions |
|---|---|---|
| **Quality** | AI guesses; "plausible code" instead of verified code | RPI runs but has no exit ritual; work piles up unmerged |
| **Cadence** | Sessions still close cleanly; just slower | Inner loop is sharp but does not compound across days |
| **Compounding** | Repo memory + close ritual still work | Research/plan/review artifacts exist but no one reads them next time |
| **Framing** | The frame still works | RPI plans don't get pressure-tested against time |

Sessions provide the **shape and rhythm**. RPI provides the **rigor inside one cycle**. The fit check is what makes them aware of each other.

## When to Skip Either

- **Skip RPI for trivial tasks.** Typo fixes, log statements, refactors under ~50 lines. The RPI overhead exceeds the work.
- **Skip the formal session frame for spike work** where the goal is genuinely *"see what's possible in 30 minutes."* Spikes are not sessions; they are exploration. Convert their output into a session frame for the *next* time block.
- **Never skip the fit check** once you've written a plan. It costs two minutes and is the highest-leverage beat in the entire workflow.

## See Also

- [sessions-not-stories.md](sessions-not-stories.md) — the engineering chapter; the full session model.
- [ceremonies-as-sessions.md](ceremonies-as-sessions.md) — how standups, retros, planning, and demos remap onto sessions.
- [HVE Core RPI documentation](https://github.com/microsoft/hve-core/blob/main/docs/rpi/README.md) — the upstream source for the inner loop.
- [experiments/fit-check/](../experiments/fit-check/) — the experiment whose dry-run produced the fit-check beat.
