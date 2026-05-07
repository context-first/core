# movies — Session Log

> One entry per session. Frame before, ritual after. The log itself is the experiment.
>
> Arc plan: [session-plan.md](session-plan.md) · Spec: [repos/movies/README.md](../../repos/movies/README.md)

---

## Session 1 — [date] — Service end-to-end on localhost

**Frame**
- Goal: one cognitive arc — "the API works against the JSON files on my laptop." Stack picked with RPI Research evidence; data layer loads all three JSON files into immutable in-memory collections; all §6 GET endpoints implemented with §6 validation rules; integration tests cover happy + negative; ≥80% coverage on data + HTTP layers.
- Stretch (last item): `/swagger` UI + OpenAPI doc + `/` redirect. Attempt only after core is green. Spillover rule: push through 120 min if attention holds, or defer to Session 2 — never merge half-done.
- Out of scope: `/healthz`, `/readyz`, `/version`, `/metrics`, structured logs, container, k8s manifests, Prometheus, Grafana.
- Failure condition: no written stack rationale; schema invented rather than inferred; any §6 validation rule missing a negative test; coverage < 80%; `/swagger` half-shipped.

**RPI cycle**
- Research artifact: `repos/movies/.copilot-tracking/YYYY-MM-DD-service-mvp-research.md`
- Plan artifact: `repos/movies/.copilot-tracking/YYYY-MM-DD-service-mvp-plan.md`
- Changes log: `repos/movies/.copilot-tracking/YYYY-MM-DD-service-mvp-changes.md`
- Review artifact: `repos/movies/.copilot-tracking/YYYY-MM-DD-service-mvp-review.md`

**During**
- Drift moments:

**Close**
- [ ] Tests green
- [ ] FF-merge + tag (`0.1.0`)
- [ ] Repo memory updated
- [ ] Next session starter:

**One-paragraph summary**


**Health signal**
- Framing quality (1–5):
- Drift (yes/no):
- Close complete (yes/no):

---

<!--
Copy the block above for each subsequent session.
Sessions 2–6 are framed in session-plan.md; transcribe each frame here when you start.
-->
