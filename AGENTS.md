# Repo memory for AI assistants

> Durable notes for any AI agent (Copilot, Claude Code, Codex, Aider, etc.)
> working in this repo. Read this first. Update it at the close of every
> session that changes the facts. Keep it short.

## What this repo is

- **`context-first/core`** — the methodology repo for **context-first /
  sessions-not-stories**, a planning methodology for AI-native engineering
  teams. Public: `github.com/context-first/core`. Domain: `context-first.ai`.
- The repo holds three things: (1) the methodology, (2) the experiments
  testing it, (3) a public landing page (`index.html`).
- Outer-loop methodology (sessions, arcs, ceremonies, repo memory) composes
  with inner-loop **RPI** (Research → Plan → Implement → Review) from
  Microsoft's HVE Core. RPI runs *inside* a session.
- This is **bartr's personal work, MIT licensed.** Not Microsoft IP.

## Layout

```
README.md                       methodology overview
contributing.md                 how to submit experiments
index.html                      public landing page (context-first.ai)
methodology/
  sessions-not-stories.md       the session primitive
  ceremonies-as-sessions.md     standups/retros/planning as sessions
  sessions-and-rpi.md           how sessions compose with RPI
  drafts/
    ai-native-for-business.md   PM/stakeholder angle (draft)
experiments/
  session-log-template.md       template for new experiments
  cllm/                         Experiment 001 — accidental, with reuse
  fit-check/                    Experiment 002 — methodology-shaping
  movies-bartr/                 Experiment 003 — intentional, no reuse
repos/                          gitignored working area (see below)
```

## The three experiments (current state)

| # | Folder | What it proves | Status |
|---|---|---|---|
| 001 | `experiments/cllm/` | Accidental discovery of the methodology. Partner FDE, ~130× vs Helium baseline, 125 commits / 12 days. Public: `github.com/bartr/cllm`. | shipped |
| 002 | `experiments/fit-check/` | Methodology evolves under stress — a chat-only dry-run produced the **fit check** beat (added between Plan and Implement). Branch `fit-test`. | shipped |
| 003 | `experiments/movies-bartr/` | Intentional replication of Helium MVP scope. ~5 focus hours, ~830× ratio, p95 50–500× under spec, honest RETRO. Also serves as **participant 1** of a reusable harness. | shipped |

A fourth experiment (different operator running the same movies spec) is
in flight. Tests the seniority confound that the first three don't strip.

## `repos/` — gitignored sibling clones

