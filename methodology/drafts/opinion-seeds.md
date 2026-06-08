# Opinion Seeds — Extracted From Working Conversations

> **DRAFT — NOT FOR PUBLICATION.** A working file of
> opinions surfaced in conversation that should become
> opinion cards once the schema and authoring workflow
> in [opinion-library-design.md](opinion-library-design.md)
> is implemented. These are **seeds**, not cards:
> structured enough that conversion is mechanical,
> light enough that the schema can still change without
> invalidating the content.

## Status conventions used here

- **`seed`** — extracted from a conversation. Not
  reviewed, not approved, not yet a card.
- **`draft`**, **`approved`**, **`contested`**,
  **`retired`** — the four states from the
  opinion-library design.

A seed becomes a draft when (a) it has at least one
evidence path into the repo, (b) it has a written
"how this might be wrong" section, and (c) someone
proposes it for review. The seed-to-draft conversion
is what the first round of card extraction will test.

## How to use this file

When extracting opinion cards (the proposed pressure
test from `opinion-library-design.md`), start here.
Each seed becomes one card; the seed body becomes the
card body, the seed metadata becomes the card
frontmatter, and the "how this might be wrong"
section gets written from scratch (it's deliberately
not pre-filled, because that's the load-bearing
review constraint).

When new opinions surface in conversation, add seeds
here first. They cost ~3 minutes to capture and they
prevent the "I know we discussed this somewhere" rot.

---

## Seeds

### I-001 — TSV is structured

- **status**: seed
- **domain**: I (testing) — also G (observability)
- **applies_to**: webv, instrumentation, log formats, observability tooling
- **origin**: chat refinement of `study-guide-testing.md` Module 5 (2026-06-08)
- **evidence**:
  - `repos/movies-bartr/src/cmd/webv/runner.go` (`writer.emit` TSV format)
  - `learning-library/study-guide-testing.md` Module 5

**The opinion.** TSV with fixed columns is a
structured log format, not a "human-readable text
blob." The structure is the column contract; the
delimiter is just the encoding choice. JSON is *also*
structured. The two serve different consumers:

- **TSV for humans, `awk`, `cut`, `sort`, Excel.**
  One row per event, one column per field, eyeballable
  at the terminal, importable into a spreadsheet without
  a parser.
- **JSON for log platforms** (Loki, Splunk, ELK,
  Datadog). One object per event, nested fields,
  parseable by ingest pipelines without column
  conventions.

The historical webv had a `--json` flag for exactly
this reason: human-friendly TSV by default, platform-friendly
JSON when shipping to a log store.

**Why it matters.** The wrong framing is "structured ==
JSON, unstructured == TSV." Under that framing, teams
default to JSON for everything and pay the human-reading
tax for the lifetime of the system. The right framing
is "pick the encoding to match the consumer."

**How to apply.** Emit dual-format from instrumentation
that has more than one consumer. TSV by default; JSON
behind a flag. Document the column contract as a header
comment in the emitter so the awk pipelines downstream
have a stable reference.

---

### I-002 — TSV opens in Excel as `.xls`

- **status**: seed
- **domain**: I (testing) — also G (observability)
- **applies_to**: webv output, ad-hoc analysis, pivot tables
- **origin**: chat addition to `study-guide-testing.md` Module 5 (2026-06-08)
- **evidence**:
  - `learning-library/study-guide-testing.md` Module 5 lab step 10

**The opinion.** Route TSV output to a `.xls`
extension and Excel opens it directly as a pre-parsed
table, no import wizard. Once open, pivot tables on
status code × latency bucket × endpoint take seconds.

**Why it matters.** "I need to slice this load-test
output by N dimensions" is a recurring need. The default
path (load CSV/TSV into a notebook, write pandas) costs
20+ minutes per question. The Excel path costs ~30
seconds and produces an interactive pivot the operator
already knows how to drive.

**How to apply.** When emitting TSV from a load-test
or benchmark tool, document the `.xls` trick alongside
the awk/jq examples. Treat Excel as a first-class
consumer, not a fallback for people who can't write
pandas.

---

### L-001 — Coaching does not scale by adding more coaching

- **status**: seed
- **domain**: L (methodology)
- **applies_to**: methodology scaling, FDE enablement, partner programs
- **origin**: chat framing of the scaling problem (2026-06-08)
- **evidence**:
  - `methodology/drafts/sessions-and-skill-compounding.md` § "The scaling problem"
  - the 78-minute Helium baseline implied by this conversation's total wall time

**The opinion.** Every proposed remediation for the
v1 learning gap — per-release review, learning retro,
coaching prompts, deliberate platform exposure — is
*more senior attention per session.* That is a real
fix at small N. **It does not scale.** A Partner-level
engineer cannot spend ~78 focused minutes per spec on
coaching across an organization's worth of specs.

