# Study Guide — Observability (movies-spec §7)

> **DRAFT — NOT FOR PUBLICATION.** Companion to
> [sessions-and-skill-compounding.md](sessions-and-skill-compounding.md).
> Held internal until at least one run uses it end-to-end and we have
> evidence on whether it closes the learning gap without breaking the
> artifact bar.

## Why this exists

[sessions-and-skill-compounding.md](sessions-and-skill-compounding.md)
names a structural gap: in 2020, the operator had to learn the platform
to ship; in 2026, the agent will do the learning for them. Matt-v1
shipped movies 1.0.0 in ~9 focus hours and **did not know that Grafana
dashboards could be edited in the UI.** Substantial Prometheus / Grafana
/ k3s exposure went past him without becoming skill.

This guide is the **proposed remediation for the observability slice
of the spec.** It does three things:

1. Gives the operator a small, finite curriculum bounded by what the
   movies-spec actually requires (§7.1 metrics, §7.2 logs, §7.3
   dashboards, §8.1 ServiceMonitor).
2. Forces hands-on contact with the tools the agent would otherwise
   drive end-to-end — Grafana UI, PromQL, `kubectl`, log shape.
3. Provides a **knowledge check** that becomes part of the per-release
   human + Claude review (see [§ Per-release review](#per-release-review-template)).

The methodology bet: *if the operator can answer the checks on their
own at each tag, the skill compounded; if not, the agent did the work
and the operator skipped the learning beat.*

## How to use this guide

- **Before** the first observability-touching session, scan the module
  list and pick the one that maps to the work coming up. Don't read the
  whole guide front to back — that is reading-not-doing, exactly the
  failure mode this guide exists to prevent.
- **During** the session, when the agent reaches for a tool listed in a
  module, pause and run the lab yourself. The session timer is paused
  for lab time; treat it as a deliberate platform-exposure beat, not
  overhead.
- **At close**, append a one-paragraph "what I learned" to the session's
  RETRO entry. Cite the lab you ran.
- **At each tag**, run the [per-release review](#per-release-review-template)
  with Claude and a human reviewer. The knowledge checks are the test.

## Prerequisite — your cluster exposes services on real ports

Every lab in this guide assumes your local cluster (k3s or k3d)
exposes the following ports via the built-in Traefik LoadBalancer
(klipper-lb on k3s, the `--port ...@loadbalancer` map on k3d):

- `:8080` — `movies-api`
- `:3000` — Grafana
- `:9090` — Prometheus

If you are reaching for `kubectl port-forward`, **stop.** That's a
smell, not a workflow — it taxes every lab with terminal management,
hides Service / Ingress correctness, and teaches a habit that
doesn't transfer to production. See
[study-guide-kustomize.md](study-guide-kustomize.md) Module 7 (and
the forthcoming K8s guide, domain C of
[skills-inventory.md](skills-inventory.md)) for how to wire this
once. From here, labs use `curl http://localhost:<port>` and a
browser tab.

## Module format

Every module has the same four sections. None should take more than
~20 minutes; if a module is taking longer, the module is too big and
should be split.

1. **Concept** — one paragraph. What it is, why the spec requires it.
2. **Example** — minimal, runnable, copy-pasteable. Anchored in the
   movies repo when possible.
3. **Lab** — the operator does this with their hands. The agent may
   answer questions but does not type the commands.
4. **Knowledge check** — 3–5 questions. The operator answers without
   re-reading the module. Used at per-release review.

---

## Module 1 — Prometheus metrics fundamentals (spec §7.1)

### Concept

Prometheus is pull-based: the service exposes `/metrics` in a plain-text
exposition format, and a Prometheus server scrapes it on an interval.
Three metric types cover ~all of what the movies-spec needs:

- **Counter** — monotonically increasing (e.g. `http_requests_total`).
  Reset only on process restart.
- **Histogram** — observations bucketed (e.g. request latency). Lets
  you compute quantiles like p95 server-side via PromQL.
- **Gauge** — instantaneous value that can go up or down (e.g. in-flight
  requests, memory bytes).

The spec is deliberately silent on metric names — implementers pick
idiomatic ones for their client library — but the *shape* is fixed:
counters for events, histograms for latency, gauges for current state.

### Example

```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",route="/api/movies",code="200"} 4827

# HELP http_request_duration_seconds Request latency
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{route="/api/movies",le="0.001"} 4101
http_request_duration_seconds_bucket{route="/api/movies",le="0.005"} 4810
http_request_duration_seconds_bucket{route="/api/movies",le="+Inf"} 4827
http_request_duration_seconds_count{route="/api/movies"} 4827
http_request_duration_seconds_sum{route="/api/movies"} 1.84
```

### Lab

In the running movies cluster:

```bash
curl -s http://localhost:8080/metrics | head -40
curl -s http://localhost:8080/metrics | grep -E '^http_request' | head -20
```

Then, **without re-reading the example**:

1. Identify one counter, one histogram, and one gauge in the output.
2. Find the metric that would tell you total 5xx responses on
   `/api/movies/{id}`.
3. Find the metric that would let you compute p95 latency on
   `/api/actors`.

### Knowledge check

1. Why does the spec use a histogram for latency and not a gauge?
2. What happens to a counter on pod restart? How does PromQL handle that?
3. If `http_requests_total{code="500"}` is missing from the output,
   does that mean zero 500s, or that no 500 has ever occurred? Why
   does the difference matter?
4. The spec splits metrics onto port 9090 as an option (§8.1). What's
   the operational reason you might want that?

---

## Module 2 — Structured JSON logging (spec §7.2)

### Concept

The spec requires one JSON object per line on stdout, with levels
configurable via `MOVIES_LOG_LEVEL`. JSON-on-stdout is the 12-factor
contract: the platform (k8s, Loki, Datadog, whatever) handles
collection. The operator's job is to make sure every log line is
parseable and carries the fields a future on-caller needs.

Required field discipline:

- A timestamp the platform can parse.
- A level field matching `debug|info|warn|error`.
- A message.
- Request-scoped context (route, status, duration) on request logs.
- **No PII, no request bodies.** Query strings are OK (service is GET-only).

### Example

```json
{"ts":"2026-05-05T10:14:22Z","level":"info","msg":"request","route":"/api/movies","status":200,"duration_ms":0.42}
{"ts":"2026-05-05T10:14:22Z","level":"warn","msg":"slow query","route":"/api/movies","duration_ms":47.1}
```

### Lab

```bash
kubectl logs -n movies deploy/movies-api --tail=50
kubectl logs -n movies deploy/movies-api --tail=200 | jq -r 'select(.level=="warn" or .level=="error")'
kubectl logs -n movies deploy/movies-api --tail=200 | jq -r 'select(.route=="/api/movies") | .duration_ms' | sort -n | tail -5
```

> `kubectl logs` is the correct tool here — logs come out of the
> kubelet, not over a service port. The no-port-forward rule applies
> to HTTP traffic; log retrieval is its own path.

Then, change `MOVIES_LOG_LEVEL` from `info` to `debug` via the
deployment env, redeploy, and watch the volume change.

### Knowledge check

1. Why stdout-as-JSON instead of writing to a file?
2. The spec says no request bodies are logged. Why is that safe to
   say for movies but not safe to say for, e.g., a POST-accepting API?
3. If logs aren't valid JSON (one bad `printf` somewhere), what
   downstream tool breaks first, and how would you notice in this
   cluster?
4. What does `jq` do for you here that `grep` cannot?

---

## Module 3 — Grafana dashboards and UI editing (spec §7.3)

> **This is the module Matt-v1 missed entirely.** Don't skip it. The
> agent will offer to regenerate the dashboard JSON; resist, and edit
> the panel in the UI first, then export.

### Concept

Grafana dashboards in this repo are provisioned from a ConfigMap (the
dashboard JSON is checked in). At runtime, you can **edit panels in
the UI**, see the change live, and then export the updated JSON back
to the ConfigMap. The provisioning-from-disk flow does not prevent
UI editing; it just means UI changes are lost on pod restart unless
you export and commit.

The workflow is:

1. Open Grafana in the browser.
2. Edit a panel: change the query, change the visualization, change
   the legend.
3. Verify the change shows live data.
4. Use **Dashboard settings → JSON Model** to copy the updated JSON.
5. Paste into the ConfigMap source, redeploy, confirm the panel
   survives a pod restart.

### Example

A panel showing p95 latency by route, with this query:

```promql
histogram_quantile(0.95,
  sum by (route, le) (
    rate(http_request_duration_seconds_bucket[1m])
  )
)
```

### Lab

Open Grafana in your browser: <http://localhost:3000>.

Then, with no agent help:

1. Open the movies dashboard.
2. Add a new panel showing **error rate per route** (5xx per second).
3. Set the legend to `{{route}}`.
4. Save the dashboard *in the UI*.
5. Export the JSON.
6. Replace the dashboard ConfigMap source in the repo.
7. Redeploy and confirm the new panel survives `kubectl delete pod`.

### Knowledge check

1. What is the difference between "save in UI" and "export JSON"?
2. If you redeploy Grafana without exporting, what happens to your
   panel? Why?
3. The dashboard is provisioned from a ConfigMap. Why is that
   preferred over manually creating panels at install time?
4. When would you choose a **time-series** vs a **stat** vs a **table**
   visualization for the same underlying query?

---

## Module 4 — PromQL essentials

### Concept

PromQL is the query language Grafana uses to ask Prometheus questions.
Four building blocks cover most movies-spec needs:

- **Instant vector** — current value of a metric: `http_requests_total`.
- **Range vector** — values over a window: `http_requests_total[1m]`.
- **`rate()`** — per-second average rate of a counter over a window:
  `rate(http_requests_total[1m])`.
- **`histogram_quantile()`** — compute a quantile from histogram buckets:
  `histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[1m])))`.

`sum by (label)` is how you aggregate across pods, routes, status codes.

### Example

```promql
# requests per second, by route
sum by (route) (rate(http_requests_total[1m]))

# error ratio: 5xx as a fraction of all requests
sum(rate(http_requests_total{code=~"5.."}[1m]))
  /
sum(rate(http_requests_total[1m]))

# p95 latency by route
histogram_quantile(0.95,
  sum by (route, le) (rate(http_request_duration_seconds_bucket[1m])))
```

### Lab

Open the Prometheus UI: <http://localhost:9090>.

In the expression browser:

1. Write a query for "requests per second on `/api/movies` for the
   last 5 minutes, by status code."
2. Write a query for "p99 latency on `/api/actors/{id}`."
3. Write a query that returns the **top 3 slowest routes** by p95.

### Knowledge check

1. Why `rate(...[1m])` and not just `http_requests_total`?
2. What does `sum by (le)` mean and why is it required inside
   `histogram_quantile`?
3. If the scrape interval is 15s, why is `rate(...[5s])` a bad idea?
4. What's the difference between `rate` and `irate`? When would you
   pick `irate` for a Grafana panel?

---

## Module 5 — Prometheus Operator and ServiceMonitor (spec §8.1)

### Concept

The spec mandates the Prometheus Operator and a `ServiceMonitor`
resource for scrape configuration. The legacy `prometheus.io/scrape`
pod/service annotation is **explicitly forbidden** by the spec.

A `ServiceMonitor` is a Kubernetes Custom Resource that the
Prometheus Operator watches; it tells Prometheus *which services to
scrape, on which port, at which path, with what labels*. This is the
declarative, GitOps-friendly version of "annotate your pod and hope."

### Example

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: movies-api
  namespace: movies
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: movies-api
  endpoints:
    - port: http
      path: /metrics
      interval: 15s
```

### Lab

```bash
kubectl get servicemonitor -A
kubectl describe servicemonitor -n movies movies-api
kubectl get prometheus -n monitoring -o yaml | grep -A4 serviceMonitorSelector
```

Then:

1. Find where the operator decides this `ServiceMonitor` belongs to
   this Prometheus instance. (Hint: label selector.)
2. Break it: change the `release` label, reapply, and watch the
   target disappear from the Prometheus **Status → Targets** page.
3. Fix it.

### Knowledge check

1. Why does the spec forbid `prometheus.io/scrape` annotations?
2. What's the relationship between `ServiceMonitor.spec.selector` and
   the `Service` it watches?
3. If a `ServiceMonitor` exists but the target never appears in
   Prometheus, where do you look first?
4. The `ServiceMonitor` has its own `interval`. What happens if it
   conflicts with the Prometheus-level default?

---

## Module 6 — Probes, /version, and tying observability back to the dev loop (spec §8.1, §12)

### Concept

Observability is not just metrics and logs; it's the closed loop the
dev process in §12 step 7 depends on. `livenessProbe` and
`readinessProbe` are signals to the platform; `/version` is the signal
to the human that the deploy actually landed. The Grafana dashboard
is the signal that the *new* version is behaving.

The §12 inner loop is the observability test: bump → build → deploy →
**verify /version** → run validation → **inspect Grafana**. If any one
of those signals is missing, the loop is broken even if the code is
fine.

### Example

```bash
kubectl rollout status deploy/movies-api -n movies
curl -s http://localhost:8080/version
curl -s http://localhost:8080/healthz; echo
curl -s http://localhost:8080/readyz; echo
```

### Lab

Run the §12 inner loop end-to-end:

1. Bump the version (patch bump is fine).
2. Build, deploy.
3. Confirm `/version` returns the new semver.
4. Run the validation suite.
5. **Open Grafana and confirm you can see the validation run as a
   spike on the requests-per-second panel.** This is the bit Matt-v1
   never saw with his own eyes.

### Knowledge check

1. Why does `/healthz` respond 200 before the dataset has finished
   loading, but `/readyz` does not?
2. If `/version` returns the *old* version after a deploy, what are
   the two most likely causes?
3. The validation suite run should be *visible* on the dashboard. If
   it isn't, what's broken — the validation, the metrics, the
   scrape, or the panel? How do you isolate which?
4. What's the smallest change you could make to the dashboard to make
   "the validation run just happened" obvious at a glance?

---

## Per-release review template

> **Proposed addition to the close ritual at each tag.** Runs with
> Claude in the room and (when available) a second human reviewer.
> The artifact is a short markdown block appended to RETRO.md for
> that release.

For each module touched in this release:

1. **Concept check (oral, 60 sec):** operator answers one knowledge-check
   question from each touched module, without re-reading the guide. Claude
   acts as the examiner; the human reviewer scores honest / hand-wavy /
   missed.
2. **Hands-on check (5 min):** operator drives one lab step live —
   typically the one most relevant to the work in this release. Agent
   may answer "what does this command do" questions but does not type.
3. **Coaching prompt:** operator asks Claude *"What did I miss in this
   release that you handled silently? Where could I have learned more
   if I'd asked?"* Answer is recorded verbatim in the RETRO block.
4. **Gap log:** anything the operator could not answer, or could not
   do hands-on, gets logged as a gap. Gaps drive the next release's
   pre-session module pick.

**RETRO block format:**

```markdown
### Learning check — <tag>

- Modules touched: <list>
- Concept checks: <honest | hand-wavy | missed> per module
- Hands-on check: <module> — <pass | partial | fail>
- Claude coaching prompt response: <verbatim>
- Gaps logged for next release: <list>
```

A release is **not closed** until this block is appended. This is the
mechanism that promotes "I learned" from a private feeling to a
falsifiable, reviewable artifact — the same move sessions made for
"I shipped."

## What this guide is and is not

- **Is:** a small, finite, spec-anchored curriculum with a built-in
  test that runs at every release.
- **Is not:** a Prometheus / Grafana / Kubernetes textbook. The
  modules deliberately end at what the movies-spec requires. If the
  next experiment requires more (alerting rules, recording rules,
  multi-cluster federation), it gets its own guide.
- **Is not:** a pass/fail gate. A failed knowledge check does not block
  a tag; it logs a gap and changes what gets picked up next session.

## Open questions

- Is six modules the right size, or should each release's review only
  touch modules **changed in that release**? (Current answer: only
  touched modules — keeps the per-release cost bounded.)
- Should the knowledge check be written-and-graded (more rigorous) or
  oral-with-Claude (lower friction)? Current proposal: oral by default,
  written for the final release of a campaign.
- Does the guide format itself generalize? If yes, this becomes a
  template (`study-guide-template.md`) and observability is the first
  instance.
- Where does the gap log go? Per-release RETRO is the proposal; a
  cross-release "skills ledger" might be a better artifact long-term
  but adds a second file to maintain.

## Status

- Not yet run end-to-end with any operator.
- Not yet validated against the [sessions-and-skill-compounding](sessions-and-skill-compounding.md)
  hypothesis.
- Unlocks for promotion to `methodology/` once at least one full run
  (Matt-v2 or equivalent) has used it and reported honestly on
  whether it closed the learning gap without inflating focus time
  beyond the artifact's value.
