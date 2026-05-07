# movies — Session Plan (Arc)

> One arc. **6 framed sessions**, anchored on the 90–120 focus-minute cognitive bound (not the 10–12 ceiling). One RPI cycle per session. One tagged dot-release per session. Headroom inside the greenfield budget for sessions that split mid-arc.

This plan is the **frame for the arc**, not a Gantt chart. Re-frame mid-arc if reality diverges. If a session fits comfortably under 120 minutes, *don't* split it just to hit a number.

## Operating Rules (apply to every session)

1. **Frame first.** Before starting, write Goal / Out-of-scope / Failure condition into [session-log.md](session-log.md). If you can't, the session isn't ready.
2. **One RPI cycle inside.** Research → `/clear` → Plan → `/clear` → Implement → `/clear` → Review. Artifacts land in `repos/movies/.copilot-tracking/` (per HVE Core convention).
3. **Stay in scope.** Drift goes into the *Parking lot* below, not into this session.
4. **Close the session.** Green tests · FF-merge (`gh pr merge --rebase --delete-branch`) · semver tag · repo memory updated · one-paragraph summary · next-session starter.
5. **One thing on `main` per session.** No half-merged work. PRs are required even solo.
6. **Spec is the contract.** Every change traces to a §-reference in [repos/movies/README.md](../../repos/movies/README.md).
7. **Split on signal, not on plan.** If a session blows past 120 minutes or RPI Research surfaces unknowns the frame didn't anticipate, close it short and re-frame the next one. The 6 below are the *expected* shape; 7–9 actual is fine.
8. **Run the fit check after Plan, before Implement.** Once RPI Plan is written, compare it to the frame: *will this fit in 90–120 min?* If no, name the smallest cut that keeps the session release-able and decide — proceed, cut, or re-frame. Record the decision in the session log. The fit check is the cheapest moment to cut scope; it is also the only moment with enough evidence to do so honestly. See [methodology/sessions-not-stories.md](../../methodology/sessions-not-stories.md) §Making It Intentional.

## Arc (6 sessions)

Each row: Goal · Out-of-scope · Failure condition · Spec §-touchpoints · Exit tag.

### Session 1 — Service end-to-end on localhost
- **Goal:** one cognitive arc — "the API works against the JSON files on my laptop." Stack picked with RPI Research evidence; repo skeleton builds; data layer loads `data/movies.json` · `data/actors.json` · `data/ratings.json` into immutable in-memory collections at startup; all `GET` endpoints in §6 (`/api/movies`, `/api/movies/{id}`, `/api/actors`, `/api/actors/{id}`, `/api/genres`) implemented with §6 validation rules returning 400 on violation; integration tests cover happy + negative paths; ≥80% line coverage on data + HTTP layers.
- **Stretch (last item; explicit cut-line):** `/swagger` UI · `/swagger/v1/swagger.json` OpenAPI 3 doc covering every §6 endpoint · `/` → redirect to `/swagger`. Attempt only after the core goal is green.
- **Spillover rule:** if the stretch item is in flight when the 120-minute mark hits, choose one — *(a)* push through to finish if attention is still sharp, or *(b)* close the session at the core goal and carry `/swagger` into Session 2. Do **not** stop mid-stretch and merge it half-done.
- **Out of scope:** `/healthz`, `/readyz`, `/version`, `/metrics`, structured JSON logs, container, k8s manifests, Prometheus, Grafana.
- **Failure condition:** no written stack rationale; schema invented rather than inferred from the JSON; any §6 validation rule missing a negative test; coverage < 80%; `/swagger` partially shipped (either complete or deferred — never half).
- **Spec touchpoints:** §2, §5.1–§5.5, §6, §10.1, §10.2. *(Stretch adds the swagger rows of §6.)*
- **Exit:** `0.1.0`.

### Session 2 — Operability surface
- **Goal:** `/healthz` (liveness), `/readyz` (200 only after dataset loaded), `/version` (plaintext semver, §6.1 exact body rules), `/metrics` Prometheus exposition via the idiomatic client, JSON-per-line stdout logs at `debug|info|warn|error` controlled by `MOVIES_LOG_LEVEL`, effective config logged once at startup with secrets redacted, `--help` prints all flags+envs+defaults+effective values per §11.
- **Carry-in (only if deferred from Session 1):** `/swagger` UI · `/swagger/v1/swagger.json` · `/` → redirect to `/swagger`. Do this **first** so it ships clean before the operability work begins.
- **Out of scope:** container hardening, k8s manifests, Grafana dashboards, ServiceMonitor.
- **Failure condition:** `/version` body doesn't match §6.1 byte-for-byte; `/readyz` returns 200 before load completes; logs not parseable as one JSON object per line; `/metrics` fails `promtool check`.
- **Spec touchpoints:** §6, §6.1, §7.1, §7.2, §11.
- **Exit:** `0.2.0`.