The scaling lever is not "more coaching." It is
**compressed coaching** — extract the load-bearing
opinions into a reusable artifact (the opinion library;
the study guides) so the senior bottleneck moves from
"every spec" to "every new approved opinion."

**Why it matters.** Without this opinion stated
explicitly, the methodology overclaims what
per-release review can do. Matt-v2 will test the
remediation at N=1; it cannot prove the remediation
scales. The honest framing is: v2 tests *whether the
remediation works at all*; scaling is a separate
experiment with a separate participant who *self-coaches*
against the compressed artifact with no live senior
attention.

**How to apply.** Whenever a methodology beat
proposes "the coach reviews X," ask: does the coach
review every instance forever, or is there a path to
a compressed artifact that the next operator
self-coaches against? Beats that lack the second
path are honest-but-unscalable and should be marked as
such.

---

### L-002 — Helium-Anne cost is the north star, named honestly

- **status**: seed
- **domain**: L (methodology)
- **applies_to**: skill-compounding claims, comparison framing
- **origin**: chat reframing of the Helium baseline (2026-06-08)
- **evidence**:
  - `methodology/drafts/sessions-and-skill-compounding.md` § "The scaling problem"

**The opinion.** Helium-Anne's skill compounding —
the outcome the methodology aspires to reproduce —
was bought with **six months on a four-person team
under daily senior review with real production
pressure.** That is the cost of the north star. It
has always been the cost.

