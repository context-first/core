# Sessions, Not Stories — The Session Primitive

> The engineering chapter of [context-first](../README.md), in long form. Sessions are the unit of attention engineers plan, build, and ship inside.

## The Shift in Primitive

Old unit of planning: **a story** — sized in points, scoped to fit a sprint, accepted via a demo.

New unit of planning: **a session** — sized in *focus minutes*, scoped to "what one engineer + an AI coding assistant can ship before context drift," accepted via tests + commit + (often) a release.

A session has properties a story doesn't:

- **It is bounded by cognitive context, not calendar time.** A 90-minute session with a clean repo and a sharp goal beats two days of fragmented work. The constraint is *attention*, not *capacity*.
- **It produces a coherent artifact.** Code + tests + design notes + a release commit, ideally. Sessions that don't produce all four are a smell.
- **It is replayable.** The chat transcript, repo memory, and git history are the audit trail. Stories were never replayable; sessions are.
- **It compounds.** What repo memory captured this session makes the next session faster. Stories don't compound; backlogs just get longer.

## What This Changes for Each Role

### Developers

- Plan in sessions, not tickets. A "release-able session" becomes the natural commit unit.
- The dot-release-per-session rhythm is the pattern. Codify it: every session ends with green tests, FF-merge, tag, repo-memory update.
- Context hygiene becomes a first-class skill: prepping the session (repo memory, design doc, scope) is now part of the work, not overhead.

### PMs

- Stop estimating in points. Estimate in **sessions to first demo** and **sessions to ship**. Way more honest.
- The PM's job shifts from "decompose into stories" to "frame the session" — goals, constraints, exit criteria, what *not* to do. That is a higher-leverage skill, and a great fit for AI-native engineering.
- Roadmaps become "this quarter, we expect ~N session-equivalents of capacity per engineer" — testable against actuals and self-correcting.

### Planning

- Velocity becomes **sessions-per-week to value** — a number you can actually feel.
- Risk shifts from "did we estimate right?" to "did we frame right?" Mis-framed sessions waste a session; mis-estimated stories used to waste a sprint. Smaller blast radius, faster correction.
- Cross-team dependencies get easier: "I need 1 session of your time on X" is much clearer than "a 5-point story."

## Greenfield Cadence — Staying Current

The session model implies a corollary for individual development: **one greenfield project per half, budgeted at ~10–12 sessions.**

Why it matters:

- The landscape moves fast. Six months of customer work without a net-new build quietly erodes the instincts AI-native engineering depends on.
- Greenfield is where you encounter new stacks, new domains, and new failure modes — the raw material that makes your next customer engagement sharper.
- 10–12 sessions is small enough to protect on any calendar, and large enough to produce real, shippable evidence.

In session terms: this is one arc, not a project. Frame it, time-box it, ship something real, write the story. Then bring it back to customers.

## What's Tricky

- **Sessions are personal.** My session ≠ a junior engineer's session. The unit does not natively normalize across people. (Stories had the same problem; we just pretended they didn't.)
- **Big things still take many sessions.** We need a "campaign" or "arc" concept above sessions — basically what an epic was, but composed of sessions and explicit memory checkpoints between them.
- **Ceremonies need to change.** See [ceremonies-as-sessions.md](ceremonies-as-sessions.md).
- **Reporting up.** Leadership wants quarter-level numbers; sessions roll up awkwardly. We will need a translation layer: *X sessions = Y dot releases = Z customer-visible features.*

## Open Questions

- What is a sensible session-length norm? (Hypothesis: 60–120 minutes, gated by attention not clock.)
- Is "release-able session" the right exit bar, or too strict for early-arc work?
- How do we represent arcs / campaigns without rebuilding the epic-and-story machinery we are trying to replace?
- How do PMs and engineers co-frame a session without re-introducing ticket overhead?
- What does this look like for non-greenfield work (large existing codebases, legacy)?

## Making It Intentional

The session model produces results even when it is accidental — but it compounds faster when it is deliberate.

**Before each session — the frame.**
Two minutes, three questions:
- What does done look like for this session?
- What is explicitly out of scope?
- What is the one thing that would make this session a failure?

If you cannot answer all three, the session is not ready to start.

**During the session — one rule.**
When you feel the urge to pull a thread that's outside your frame, don't. Write it down in a "next sessions" list and stay in scope. Context drift is the enemy — and it's seductive because the tangent usually feels important in the moment.

**Closing the session — the ritual.**
Every session ends with:
- Green tests
- FF-merge and tag
- Repo memory updated
- One paragraph: what you built, what you decided, what the next session should start with

That paragraph is the compounding mechanism. Without it, each session starts cold.

**Across sessions — the meta layer.**
Keep a session log. After each session, one line: goal, outcome, drift (yes/no), close ritual complete (yes/no). By session 5 you have enough data to see your own patterns — where you drift, where framing was weak, where the close ritual got skipped. That is the feedback loop that makes it intentional rather than accidental.

See [session-log-template.md](../experiments/session-log-template.md) for a fillable template.

## Connection to AI-Native Engineering

Working AI-native means most of the bottlenecks engineering used to fight have moved.

- **Implementation speed is no longer the constraint.** Framing quality is.
- **Documentation is no longer the cost.** It is a side-effect of the close ritual.
- **Memory across context boundaries is no longer manual.** Repo memory and chat transcripts carry it.

Sessions are the planning primitive that fits this world. Stories were the planning primitive for a world where shipping took weeks. The cadence has changed; the primitive needs to change with it.