### Session 3 — Container + Kustomize base on local k3s
- **Goal:** single multi-stage `Dockerfile`, minimal runtime base (distroless or Alpine), runs as uid 1000, read-only root FS, all caps dropped, no privilege escalation, `data/` baked at `/data` via `COPY`. `base/` Kustomization for `movies-api` (Deployment with §8.1 probes/resources/securityContext · Service · NetworkPolicy ingress only from Ingress controller + Prometheus pod). Applies cleanly to a local k3s/k3d; pod is healthy and serves traffic.
- **Out of scope:** Prometheus Operator, Grafana, ServiceMonitor, validation suite, benchmarks.
- **Failure condition:** image runs as root or mutates rootfs; any §8.1 line item missing; NetworkPolicy not verified end-to-end.
- **Spec touchpoints:** §4, §8, §8.1, §9, §13.
- **Exit:** `0.3.0`.

### Session 4 — Prometheus Operator + Grafana + dashboard
- **Goal:** Prometheus Operator installed via Kustomize; `ServiceMonitor` (`movies-servicemonitor.yaml`) scrapes `/metrics` every 15s; Grafana deployed with auto-provisioned `prometheus` datasource and a checked-in dashboard JSON showing live data; anonymous viewer in dev overlay; admin password from a `Secret`.
- **Out of scope:** validation suite, benchmark targets, README polish.
- **Failure condition:** any reliance on `prometheus.io/scrape` annotations; dashboard panels show "No data"; admin password baked into a manifest.
- **Spec touchpoints:** §4, §7.3, §8.1, §13.
- **Exit:** `0.4.0`.

### Session 5 — Inner loop docs + Web Validate suite + benchmarks
- **Goal:** README §9.1 walks a fresh contributor verbatim from clean machine → live dashboard, every step runnable standalone, no external network beyond base images + language toolchain. Web Validate-compatible E2E suite covers every §6 endpoint plus a negative case for every validation rule. Benchmark suite hits §10.4 targets (p95 `/api/movies` < 50 ms, p95 `/api/movies/{id}` < 10 ms, 500 RPS sustained < 1% errors on a 500m-CPU pod).
- **Out of scope:** acceptance-criteria sign-off ceremony, experiment writeup.
- **Failure condition:** any README command differs from what works on a clean k3s/k3d; any §6 endpoint missing E2E coverage; any §10.4 target missed without a documented mitigation.
- **Spec touchpoints:** §9.1, §10.3, §10.4, §12.
- **Exit:** `0.5.0`.

### Session 6 — Acceptance pass · 1.0.0 · writeup
- **Goal:** run §14 acceptance criteria end-to-end on a freshly-wiped local k3s. All eight checkboxes green. Tag `1.0.0`. Write the experiment writeup back into [summary.md](summary.md) (Outcome · Evidence · Surprises · Caveats sections, mirroring [experiments/cllm/summary.md](../cllm/summary.md)) including the honest comparison to Helium 2020 (26 wk · 4 ppl · ~104 person-weeks).
- **Out of scope:** new features, non-spec polish, follow-on arcs.
- **Failure condition:** any §14 checkbox not green; the writeup omits the cllm-style honest caveats.
- **Spec touchpoints:** §14.
- **Exit:** `1.0.0`.

## Post-arc

- Cross-arc retro: compare actual session count vs the 6 framed here. Where did sessions split? Where did they merge? Did any exceed 120 minutes? (That's the signal the frame was wrong, not the budget.)
- Update [methodology/sessions-not-stories.md](../../methodology/sessions-not-stories.md) Open Questions with anything this arc answered — especially session-length norm.
- Helium-replication evidence: this arc *is* the comparison data point against Helium (2020) 26 wk · 4 ppl · ~104 person-weeks.

## Parking lot (drift catcher)

Threads to **not** pull mid-session. Reconsider between sessions only.

- _(empty — fill in as sessions run)_