AI-native tooling and the session methodology do not
reduce that cost. They relocate where the senior
attention goes (from "writing the dashboards with
Anne" to "reviewing the opinions Anne's successor
self-coached against"). They make the same input
cheaper to produce given the senior attention; they
do not eliminate the senior-attention requirement.

**Why it matters.** Claims of "AI-native makes
juniors as productive as seniors" routinely elide the
senior-attention input. Naming the Helium cost
honestly is the discipline that keeps the methodology
falsifiable: if a remediation closes the gap with
*less* senior attention than the Helium baseline,
that is the genuine finding. Anything else is
relocating the cost, not reducing it.

**How to apply.** Whenever a session-methodology claim
about skill compounding is made, name the senior-attention
budget alongside the tool / methodology budget.
Comparisons that omit senior-attention cost are
incomplete by definition.

---

### L-003 — Solved problems are a design source

- **status**: seed
- **domain**: L (methodology) — meta
- **applies_to**: design-thinking, methodology evolution
- **origin**: chat observation across this conversation (2026-06-08)
- **evidence**:
  - Helium-MVP → movies benchmark experiment (this repo)
  - SAS RFP-response system → opinion-library-design.md (this conversation)

**The opinion.** Before designing a new system,
inventory what you have already shipped that solves
this shape. The closest prior solved problem is
usually the highest-leverage design source — better
than first-principles design, better than literature
review, better than asking the agent for ideas.

Two examples from this repo's own history:

- The Helium MVP shape was reused as the
  movies-experiment harness. The harness is what made
  Matt-v1 comparable to bartr-v1 comparable to
  Anne-2020. None of that comparability exists
  without the prior solved problem.
- The SAS RFP-response system shape was reused as
  the source for `opinion-library-design.md`. The
  approved-answers library, the new-answer review
  queue, the retirement protocol — all of it maps
  one-for-one. The design took ~10 minutes to
  recognize once the analogy was named, because the
  hard work was already done in a different domain.

**Why it matters.** AI assistants will happily
generate plausible designs from first principles
that miss the leverage of a previously-solved
analogous problem. The operator's job is to bring
the analogy; the agent's job is to verify the
mapping and extend it. The methodology beat is:
**name the analogous solved problem before
generating design.**

**How to apply.** Add to the Frame step (before
Plan): "What previously-solved problem is this
shaped like?" If the answer is non-trivially "none,"
that is itself a finding — fully novel design is
expensive and should be flagged before being entered.

---

### L-004 — Three guide forms earn their use

- **status**: seed
- **domain**: L (methodology) — learning-library specifically
- **applies_to**: study-guide authoring, curriculum design
- **origin**: chat decisions during curriculum work (2026-06-08)
- **evidence**:
  - `learning-library/study-guide-floor.md` (reduced form)
  - `learning-library/study-guide-methodology.md` (reading-list form)
  - the other 11 study guides (full 12-module form)

**The opinion.** A study guide has three honest forms.
Choose by what the domain actually demands; defaulting
to the full template wastes attention on domains where
the full template is wrong for the content.

- **Full (12-module).** For domains with substantial
  patterned content the operator has not seen, where
  each module earns its place via a worked example and
  a per-release residual. The default for cloud-native
  domains.
- **Reduced (6-module, ~90 min total).** For
  foundational domains (process model, FDs, DNS, HTTP,
  TLS basics) that are a *floor* the operator should
  know but where the curriculum's job is to point at
  canonical reading, not re-teach what TLPI or HPBN
  already covers. Each module is `why-matters /
  minimum-to-know / 10-min lab / canonical reading.`
- **Reading-list / index.** For domains where the
  load-bearing content already exists in canonical form
  elsewhere, and the curriculum's job is to *index it*
  — annotated entries, onboarding order, lookup table
  by question. The right form when re-teaching would
  duplicate authoritative source with less authority.

**Why it matters.** The 12-module template
over-applied produces guides that look comprehensive
and read padded. The reduced and reading-list forms
are the honest answers when the domain doesn't earn
12 modules.

**How to apply.** When scoping a new study guide,
ask: does this domain have substantial unique
patterned content (full), is it a foundational floor
where canonical reading exists (reduced), or does
the load-bearing content already exist elsewhere
(reading-list)? Pick the form before writing.

---

### L-005 — Outputs of the methodology live separately from drafts of the methodology

- **status**: seed
- **domain**: L (methodology) — repo conventions
- **applies_to**: this repo's directory layout
- **origin**: chat decision to rename `methodology/drafts/{inventory,study-guide-*}` to `learning-library/` (2026-06-08)
- **evidence**:
  - this repo's current layout: `methodology/`, `methodology/drafts/`,
    `learning-library/`
  - commit `b14d619`

**The opinion.** A methodology repo has two
categories of content that look similar but mean
different things:

- **Methodology essays and drafts** (in
  `methodology/` and `methodology/drafts/`): documents
  *about how the work is done.*
- **Outputs of the methodology applied to a domain**
  (in `learning-library/`): the skills inventory, the
  study guides — *what the methodology produces when
  pointed at a specific subject area.*

Mixing them muddies what the directories mean: a
reader can't tell at a glance whether a file in
`methodology/drafts/` is a draft of the methodology
itself or a draft of an output. The fix is structural,
not prose: separate directories.

**Why it matters.** As the methodology accretes more
applied artifacts (cloud-native curriculum today;
potentially Go-specific, Python-specific, MLE-specific,
business-user curricula later), the directory layout
either telegraphs the distinction or hides it. Hiding
it forces every reader to re-derive the difference.
Telegraphing it makes the distinction free.

**How to apply.** For any new artifact, before adding
it to a directory, ask: is this *about how the work is
done* (essay) or *what the work produces* (output)?
If output, it goes in or near `learning-library/`,
not `methodology/`.

---

### L-006 — Review discipline IS the system; automation is the leverage

- **status**: seed
- **domain**: L (methodology)
- **applies_to**: opinion library, any review-queue automation
- **origin**: chat framing of the opinion-library failure mode (2026-06-08)
- **evidence**:
  - `methodology/drafts/opinion-library-design.md` § "The failure mode to name explicitly"

**The opinion.** When a review queue exists to
maintain a corpus of trusted artifacts (RFP answers,
opinion cards, study-guide content), the automation
around the queue is leverage on a disciplined
process. **The discipline IS the system.** Drop the
discipline and the corpus degrades fast — it fills
with plausible-sounding LLM drafts that nobody
defended, retrieval looks healthy, consumers slowly
lose trust.

**Why it matters.** SAS has seen this failure mode in
RFP systems. The methodology version is identical:
an opinion library full of approved-but-undefended
drafts will produce study guides that *look right and
cut wrong*. The schema constraints in
`opinion-library-design.md` (mandatory "how this might
be wrong" section, automatic `review_due`
re-contesting, public retirement provenance) are
built around making the discipline structural, not
relying on coach attentiveness on a busy day.

**How to apply.** Any system that proposes
"automation reduces the coach bottleneck" must name
what discipline is being automated and how the
automation enforces (not replaces) the discipline.
Systems that frame automation as a substitute for
discipline are misframed.

---

## What's NOT in here yet

These opinions were referenced in conversation but
predate this thread — they belong in seeds too but
need their own extraction pass against their origin
files:

- **"Every Ingress declares exactly one entrypoint"**
  (domain F) — repo memory rule, anchors
  `study-guide-ingress.md` Module 3.
- **"Don't fight the distro"** (domain F or B) —
  k3s + Traefik specifically; the broader form is
  "use defaults until you have a named reason to
  override."
- **"The dashboard signature IS the SLO"** (domain G)
  — capstone of `study-guide-observability.md`.
- **"Named gap as a curriculum feature"** (domain I,
  L) — the framing that "webv has no `--json` mode
  and no `/metrics` endpoint" is a *named gap*, not a
  *bug to fix before publishing.*

These should be extracted in a second pass once the
schema is finalized and the first six seeds above
have been pressure-tested as cards.

## Status

DRAFT. 6 seeds extracted from this conversation,
4 named for later extraction. None has been
converted to an opinion card yet — that conversion
is what the proposed Ingress-domain pressure-test
in `opinion-library-design.md` would do.