`repos/` is a gitignored working area for cross-repo sessions. **It may
contain zero, one, or more** of the clones below at any given time —
presence is not guaranteed. Always `ls repos/` (or equivalent) at the start
of a cross-repo session to see what's actually checked out. Tools that
respect `.gitignore` (e.g. ripgrep, GitHub Copilot's `grep_search`) will
silently return empty results inside `repos/`; pass an "include ignored"
flag when searching there.

Known clones that may appear under `repos/`:

| Path | Public URL | Role | IP |
|---|---|---|---|
| `repos/cllm/` | `github.com/bartr/cllm` | cLLM experiment source (Go, K3s/Flux, MCP server). | bartr, MIT |
| `repos/hve-core/` | `github.com/microsoft/hve-core` | Reference: HVE Core agents/instructions/skills + RPI. | **Microsoft, MIT** |
| `repos/movies-template/` | `github.com/bartr/movies` | The Movies experiment harness / template. Other FDEs `Use this template`. May also appear at `~/movies/`. | bartr, MIT |
| `repos/movies-bartr/` | `github.com/bartr/movies-bartr` | Participant 1's run through the harness. May also appear at `~/bartr-movies/`. | bartr, MIT |

**Don't conflate `movies-template` and `movies-bartr`.** The template has
no filled-in RETRO, no session log, no `.copilot-tracking/` artifacts —
those are populated by the participant. Edits to the harness go to
`movies-template`; edits to the run go to `movies-bartr`.

## IP / licensing

- All bartr-owned IP is **MIT, personally owned by bartr** (NOT Microsoft):
  this repo, `bartr/cllm`, `bartr/movies`, `bartr/movies-bartr`.
- **HVE Core (incl. RPI) is Microsoft, MIT.** Always credit Microsoft when
  referencing RPI or HVE Core. Safe to use, link, compose with — not safe
  to claim as bartr's.

## Cross-repo session protocol

At the start of any session that may span repos, **expect the user to name
the target repo in the framing message.** Example:

> *"This is a session on the `cllm` repo, using the methodology from
>  `core`."*

That single sentence disambiguates whether edits land in `core/` or in a
`repos/<name>/` clone. If it's missing, ask before editing.

Each `repos/<name>/` is an independent git repo with its own history,
branches, conventions, and release process. Don't assume `core/`
conventions apply downstream (and vice versa).

## Conventions in this repo (`core`)

- **Sessions + RPI.** Plan in sessions; one RPI cycle per session; close
  ritual (green tests · FF-merge · tag · repo memory · next-session
  starter). The `methodology/` docs are the canonical reference.
- **Merge strategy: `gh pr merge --rebase --delete-branch`** (replays
  commits, FFs `main`, retains full per-commit history). Fall back to
  `--merge` only when rebase can't succeed cleanly. **Never `--squash`** —
  destroys per-commit history needed for storytelling and forensics.
- **Always ask before merging future PRs** until the user signals
  otherwise. PRs are required even on solo-dev repos like this one;
  self-approval is OK. The PR is the **repo evidence**, not a gate.
- **Versions are per-repo, not synced.** `core/` tags advance independently
  of `repos/cllm/`, `repos/movies-template/`, etc. Don't propose unifying
  them.
- **Don't create markdown files to document changes** unless explicitly
  asked. Git history + the session log are the audit trail.
- **Don't add docstrings, comments, or type annotations** to code or docs
  you didn't change.

## Methodology one-pager

- **Session** — the unit of attention. ~60–120 focus minutes. Bounded by
  cognitive context, not calendar. Always ends with a coherent artifact
  (tests + tag + repo memory).
- **Frame** (before) — three questions, two minutes: what does done look
  like, what's out of scope, what would make this a failure?
- **Fit check** (between Plan and Implement) — does this plan fit in the
  90–120 min bound? If not, what's the smallest cut?
- **Close ritual** (after) — green tests · FF-merge · tag · update repo
  memory · write the next-session starter. Sessions that close cleanly
  produce 2× the value of sessions that don't.
- **Arcs / campaigns** sit above sessions when work spans many of them.
- See [methodology/sessions-not-stories.md](methodology/sessions-not-stories.md)
  and [methodology/sessions-and-rpi.md](methodology/sessions-and-rpi.md).

## Movies spec — important context for any work in `repos/movies-template/`

The spec is **deliberately under-specified.** §1.1 of the spec explains why.
The 2020 Helium baseline started with no schema, no metric names, no
pre-negotiated query semantics — figuring all that out *was* the work.
Specifying it would measure typing speed, not methodology.

Surfaces that are intentionally open (do **not** "fix" them):

- Data schema (read the JSON files)
- Query semantics for `/api/movies` (`q`, `genre`, `year`, etc.)
- Prometheus metric names, labels, histogram buckets
- Grafana dashboard panels and queries
- Inner-loop ergonomics (scripts vs make vs just vs task)
- **The HTTP replay / load-test tool** — the spec mandates building one
  in-repo. **Off-the-shelf tools (k6, Locust, Vegeta, hey, wrk, JMeter,
  Gatling) are explicitly out of scope.** This is part of the experiment.

What the spec *does* fix: public contract surface (§6 paths/status codes),
deployment shape (§4, §8), performance bar (§10.4), acceptance checklist
(§14). That's the comparability surface across participants.

## Don't

- Don't add scope to the methodology that hasn't been earned by a real
  session. Rules earn their keep by surviving sessions, not by being
  designed.
- Don't fabricate URLs or facts. If you don't know, ask or check.
- Don't edit `repos/<name>/` clones without explicit framing that names
  the target repo.
- Don't propose unifying versions/tags across repos.
- Don't `--squash` merges anywhere.
- Don't claim RPI or HVE Core as bartr IP. Microsoft, MIT.
