# Opinion Library — Design Sketch

> **DRAFT — NOT FOR PUBLICATION.** Half-page design
> sketch responding to the RFP-response-system analogy
> raised in this thread. Names the schema + state
> machine before any code or layout decisions. The
> question this sketch addresses: *can the
> methodology's load-bearing opinions be extracted
> into an indexed, reviewable corpus so the coach
> bottleneck moves from "every spec" to "every new
> approved opinion"?*

## The analogy

A mature RFP-response system has two halves:

1. **An indexed library of approved answers**, each
   with an owner, a date, a review history, and a
   retire-by signal.
2. **A workflow** that decomposes a new RFP into
   questions, retrieves approved answers where they
   fit, generates candidates where they don't, and
   routes new candidates through an approval queue.

The methodology has the same shape:

1. **Load-bearing opinions** — "every Ingress
   declares exactly one entrypoint," "TSV is
   structured," "don't fight the distro" — currently
   embedded in guide prose, not indexed, not dated,
   not retirable.
2. **New-domain authoring** decomposes a spec into
   modules, *should* retrieve opinions where they
   fit, *does* re-generate them from scratch
   instead. Approval is implicit (the coach reads
   the draft) instead of structural.

If the analogy holds, the artifact to design is the
opinion card and the workflow around it.

## The opinion-card schema (proposed)

One opinion = one card. Markdown with frontmatter.

```yaml
---
id: F-003                 # domain letter + monotonic counter
title: every Ingress declares exactly one entrypoint
domain: F                 # the inventory domain letter
applies_to: [ingress, traefik, k3s]
status: approved          # draft | approved | contested | retired
authored: 2026-06-08
authored_by: bartr
last_reviewed: 2026-06-08
review_due: 2026-12-08    # 6-month default; tighter if contested
replaces: []              # opinion ids retired by this one
replaced_by: null         # set when retired
evidence:
  - repos/movies-bartr/deploy/traefik/base/entrypoints.yaml
  - repos/movies-bartr/deploy/ingress/*/route.yaml
---

## The opinion

Every Ingress (or IngressRoute) declares exactly one
entrypoint. Never rely on the default.

## Why

Two-line rationale. Failure mode when violated.

## How to apply

The audit one-liner. The kubectl + jq snippet.

## How this might be wrong

The honest case against. What evidence would retire
this opinion.
```

Notes on the schema:

- **`id` is durable.** Once issued, never reused.
  Retirement keeps the id; it just flips `status`.
- **`evidence` is a list of paths** into the repo or
  the worked-example repos. An opinion without
  evidence is a draft. This is the structural form
  of "rules earn their keep by surviving sessions."
- **`replaces` / `replaced_by`** make retirement a
  first-class operation with provenance. The
  retirement is the audit trail.
- **`review_due`** is the structural answer to
  "approved answers go stale." Default 6 months;
  shortened automatically if the opinion is
  contested or recently retired-and-replaced.
- **"How this might be wrong"** is non-optional. An
  opinion that doesn't name its falsification path
  cannot be approved.

## The state machine

```
   draft  ──review──▶  approved  ──contest──▶  contested
     ▲                    │                       │
     │                    │                       ├──evidence──▶ approved
     │                    │                       │
     │                    │                       └──evidence──▶ retired
     │                    │
     └────reject──────────┘
                          │
                          └──review_due──▶ contested (auto, if no recent confirm)
```

Five states, five transitions. The whole protocol.

- **draft → approved**: coach review. Defaults to
  rejected if "How this might be wrong" is missing.
- **approved → contested**: any operator flags a
  failure case. No coach sign-off needed.
- **contested → approved**: counter-evidence
  produced. Coach signs off on the reaffirmation.
- **contested → retired**: confirming evidence
  produced. `replaced_by` populated if a successor
  exists.
- **approved → contested (auto)**: `review_due`
  passes with no recent re-confirmation. Forces
  the question without requiring a human to
  notice.

## The workflow (the second half of the analogy)

