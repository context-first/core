# Sessions and Skill Compounding — Working Draft

> **DRAFT — NOT FOR PUBLICATION.** Working notes after Matt's movies-spec
> v1 run, before his post-v1 learning-loop session. Held internal until
> that session completes and we have evidence on the proposed remediation.

## The matched-profile comparison

For the first time, two runs at the same scope and the same employer have
matched operator profiles at different points in time:

| | Helium MVP (2020) — Anne | movies (2026) — Matt |
|---|---|---|
| Level | SE II (possibly newly Senior) | SE II |
| Age | ~30 | ~30 |
| Reporting line | bartr | bartr |
| Disposition | smart, good work effort, curious | smart, good work effort, curious |
| Tooling | none AI-native | GitHub Copilot / Claude-class agent |
| Methodology | none | sessions + RPI |
| Outcome | Helium MVP shipped (26 weeks, 4-person team) | movies 1.0.0 shipped (~9 hours focus, 7–8 sessions, solo) |

Matt's artifact result lands close to bartr's senior-operator run on the
same spec (~5 focus hours vs ~9). That is itself the headline on the
**seniority-confound axis** — the methodology mostly held with a less
senior operator. If this were the whole story it would be a clean win.

It is not the whole story.

## The four differences (what's actually surprising)

Reflecting on Matt-v1 against Helium-Anne, four differences stand out.
The first is a methodology win. The next three are blind spots the
methodology has no current name for.

### 1. Implementation speed — the win

AI-native + sessions delivered the artifact in roughly the ratio the
existing experiments predicted. This is what the README is already
correct about. No new claim.

### 2. Learning during the run — the gap

Anne, over Helium MVP, made **100+ dashboard edits in the Grafana UI**
and dozens of corresponding changes in Prometheus. By the end of
Helium she was recognized inside the org as a strong observability
engineer — Prometheus, Grafana, Kubernetes, inner-loop / outer-loop.
"KiC" (Kubernetes in Codespaces) emerged from that work. Anne's
career compounded from what she *learned*, not from what she shipped.

Matt, over movies-v1, **did not know that Grafana dashboards could be
edited in the UI.** The dashboard he shipped is the one the agent
generated. Substantial Prometheus / Grafana / k3s exposure went past
him without becoming skill.

The honest framing: in 2020 the operator had no choice but to learn
the platform — there was no agent to do it for them. In 2026 the agent
will do it, and unless the session explicitly creates a learning beat,
**the operator skips the skill-acquisition step that used to be a
side-effect of shipping.**

This is not Matt's failure. It is the methodology being silent on
something Helium got for free from its tooling constraints.

### 3. Engineering fundamentals — the gap

Helium, as a new team, codified what engineering fundamentals ("EF")
meant for the team and the broader org — commit granularity, code
review cadence, what good looks like, what is not acceptable. Many of
those norms are still in use a decade later.

Matt-v1 review surfaced specific EF gaps:

- **One commit per session** — far coarser than the granularity the
  team norm calls for. Sessions and commits are different units; a
  session typically contains many commits.
- **No code reviews until the very end of the run.** The review-as-you-go
  loop never happened.
- **Did not ask Claude for coaching.** "How could I have done this
  better?" is a free, available, high-leverage prompt that did not get
  used.

These are coaching findings, and they are good findings *for Matt*. But
the methodology has no current beat that produces them. The Helium team
produced them by working together day-in-day-out under pressure; the
solo session model does not have an analogue.

### 4. Team culture — the gap (separate draft)

Helium was the first project for a new team. The team used it to define
how they worked together, what the quality bar was, and what behavior
was and was not acceptable. Many of those norms persist today —
including for bartr.

A solo experiment cannot produce this by construction. Sessions are an
**individual primitive**; culture is **multi-person** and forms in the
gaps *between* sessions — code reviews, quality-bar disagreements,
calibration of what "done" means, what gets pushed back on.

Treated separately in
[team-culture-as-sessions.md](team-culture-as-sessions.md).

## What this means for the methodology

The README claims the methodology "fails legibly." Matt-v1 is the first
run where it failed legibly on a **systematic blind spot**, not a
tactical bug. The fit-check addition (from
[fit-check](../../experiments/fit-check/summary.md)) and the missing
session-5 entry (from movies-bartr's
[RETRO](../../experiments/movies-bartr/RETRO.md)) were tactical fixes.
This is structural: sessions optimize for shipping coherent artifacts
and are silent on the human-development outcomes that the pre-AI
delivery cycle produced as a side effect.

That is publishable as an observation. The remediation is not yet
publishable, because no run has tested it.

## The hypothesis under test (Matt-v2)

The proposed remediation is **not yet methodology**. It is the
hypothesis Matt-v2 will test:

- **Per-release code review with Claude and bartr.** Each tag is a
  review checkpoint, not a ship-and-forget waypoint.
- **Per-release learning retro.** Two questions: what did I learn about
  the platform; what did I miss that the agent did for me. Recorded.
- **Explicit coaching prompts to Claude.** "How could I have done this
  better?" / "What did I miss?" — at least once per session.
- **Commit granularity coaching.** Many commits per session, not one.
  Reviewed in the per-release checkpoint.
- **Deliberate platform exposure.** When the agent reaches for a tool
  Matt hasn't used (Grafana UI editing, Prometheus query language,
  kubectl primitives), the session pauses and Matt drives that step.

If Matt-v2 closes the learning / EF gap **and** the artifact bar still
holds, these beats earn a place in the methodology. If the gap closes
but the artifact takes 3× longer, the trade-off is the new finding. If
the gap doesn't close, the hypothesis is wrong and we keep looking.

Per the repo's own provenance pattern: rules earn their keep by
surviving sessions, not by being designed.

## Confounds the v1 → v2 comparison cannot strip

- **Matt has now seen the spec.** Matt-v2 is not a clean N=2 participant
  replication. It is a same-operator second run with a learning loop
  applied. To re-strip the seniority confound, a third participant who
  has not seen the spec is still needed.
- **The Hawthorne effect is real.** Matt knowing he is being coached
  changes his behavior. That is fine for testing the remediation; it is
  not a clean test of "what would a typical SE II do unprompted."
- **Bart is the coach and the methodology author.** Independent review
  of the v2 artifact (and its honest retro) would strengthen the
  evidence.

## Open questions

- Is "per-release code review + learning retro" the right shape, or is
  a separate **learning beat** *inside* the session (between Implement
  and Review) more honest?
- Does the methodology need an explicit **skill-compounding artifact**
  alongside the code artifact — e.g. a one-paragraph "what I learned"
  appended to repo memory at close ritual?
- For multi-person teams (which solo experiments cannot test): does the
  per-release code review naturally absorb the EF coaching, or does EF
  need its own beat?
- Where does this live in the methodology layout? An addition to
  [sessions-not-stories.md](../sessions-not-stories.md), or a new
  doc on "session-as-development-loop, not just delivery-loop"?

## Status of v1 facts (held internal)

- Sessions: 7–8
- Total focus time: ~9 hours
- Outcome: artifact bar held (movies-spec acceptance criteria green)
- Comparable to bartr's ~5-hour senior run on the same spec
- Not yet published; held until the post-v1 learning-loop session
  completes and Matt has agreed on the public framing

## What we are not doing yet

- No `experiments/movies-matt/` folder
- No README or index.html updates
- No public mention of Matt by name
- No methodology beat additions (only this draft)
- No Matt-v2 framing committed to the repo

All of the above unlock once Matt-v2 has run and there is honest
evidence on whether the proposed remediation works.
