# Study Guide — Methodology (domain L, reading-list form)

> **DRAFT — NOT FOR PUBLICATION.** Thirteenth and final
> instance of the study-guide format, in **reading-list
> form** — an annotated index into the existing
> methodology documents, not a re-teach. Scoped from
> domain L of [skills-inventory.md](skills-inventory.md).

## Why this exists — and why it's a reading list

The methodology this curriculum sits on is already
documented, in canonical form, in
`repos/context-first/methodology/`. Six documents,
written across 2024–2026, capture the full picture:
what a session is, how it composes with HVE Core's
RPI inner loop, how ceremonies fold into the session
shape, how skills compound across sessions (and where
they don't), and how the same primitive extends to
team-formation and business-user contexts.

Writing a 12-module study guide for domain L would
duplicate that content with less authority. **The
right artifact is an annotated index** — a single
page that tells the operator which doc to read for
which question, in what order, with what to look for.

That's what this is.

**What this guide is anchored in:**

- The six existing methodology docs (paths below).
- The curriculum-wide rule: methodology lives in
  `methodology/`; this index page lives in
  `learning-library/` alongside the study guides
  and skills inventory as a pointer, not as a re-teach.
- [skills-inventory.md](skills-inventory.md) domain
  L (the 5 L-rows the rest of the curriculum
  references).

## How to use this guide

Three patterns, depending on why you're reading:

1. **New operator joining the project** — read the
   docs in the **onboarding order** below (~90
   minutes). Then run your first session.
2. **Operator with a specific question** — jump to
   the **lookup table** and find the doc that
   answers it.
3. **Operator preparing to teach this to someone
   else** — read the **canonical sequence** below
   in the order written, so the "why each doc
   exists" arc is visible.

## The six methodology documents

### Published (in `methodology/`)

| File | One-line purpose |
|---|---|
| [`sessions-not-stories.md`](../methodology/sessions-not-stories.md) | **The session primitive.** What a session is, what changes for developers / PMs / planning, the greenfield-cadence corollary. |
| [`sessions-and-rpi.md`](../methodology/sessions-and-rpi.md) | **How sessions compose with HVE Core's RPI inner loop.** Outer loop (session) wraps inner loop (Research → Plan → Implement → Review). One session ≈ one RPI cycle. |
| [`ceremonies-as-sessions.md`](../methodology/ceremonies-as-sessions.md) | **How Agile ceremonies fold into the session shape.** Code review as a session not a queue; retro → reflection session; standup → synchronized code-review window. |

### Drafts (in `methodology/drafts/`)

| File | One-line purpose | Status |
|---|---|---|
| [`sessions-and-skill-compounding.md`](../methodology/drafts/sessions-and-skill-compounding.md) | **Where the methodology has blind spots.** Matched-profile comparison (Helium-Anne 2020 vs movies-Matt 2026), four observed differences, three of which are methodology gaps. **The honest retrospective.** | Held until Matt's post-v1 learning-loop session completes |
| [`ai-native-for-business.md`](../methodology/drafts/ai-native-for-business.md) | **Sessions for business users.** Decision sessions, planning sessions, working sessions — the same primitive without code. | Early draft |
| [`team-culture-as-sessions.md`](../methodology/drafts/team-culture-as-sessions.md) | **The team-formation gap.** Sessions are an individual primitive; culture is multi-person. The Helium-MVP team-formation arc as the historical evidence. | Held until multi-person session evidence is in hand |

## Onboarding order (new operator, ~90 minutes)

1. **[`sessions-not-stories.md`](../methodology/sessions-not-stories.md)** *(20 min)* — read end to end. This is the unit of work the rest of the curriculum sits inside.
2. **[`sessions-and-rpi.md`](../methodology/sessions-and-rpi.md)** *(25 min)* — the inner-loop discipline. Note the "/clear between every RPI phase" rule and the artifact convention (`.copilot-tracking/`).
3. **[`ceremonies-as-sessions.md`](../methodology/ceremonies-as-sessions.md)** *(20 min)* — only the sections relevant to your role (developer reads code-review + retro; PM reads planning).
4. **[`sessions-and-skill-compounding.md`](../methodology/drafts/sessions-and-skill-compounding.md)** *(15 min)* — the honest retro. Read the "named gap" sections most actively; this is where the methodology is least confident about itself.
5. **First session** *(60–90 min)* — pick a small change, frame it, run one RPI cycle inside one session, close cleanly. The reading is in the books; the skill is in the close ritual.

Total: ~90 min reading + first session.

## Lookup table — "I have a question about ___"

| Question | Doc | Section |
|---|---|---|
| What is a session? | [`sessions-not-stories.md`](../methodology/sessions-not-stories.md) | top |
| What's the close ritual? | [`sessions-not-stories.md`](../methodology/sessions-not-stories.md) | "Making It Intentional" |
| How long should a session be? | [`sessions-not-stories.md`](../methodology/sessions-not-stories.md) | "Bounded by cognitive context, not calendar time" |
| What is RPI? | [`sessions-and-rpi.md`](../methodology/sessions-and-rpi.md) | "RPI in 90 Seconds" |
| Why `/clear` between RPI phases? | [`sessions-and-rpi.md`](../methodology/sessions-and-rpi.md) | "/clear (or start a new chat) between every phase" |
| What's the fit check? | [`sessions-and-rpi.md`](../methodology/sessions-and-rpi.md) | "How One Session Maps to One RPI Cycle" |
| How should code review work? | [`ceremonies-as-sessions.md`](../methodology/ceremonies-as-sessions.md) | "Code Review" |
| What replaces standup? | [`ceremonies-as-sessions.md`](../methodology/ceremonies-as-sessions.md) | "Daily Standup → Synchronized Code Review Window" |
| What replaces sprint retro? | [`ceremonies-as-sessions.md`](../methodology/ceremonies-as-sessions.md) | "Sprint Retro → Reflection Session" |
| Where does the methodology fall short? | [`sessions-and-skill-compounding.md`](../methodology/drafts/sessions-and-skill-compounding.md) | "The four differences" |
| Why coaching matters | [`sessions-and-skill-compounding.md`](../methodology/drafts/sessions-and-skill-compounding.md) | "Asking the agent for coaching" |
| Can business users run sessions? | [`ai-native-for-business.md`](../methodology/drafts/ai-native-for-business.md) | "The Three Session Types That Matter Most" |
| How do teams form across sessions? | [`team-culture-as-sessions.md`](../methodology/drafts/team-culture-as-sessions.md) | "The thesis" |
| Why doesn't the methodology cover team formation? | [`team-culture-as-sessions.md`](../methodology/drafts/team-culture-as-sessions.md) | "The historical evidence" |

## Canonical sequence — how the docs were written

Read in this order to see the arc:

1. **[`sessions-not-stories.md`](../methodology/sessions-not-stories.md)** — the original engineering chapter. Established the primitive.
2. **[`sessions-and-rpi.md`](../methodology/sessions-and-rpi.md)** — composed the primitive with HVE Core's inner loop. The "two altitudes" diagram makes the relationship visible.
3. **[`ceremonies-as-sessions.md`](../methodology/ceremonies-as-sessions.md)** — applied the primitive backward to Agile, showing what compresses, what stays, what disappears.
4. **[`ai-native-for-business.md`](../methodology/drafts/ai-native-for-business.md)** — applied the primitive outward to non-engineering work. Early draft; the business-user angle started feeling like its own topic.
5. **[`sessions-and-skill-compounding.md`](../methodology/drafts/sessions-and-skill-compounding.md)** — applied the primitive *to itself*. The matched-profile comparison surfaced three things the methodology has no name for. The honest retrospective.
6. **[`team-culture-as-sessions.md`](../methodology/drafts/team-culture-as-sessions.md)** — named the largest of those gaps. Sessions are individual; culture is multi-person. The methodology has nothing to say about team formation yet, and this doc says so directly.

The arc: **establish, compose, retrofit, extend, critique, name the gap.** That's the shape of an honest methodology — not a manifesto but a working artifact that keeps surfacing its own limits.

## The L-domain skills, mapped to docs

From the inventory:

| Skill | Where it lives |
|---|---|
| L1 — Sessions + RPI cycle | [`sessions-not-stories.md`](../methodology/sessions-not-stories.md) + [`sessions-and-rpi.md`](../methodology/sessions-and-rpi.md) |
| L2 — Frame → Plan → Fit-check → Implement → Review → Close | [`sessions-and-rpi.md`](../methodology/sessions-and-rpi.md) "How One Session Maps to One RPI Cycle" |
| L3 — RETRO honesty | [`sessions-and-skill-compounding.md`](../methodology/drafts/sessions-and-skill-compounding.md) is the worked example of this skill applied to itself |
| L4 — Asking the agent for coaching | [`sessions-and-skill-compounding.md`](../methodology/drafts/sessions-and-skill-compounding.md) — the named gap from Matt-v1 |
| L5 — Per-release human + Claude review | The proposed remediation in [`sessions-and-skill-compounding.md`](../methodology/drafts/sessions-and-skill-compounding.md); operationalized across the 12 study guides in `learning-library/` (per-release review template in [`study-guide-observability.md`](study-guide-observability.md)) |

## What this guide is

- An annotated index into the six existing methodology
  documents.
- A reading order for new operators (~90 min).
- A lookup table by question.
- The map from inventory L-rows to source docs.

## What this guide is not

- Not a re-teach. The canonical text lives in
  `methodology/`; this page points there.
- Not a substitute for actually reading the docs.
  The lookup table tells you which doc; the doc
  tells you the answer.
- Not the place to record methodology updates.
  Edits go to the source documents.

## Open questions

1. Three of the six docs are still drafts
   (`sessions-and-skill-compounding`,
   `ai-native-for-business`, `team-culture-as-sessions`).
   When (and how) does each promote to
   `methodology/`? The first one is gated on Matt's
   post-v1 learning-loop session; the third is gated
   on multi-person session evidence; the second is
   gated on business-user adoption signals nobody is
   currently tracking. **This index page should
   re-check those gates quarterly.**
2. Is the canonical-sequence framing right
   ("establish, compose, retrofit, extend, critique,
   name the gap")? It's accurate as description but
   may be too literary for the operator audience.
3. The L-domain inventory rows are 5; this guide
   maps them all but doesn't expand them. Is the
   right next move to *retire* the L rows in favor
   of "see methodology index," to keep the
   inventory single-source-of-truth crisp?

## Status

DRAFT. The reading-list form has a lower validation
bar than the full and reduced-form guides — promotion
criterion is: one new operator follows the onboarding
order, runs a first session cleanly, and reports
back whether the order + time estimate (~90 min
reading + first session) holds. Reading-list guides
fail when the reading order is wrong for the actual
on-ramp, not when the content is wrong.