For a new domain spec, the authoring loop becomes:

1. **Decompose** the spec into module-shaped
   questions (this is what the 12-module template
   already does).
2. **Retrieve** opinions whose `applies_to` matches
   the module. The agent reads them as authoritative
   context before writing module prose.
3. **Identify gaps** — module questions with no
   matching opinion. These become draft opinion
   cards.
4. **Generate** draft cards + module prose. The
   draft cards go to the review queue; the module
   prose cites approved opinions and flags drafts
   inline as "draft, not yet reviewed."
5. **Review** is two queues, not one document
   review: (a) approve/reject draft opinions, (b)
   approve module prose given the opinion set.

The coach bottleneck collapses to queue (a). Queue
(b) is mostly mechanical once (a) is clean.

## What this changes about the curriculum

If this works:

- **The 13 study guides become regenerable.** Their
  load-bearing content is the opinion set; the
  prose is the rendering. Updating an opinion
  triggers a re-render of every guide that cites it.
- **`skills-inventory.md` becomes a view** over the
  opinion library, not a hand-maintained list.
- **Honest gap-naming gets a home.** "Named gap"
  becomes "no approved opinion exists for this
  question yet" — a structural fact, not a prose
  convention.
- **The L-domain reading-list form generalizes.**
  Every domain gets an index page generated from
  its opinions; the long-form guide becomes
  optional, not the default.

## The failure mode to name explicitly

SAS has surely seen what happens when an RFP
library's approval bar drops: the library fills
with plausible-sounding answers nobody actually
defended, the retrieval surface looks healthy, and
the sales org slowly degrades trust in the
artifacts the system produces. The methodology
version is the same — an opinion library full of
LLM-generated drafts that were approved on a busy
day will produce study guides that look right and
cut wrong.

**Review discipline IS the system.** The
automation is the leverage on a disciplined
process; it is not a substitute for the process.
The schema above is built around making the
discipline structural (mandatory "how this might
be wrong," automatic `review_due` re-contesting,
public retirement provenance) rather than relying
on coach attentiveness.

## Open questions (for the design itself)

1. **Where does this live in the repo?**
   `methodology/opinions/{domain}/{id}.md`? Or a
   single flat directory? Flat is simpler; nested
   reads like the inventory.
2. **Who can author a draft?** Any operator who
   ran a session that produced the opinion. Who
   can approve? Coach-only, for now. Does that
   scale? (No — but it is the right starting
   constraint until the schema proves itself.)
3. **How is retrieval implemented for the
   authoring workflow?** Initially: the agent
   reads the opinion library as text context. At
   scale: real embedding-based retrieval. The
   schema should not assume either.
4. **What is the relationship between an opinion
   and a session?** Every approved opinion should
   trace back to one or more sessions that
   produced or confirmed it. Should `evidence`
   include session ids, not just file paths?
5. **Does this retire `sessions-and-skill-compounding.md`'s
   "agent reimplementation" open question?** Partly.
   Structure / format scoring stays "high"; opinion
   retrieval upgrades the load-bearing scoring from
   "low to moderate" to *testable*.

## Status

DRAFT, design sketch only. No code, no schema
migration, no opinion cards extracted yet. The
useful next move is one of:

- **(a)** Pick a single domain (F — Ingress is the
  most opinionated and has the smallest opinion
  set) and hand-extract its opinions as cards, to
  pressure-test the schema.
- **(b)** Run a paper experiment: hand Claude the
  current Ingress study guide and ask it to
  extract opinion cards in this schema. Score the
  extraction. (This is the agent-reimplementation
  open question, in miniature, on a problem with
  ground truth.)
- **(c)** Hold the sketch as-is and revisit after
  Matt-v2, since the scaling problem this
  addresses is downstream of whether the v2
  remediation works at all.

My recommendation, honestly: **(a) on a single
domain.** It is the smallest concrete move that
would tell us whether the schema is right. (b) is
tempting but tests the agent, not the design. (c)
is the safe answer that loses the momentum from
naming this analogy.
