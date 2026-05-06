# Experiment 002: movies (Helium replication)

> Second experiment. Intentional from day one. Framed before scaffolding.
>
> **This is the helium-replication experiment**, renamed to `movies` because the chosen MVP surface is a movies/actors catalog API. The Helium bar (service · structured logs · Prometheus · Grafana · load-test) is the acceptance bar.

## Project

**movies** — a small, self-contained, Kubernetes-native read-only HTTP API serving a movies/actors catalog from local JSON files, with first-class observability (Prometheus + Grafana) and a fully documented local k3s inner loop.

Spec: [repos/movies/README.md](../../repos/movies/README.md)
Local clone (gitignored): `repos/movies/`

## Engineer Profile

- **Level:** TBD
- **Domain experience:** TBD — fluent in Copilot/Claude Code; not necessarily fluent in the chosen API stack or Prometheus Operator.
- **Reuse base:** none (greenfield). Spec is the only prior artifact.

## Hypothesis

The cLLM result (~130× vs Helium) was partly attributable to a senior FDE with deep domain reuse. **movies** strips both: greenfield, no reuse, language/framework deliberately unspecified by the spec. If the session model + RPI inner loop still produces a Helium-bar artifact (service · structured logs · Prometheus · Grafana · load-test client) in ~10–12 sessions, the multiplier is methodology-driven, not seniority-driven.

## Methodology

- **Outer loop:** [context-first sessions](../../methodology/sessions-not-stories.md). One RPI cycle per session. Frame → work → close ritual (green tests · FF-merge · tag · repo memory · next-session starter).
- **Inner loop:** [HVE Core RPI](../../repos/hve-core/docs/rpi/README.md). Research → Plan → Implement → Review, with `/clear` between phases.
- **Cadence target:** one **release-able session** per work block, ending in a tagged dot-release on `repos/movies`.
- **Arc plan:** [session-plan.md](session-plan.md) — **6 framed sessions** anchored on the 90–120 focus-minute cognitive bound (not the 10–12 ceiling), from scaffold to acceptance-criteria pass.
- **Session log:** [session-log.md](session-log.md) — filled in as sessions happen.

## Status

- **Phase:** planning. Spec drafted (v0.3). Arc framed. Implementation has not started.
- **Identity:** this *is* the helium-replication experiment. No sibling artifact will be scaffolded.

## What This Experiment Will Test

1. Whether RPI's "verified truth, not plausible code" property survives a stack the engineer has never used.
2. Whether per-session FF-merge + tag holds up across observability and k8s manifests work (cllm showed it works for Go services).
3. Whether the close-ritual paragraph alone is enough state for a new chat to resume cleanly between sessions when the engineer is *not* the methodology author.
