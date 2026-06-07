# Team Culture as Sessions — Working Draft

> **DRAFT — NOT FOR PUBLICATION.** Sketch of a methodology gap surfaced
> by Matt's movies-spec v1 run. Held until context-first.ai has actual
> multi-person session evidence to write from.

## The thesis

**Sessions are an individual primitive. Team culture is multi-person.**

Everything currently in [sessions-not-stories.md](../sessions-not-stories.md)
and [ceremonies-as-sessions.md](../ceremonies-as-sessions.md) is shaped
by a single engineer's attention window. That is the right scope for the
*work-shipping* problem the methodology was designed to solve. It is
silent on the *team-formation* problem, because team formation does not
happen inside a session — it happens in the gaps **between** sessions,
and across **sessions other people ran**.

## The historical evidence

The Helium MVP (2020) was the first project for a newly assembled team.
Over the course of that project the team defined:

- What "good" looked like — the quality bar
- What was acceptable behavior and what was not
- How code reviews were conducted, with what cadence, against what
  rubric
- What engineering fundamentals meant for the team and how they cascaded
  to the broader org
- How disagreements were resolved
- What got pushed back on, by whom, and how

Many of those norms are **still in use a decade later**, including by
bartr. They were the most durable artifact Helium produced — more
durable than the service itself.

That is what culture *is*: the shared, lived-in agreement about how a
group of engineers works together. It outlasts the project. It outlasts
many of the people who formed it.

## Why solo experiments cannot produce this

By construction, a solo experiment has:

- No second engineer to disagree with
- No code review except self-review (or AI-assisted review)
- No quality-bar calibration against a peer
- No "this is not how we do it here" moment
- No need to articulate norms, because the operator already holds them
  privately

bartr's movies-bartr run carries Helium-era culture forward implicitly.
Matt's movies v1 run had nowhere to absorb culture from — the agent does
not transmit it, and the spec does not encode it. This is not a
methodology bug; it is a category error. Asking a solo session to
produce team culture is asking the wrong primitive to do the wrong job.

## Where culture actually forms (in AI-native work)

Candidate surfaces, none of which are inside a single session:

- **Cross-session code review** between two humans, where the agent's
  output is the artifact under review and disagreement surfaces norms
- **Shared repo memory** — when two engineers update the same
  `CLAUDE.md` / `AGENTS.md` and have to negotiate a single voice
- **Calibration sessions** — explicit time spent agreeing on the quality
  bar before independent sessions begin
- **Retro-of-retros** — patterns across multiple operators' RETRO files
  surface the team's actual values, often more honestly than a charter
  would
- **Coaching loops** — senior engineers reviewing junior engineers'
  sessions become the cultural transmission channel that pre-AI work
  got from sitting next to each other

The methodology needs a name for at least some of these.

## What the artifact would eventually be

For context-first.ai specifically, the intended artifact is a
**working agreement** — earned, not designed:

- How we work together
- What our quality bar is
- What is and is not acceptable behavior
- How code reviews are conducted in an AI-native loop
- What engineering fundamentals mean for us
- How we coach each other (and the agent)

That document does not exist yet, and **should not be drafted
pre-emptively**. Per the repo's provenance pattern, it gets written
only after multiple multi-person sessions have surfaced the actual
norms. Designing it first would reproduce the Scrum / SAFe failure
mode — rules without earned justification.

## Open questions

- Is there a "culture beat" that fits the session model, or is culture
  fundamentally an *arc-level* concern (multiple sessions, multiple
  people, observed over time)?
- Does [ceremonies-as-sessions.md](../ceremonies-as-sessions.md) need
  a culture-formation ceremony — a session-shaped equivalent of "team
  norms workshop" — or is that exactly the kind of designed-first rule
  the methodology is supposed to avoid?
- What is the minimum number of multi-person sessions before culture
  becomes legible enough to write down? Helium took ~26 weeks. Can it
  be faster in an AI-native cadence, or does culture have a wall-clock
  floor that delivery speed cannot collapse?
- Is the cross-session code review (one engineer reviewing another's
  AI-assisted artifact) the most concentrated culture-formation surface
  the methodology has? It looks like it, but we do not have evidence.
- How does an AI agent participate in culture? It is in every session;
  it transmits no norms; it has no memory of "how we do things here"
  beyond what we put in `CLAUDE.md` / `AGENTS.md`. Repo memory may be
  the *only* mechanism for the agent to inherit culture.

## What this draft is not

- Not a published methodology section
- Not a context-first.ai charter
- Not a prescriptive set of beats to adopt now
- Not a claim that sessions need to absorb team-formation work — they
  may not. The methodology may end up needing a *different* primitive
  above sessions for culture.

## Provisional next step

When context-first.ai has run a small number of multi-person sessions
on the same codebase — Matt + bartr reviewing each other, or two
participants on the harness comparing notes — collect the surfaced
norms in a working file. Re-evaluate this draft against that evidence
before promoting any of it into the main methodology.
