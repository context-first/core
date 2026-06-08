# Study Guide — Testing and Benchmarks (domain I)

> **DRAFT — NOT FOR PUBLICATION.** Tenth instance of the
> study-guide format. Scoped from domain I of
> [skills-inventory.md](skills-inventory.md). Anchored in
> movies-bartr's in-cluster `webv` deployment
> (`deploy/webv/base/deployment.yaml`) and its two suite
> files (`src/webv/benchmark.yaml`, `src/webv/test.yaml`),
> which are themselves a port of the Helium / Web Validate
> pattern (Microsoft, MIT — see
> https://github.com/microsoft/webvalidate).

## Why this exists

Most teams test up to the point a CI pipeline turns green.
After that, the service ships and the test suite goes
silent. The next time anyone runs a test is when something
breaks in prod and someone writes a postmortem.

This guide teaches a different posture: **the tests never
stop running.** A baseline workload runs continuously
against the live service in-cluster, generating a known
dashboard signature. Releases are evaluated against that
signature. Negative-path suites are run on demand to
confirm the dashboard *responds* to errors the way it
should. The test suite is part of the production system,
not an artifact of a CI run.

That posture is what made the Helium "48-hour smoke test"
work, and it's the posture this guide teaches. The
48-hour part isn't the lesson — the lesson is that **the
baseline runs forever, and you know what its dashboard
looks like.**

**What this guide is anchored in:**

- movies-spec §10 — unit / integration / E2E split,
  ≥ 80% coverage on data + HTTP layers, web-validate
  contract suite as the in-cluster acceptance gate.
- movies-spec §10.4 — 500 RPS sustained, p95 ≤ target,
  0 % 5xx.
- movies-bartr `src/webv/benchmark.yaml` — the happy-path
  baseline that the in-cluster webv deployment runs in a
  loop, 24/7, against `movies-api.movies.svc.cluster.local`.
- movies-bartr `src/webv/test.yaml` — the validation suite
  with `statusCode` / `contentType` / `length` assertions
  on the negative paths.
- movies-bartr `deploy/webv/base/deployment.yaml` — the
  in-cluster runner; reuses the `movies-api` image, runs
  `/webv --loop --threads=2 --sleep=3ms` against the live
  service over its cluster DNS name.
- [study-guide-observability.md](study-guide-observability.md)
  for the metrics + dashboards the baseline's signature
  lives in.
- [study-guide-gitops.md](study-guide-gitops.md) Module 3
  for the rings-and-revert pattern this guide's capstone
  composes with.

## How to use this guide

Same protocol as the other study guides. One
curriculum-level rule specific to this one: **every lab
that adds a test also predicts the dashboard delta the
test should produce, then confirms it.** Tests with no
observable effect on the dashboard are weaker tests; over
time you want every test to map to a metric you can see
move.

## Modules

### Module 1 — The pyramid and what belongs where (I1)

#### Concept

The spec §10 split is the operational answer to "where do
these tests belong":

| Layer | What it covers | Where it runs | Fast? | Trust? |
|---|---|---|---|---|
| **Unit** | One function or one type, no I/O | In-process, in `go test` | ms | High when narrow, low when over-mocked |
| **Integration** | Multiple components together, real dependencies (DB, HTTP) | In-process or test-container, in `go test` | 100s of ms to s | High |
| **Contract / validation** | The service's external API behaves to its spec | Against the live in-cluster service over its real LB | s to minutes | **Highest** — closest to prod |
| **E2E** | Whole-system journey through every layer | Against a deployed environment | minutes | High, expensive |
| **Load / benchmark** | Service holds up at target rate | Against the live in-cluster service | minutes | High for perf, none for correctness |

The trap most teams fall into: writing all five layers
*before they ship anything*, then never running anything
but unit again. The opposite is right — **the contract /
validation suite is the one that runs continuously**, and
unit + integration suites run on every commit. The other
layers run on a cadence (E2E nightly, load on release,
benchmarks on perf-relevant changes).

#### Example

In movies-bartr the layers map cleanly:

```
src/internal/storage/storage_test.go        ← unit (data layer)
src/internal/api/handlers_test.go            ← unit + integration (HTTP)
src/cmd/movies-api/main_test.go              ← integration (full bring-up)
src/webv/test.yaml                           ← contract / validation
src/webv/benchmark.yaml                      ← load + 24/7 baseline
docs/PERFORMANCE.md                          ← benchmark methodology + results
```

`go test ./...` runs the first three on every commit.
`webv` running in the cluster runs the fourth and fifth,
continuously, against the live deployment.

#### Lab

1. In movies-bartr, run `go test ./...` and count tests
   per layer (unit vs integration). Confirm the split
   matches the spec's §10 mandate.
2. List every test layer above and identify which one
   would catch each of:
   - A SQL injection in a handler.
   - A regression that drops `/api/genres` from the
     router.
   - A regression that doubles p95 latency.
   - A regression that returns valid JSON with the wrong
     field name.
3. Notice which regressions only the contract suite would
   catch. That's why §10.3 mandates it.

#### Knowledge check

- Why does the contract suite run in the cluster against
  the real LB instead of in `go test` against an
  in-process server?
- A teammate's PR adds 50 unit tests and zero contract
  tests. The diff changes a JSON field name. CI is
  green. Why is the change still broken?

---

### Module 2 — Coverage as a guide-rail, not a goal (I2)

#### Concept

Coverage is a **floor signal**, not a ceiling. The spec's
≥ 80% on data + HTTP layers means: *if you're below 80%
on those layers, you probably have an untested branch
that matters.* It does **not** mean *80% is enough.*

What coverage tells you:

- ✅ Lines that ran during tests.
- ✅ Branches that ran during tests.

What coverage doesn't tell you:

- ❌ Whether the assertions are right.
- ❌ Whether the test would actually fail if the code
  broke.
- ❌ Whether anyone reads the failure when it does fail.
- ❌ Whether the right *behaviors* are covered (you can
  hit 100% line coverage and never assert that the API
  returns the right shape).

Two patterns turn coverage from a guide-rail into a goal,
both bad:

1. **Coverage gating CI without context.** Forces
   engineers to write low-value tests (`assert(true)`
   wrappers, tests for trivial getters) to clear the
   bar. Reduces signal in the test suite.
2. **Treating ≥ 80% as "we're done."** Coverage of the
   data + HTTP layers is the *minimum*. Behavioral
   coverage (the contract suite) matters more.

The corrective framing: **coverage is what tells you
where you haven't *thought* about testing.** When it
drops below the floor, go look at *what* isn't covered
and decide whether it needs a test or whether the
uncovered code shouldn't exist.

#### Example

```sh
$ go test ./... -coverprofile=cover.out
$ go tool cover -func=cover.out | tail -1
total:                                  (statements)    87.3%

# That's a floor signal. Now look at what isn't covered:
$ go tool cover -func=cover.out | awk '$3 < "80.0%" {print}'
internal/api/handlers.go:152:   handleActorsGenres      0.0%
internal/storage/sqlite.go:88:  vacuum                  0.0%
```

Two findings:

- `handleActorsGenres` at 0%: real gap — that's a public
  endpoint. Add a test.
- `vacuum` at 0%: maintenance function called by ops, not
  by the API. Decide: test it, mark it
  `//nolint:funlen` and document it, or delete it.

The coverage number didn't tell you that. The line-level
breakdown did.

#### Lab

1. Run `go test ./... -coverprofile=cover.out` on
   movies-bartr. Generate the HTML report (`go tool
   cover -html=cover.out -o cover.html`) and look at one
   under-covered file.
2. Find one line of uncovered code that *should* be
   covered (a real bug-risk branch). Write the test.
3. Find one line of uncovered code that *shouldn't* be
   covered (dead code, a debug-only path, a `panic` that
   should never fire). Either delete it or explain why
   it stays.
4. Compare the contract suite (`webv test.yaml`) to the
   line coverage. Which behaviors does it test that line
   coverage can't see?

#### Knowledge check

- Why does the spec specify "data + HTTP layers" rather
  than "≥ 80% overall"?
- Coverage went from 87% to 84% on a PR. Two possible
  causes — what are they, and how do you tell which?
- A handler has 100% line coverage and no assertions on
  the response body. Is it tested?

---

### Module 3 — Contract / validation suites in-cluster (I3)

#### Concept

A contract suite is a **list of HTTP requests + expected
responses, run as a black box against the live
service.** It doesn't know anything about the service's
internals. It only knows:

- "I'm going to call this URL with this method."
- "I expect this status code."
- "I expect this content-type."
- "I expect the body to satisfy these constraints."

If the suite is happy, the service's external contract is
intact. That's a stronger claim than any unit test makes,
because it's tested against the same artifact, on the same
infrastructure, hitting the same network path, as
production traffic.

The minimum viable validation feature set:

| Validation | What it catches |
|---|---|
| `statusCode` | Wrong response code (400 when 200 expected, or 200 when 400 expected — both are bugs) |
| `contentType` | Wrong response shape (returning HTML when JSON expected) |
| `length` (exact or range) | Truncated or bloated response |
| JSON-shape assertion (field exists, array count, value match) | The "valid JSON, wrong fields" failure mode |

The fourth one is where contract suites *earn their
keep* — coverage tools and unit tests can both miss "the
JSON parses but the schema is wrong." Only a black-box
suite hitting the real endpoint can catch it without
duplicating the schema in two places.

#### Example

From `src/webv/test.yaml` in movies-bartr — the negative
path for input validation:

```yaml
- path: /api/actors?q=123456789012345678901
  validation:
    statusCode: 400
    contentType: application/problem+json

- path: /api/actors?q=a
  validation:
    statusCode: 400
    contentType: application/problem+json
```

Two requests, each with two assertions. The first sends
an over-long query; the second sends an under-length
query. Both should return a 400 with the RFC 7807
`application/problem+json` content type. If the service
silently accepts the input and returns 200, this suite
fails. If the service returns 400 with `text/plain`, this
suite fails. The unit tests covering input validation
might pass in both cases — the contract suite catches
both.

And the happy path:

```yaml
- path: /robots.txt
  validation:
    contentType: text/plain
    length: 48
```

The exact `length: 48` is a strong assertion. If anyone
edits `robots.txt`, this fails immediately — which is
exactly the signal you want.

#### Lab

1. In movies-bartr, run the suite locally against the
   in-cluster service:
   ```sh
   $ kubectl exec -n movies deploy/webv -- /webv \
       --url=http://movies-api.movies.svc.cluster.local:8080 \
       --files=/webv-suites/test.yaml
   ```
2. Break the service deliberately: edit a handler to
   return `text/plain` instead of `application/json` for
   `/api/movies`. Re-deploy. Re-run the suite. Confirm it
   fails on the `contentType` assertion.
3. Add a JSON-shape assertion to `test.yaml` for
   `/api/movies` — confirm it returns a JSON array with
   at least one element. Confirm the suite still passes.
4. Make the assertion stricter — confirm the first
   element has a `title` field. Confirm. Then break
   it: rename `title` to `name` in the handler.
   Confirm the suite fails.

#### Knowledge check

- Why is "the contract suite runs against the in-cluster
  service over the real LB" stronger than "the contract
  suite runs in `go test` against `httptest.Server`"?
- A unit test asserts `resp.Title == "Star Wars"`. A
  contract test asserts `jq -e '.[0].title' < resp`.
  Both pass. The handler renames the field to `name`.
  Which test catches it, and why?
- The contract suite passes locally but fails in dev
  cluster. Three causes — what are they?

---

### Module 4 — The webv evolution: CLI → validation → in-cluster baseline

> **The Helium pattern, in three steps.** Curriculum
> centerpiece for domain I. Originates in Helium /
> Web Validate (Microsoft, MIT —
> https://github.com/microsoft/webvalidate); the
> movies-bartr port reproduces the pattern at curriculum
> scale.

#### Concept

The Helium team didn't start with "let's run a contract
suite in-cluster forever." They got there in three
moves, each a response to a real failure mode.

**Step 1 — CLI tool, ad-hoc from a terminal.**

Web Validate (`webv`) started as a binary you ran from
your laptop against a deployed service:

```sh
$ webv --url=https://api.example.com --files=test.yaml
```

It read a YAML list of requests, fired them, printed
results. Useful for "did the release land," not much
more. Same shape as `curl + bash`, but typed and
shareable.

**Step 2 — Add validation features, one at a time.**

Each validation feature came from a real outage. The
list grew like this:

- Outage: API returned 200 with an empty body. → Add
  `length` assertion.
- Outage: API returned 200 with HTML (a CDN error
  page).  → Add `contentType` assertion.
- Outage: API returned 200 with valid JSON, wrong
  field name. → Add JSON-shape assertion.
- Outage: API returned 200 with the right JSON, but
  the array had 0 elements instead of 10. → Add array
  count assertion.
- Outage: API returned 200 with the right JSON, but
  the values were stale. → Add value-presence
  assertion (e.g. "today's date appears in the
  response").

By the time the validation feature list stabilized, the
suite could express most of the "looks healthy from the
outside" checks a human would do manually.

**Step 3 — Run the suite continuously, in the cluster,
forever.**

Once the suite could express the checks reliably, the
next move was structural: stop running it ad-hoc from a
laptop, start running it as a Deployment inside the
cluster, in a loop, against the live service over its
cluster DNS name.

That single move changed what testing *is*:

- Before: tests run during CI, then go silent.
- After: tests run 24/7 against the production service,
  generating a continuous metrics signal.

The metric the test generates ("requests/sec," "error
rate," "p95 latency") *is* the baseline. The dashboard
showing that metric *is* the test result. You don't
read a green check — you read a flat green line on a
chart.

That's the Helium pattern. movies-bartr reproduces it
end-to-end — with two named gaps that become Module 5:

| Helium piece | movies-bartr equivalent |
|---|---|
| `webv` CLI | `src/cmd/webv/main.go` (same shape) |
| Suite YAML | `src/webv/benchmark.yaml` + `src/webv/test.yaml` |
| In-cluster Deployment | `deploy/webv/base/deployment.yaml` running `/webv --loop --threads=2 --sleep=3ms` |
| Baseline dashboard | Grafana panel showing `movies-api` RPS, p95, error rate |
| Negative-path suite | `test.yaml` with the 4xx assertions |
| Structured logs — **TSV for human eyeballing** | ✅ TSV writer with fixed columns: `ts \t status \t code \t dur \t method \t path \t bytes \t content-type \t errs` |
| **JSON log mode for log platforms** | **Not yet — `--json` flag missing. See Module 5.** |
| **Client-side Prometheus histograms** | **Not yet — no `/metrics` endpoint, no per-request observation. See Module 5.** |

The last two rows are the *original* reasons `webv` existed
at Helium beyond "a CLI test runner": every request emitted
a Prometheus histogram, and the log writer supported
**both** a tab-delimited mode (easy for a human to read on a
small test) and a JSON mode (the right shape for a log
platform to index at scale). The movies-bartr port has the
TSV mode but not yet the JSON mode or the metrics endpoint,
which makes those the natural next increment — Module 5
walks through adding them.

#### Example

The movies-bartr `webv` Deployment runs continuously
against `movies-api.movies.svc.cluster.local:8080`:

```yaml
containers:
  - name: webv
    image: movies-api:1.0.0          # same image, two binaries
    command: ["/webv"]
    args:
      - --url=http://movies-api.movies.svc.cluster.local:8080
      - --files=/webv-suites/benchmark.yaml
      - --loop
      - --threads=2
      - --sleep=3ms
```

That deployment generates ~584 RPS sustained against
the API, 24/7. The Grafana dashboard shows:

- RPS hovering at ~584.
- p95 latency around 0.6 ms.
- 5xx rate at 0.
- 4xx rate at 0 (the baseline is happy-path only).

That's the **dashboard signature** of a healthy
movies-api. Any deviation — RPS drops, p95 climbs, 4xx
appears — is visible immediately, because the chart
shape changes.

The negative-path suite is the same shape, run on
demand:

```sh
$ kubectl exec -n movies deploy/webv -- /webv \
    --url=http://movies-api.movies.svc.cluster.local:8080 \
    --files=/webv-suites/test.yaml
```

The dashboard responds: 4xx rate jumps from 0 to
several per second for the duration of the run, then
drops back to 0. That's the *test of the dashboard* —
the negative-path suite proves the alerting / charting
infrastructure responds to errors when they happen.

#### Lab

1. Open the Grafana dashboard for movies-bartr. Find the
   four panels of the baseline signature (RPS, p95, 5xx
   rate, 4xx rate). Note their steady-state values.
2. Run `test.yaml` against the cluster. Watch the 4xx
   panel respond. Time how long it takes for the
   dashboard to reflect the change.
3. Stop the in-cluster `webv` deployment (`kubectl
   scale deploy/webv --replicas=0 -n movies`). Watch
   the RPS panel drop to 0. That's the *test of the
   baseline itself* — confirming the dashboard sees
   when the baseline is missing.
4. Add a new validation to `test.yaml`: a request to a
   path that doesn't exist (e.g. `/api/banana`).
   Predict: 404 in the 4xx panel. Confirm.
5. Add a JSON-shape validation to `test.yaml` for
   `/api/movies`: array with ≥ 10 elements. Confirm
   passes. Edit the handler to return only 5. Confirm
   the validation fails.

#### Knowledge check

- Why is "the dashboard signature *is* the test
  result" a stronger statement than "we run the test
  suite nightly"?
- The dashboard signature shifts permanently after a
  release — RPS drops 10%, p95 stays the same. Is the
  release broken? What's your next move?
- Helium added validation features one at a time, in
  response to outages. What's the alternative pattern,
  and why is the reactive pattern actually correct
  here?

---

### Module 5 — Dual instrumentation: client-side metrics + dual-format logs (named gap)

> **The original reason webv existed.** Every request emits
> a Prometheus histogram from the client side, plus a
> structured log line in one of two formats: tab-delimited
> for a human reading a small test on a terminal, or JSON
> for a log platform indexing at scale. The delta between
> client-side and server-side histograms is the latency
> that lives *outside* the service — and the only way to
> see it.

#### Concept

Two separate things, both shipped together in the original
webv, both currently missing from the movies-bartr port:

**1. Client-side Prometheus histograms.**

A service emitting `http_server_request_duration_seconds`
tells you what happened *inside the process*: receive
request, route, handle, write response. It does not tell
you:

- Time spent on the wire between client and server.
- Time queued at the LB / ingress / sidecar.
- Time stolen by the Linux scheduler on either end.
- Time the kernel spent in TCP retransmits.
- Time the cloud's underlay added on a noisy day.

The service can be returning in 0.6 ms internally while
the caller is seeing 200 ms p99 — and the server metrics
will never show it. **The only way to see it is to
instrument the client.**

**2. Dual-format structured logs — TSV *and* JSON.**

Logs are structured the moment they have a fixed shape
that a parser can rely on. TSV with a defined column order
is structured. JSON is structured. Free-form
`fmt.Println("thing happened: %v", x)` is not.

The interesting design choice the original webv made:
ship *both* formats, switchable by a CLI flag, because
the two consumers have opposite ergonomics:

| Consumer | Wants | Hates |
|---|---|---|
| **Human eyeballing a small run in a terminal** | Fixed columns, narrow lines, one line per request, easy to grep / `awk '{print $4}'` | JSON wrapping, quote noise, multi-line records, key repetition on every line |
| **Log platform (Loki / ELK / Splunk) at scale** | JSON with typed fields, predictable key names, easy to index and query without writing a parser | TSV with no field names — every consumer has to memorize the column order or write a regex |

Neither format wins. The right answer is **both, switchable
by `--json`**, with the same field set on both sides. The
theory: TSV for the laptop, JSON for the cluster.

> **Pro tip — TSV is a free Excel/Sheets pipeline.** TSV
> imports directly into Excel and Google Sheets with zero
> ceremony. Route a run's output to a `.xls` extension —
> `webv ... > run.xls` — and double-click it. Excel will
> warn that the file isn't really a `.xls`; click through.
> You now have columnar log data with filter / sort /
> pivot / chart in seconds, with no ETL step and no Loki
> query to write. It's the fastest path from "I want to
> look at this run" to "I have a chart of latency by path
> grouped by status code" that exists. JSON can't do this
> — every consumer has to parse it first. **This is a
> third consumer the TSV format earns its keep for: the
> analyst with a spreadsheet,** alongside the human at
> the terminal and the `awk`-and-`grep` toolchain.

**The pattern — what the original webv did, what the
movies-bartr port hasn't added yet:**

1. **Client emits the same metric shape as the server.**
   `webv_request_duration_seconds_bucket` with the same
   bucket boundaries (`{.001, .0025, .005, .01, .025,
   .05, .1, .25, .5, 1, 2.5, 5, 10}`) and the same labels
   (`method`, `path`, `status`). Prometheus scrapes
   `webv` the same way it scrapes `movies-api`.
2. **Both sides emit the same structured fields, in two
   formats.** TSV by default. `--json` flips to JSON
   handler (`log/slog`'s `NewJSONHandler` does this with
   no rewrite). Same fields, same names, same semantics
   — only the on-the-wire format changes.
3. **Correlation ID joins client and server.** Client
   sets `X-Request-Id: <uuid>` on every request; server
   logs it on receipt. A log query joins client emit →
   server receive → server complete → client complete
   for a single request, regardless of which side
   logged in which format.
4. **PromQL subtracts.** `histogram_quantile(0.95,
   sum(rate(webv_request_duration_seconds_bucket[5m]))
   by (le)) - histogram_quantile(0.95,
   sum(rate(http_server_request_duration_seconds_bucket{job="movies-api"}[5m]))
   by (le))` is one number with a clear name: **the p95
   latency outside the server.**

With that one PromQL expression, you have a continuous,
dashboardable measure of network + queueing + scheduling
overhead. Without dual instrumentation, you don't —
you have two numbers in two systems and an argument about
which one is right.

#### The distance-vs-volatility story

The single most important thing dual instrumentation
taught the Helium team — and the reason this module
is worth its slot in the curriculum:

| Topology | Typical p50 delta (client − server) | p99 delta | Volatility |
|---|---|---|---|
| Same pod (loopback / sidecar) | tens of µs | low hundreds of µs | very low |
| Same node, different pod | low hundreds of µs | low ms | low |
| Same cluster, different node | ~1 ms | tens of ms | medium |
| Same region, different cluster | low ms | low hundreds of ms | medium-high |
| Different region, same cloud | tens of ms | hundreds of ms | high |
| Different cloud | tens to hundreds of ms | seconds possible | very high — often **bimodal** |

Three things this table shows that you can't see from
either side alone:

1. **The mean grows monotonically with distance** —
   expected, and visible from either side.
2. **The p99 grows faster than the mean as distance
   grows** — the tail gets heavier in absolute terms.
3. **Volatility (the *variance*) explodes with distance**
   — and that's the part nobody sees without dual
   instrumentation. Cross-cloud p99 isn't just "higher,"
   it's *less predictable*; the same call at the same
   rate can hit 50 ms or 800 ms five minutes later. The
   client-side histogram catches that. The server-side
   histogram doesn't — from the server's perspective
   it served in 0.6 ms either way.

This is the operational answer to "is our service slow"
that actually has signal: **how much of the latency is
ours, how much is the underlay, and how much variance
is the underlay adding.** Without webv-style client
instrumentation, the conversation devolves to "the
server looks fine, must be the network," with nobody
holding a number that proves it.

#### Example

The shape the Helium webv emitted on every request,
translated to movies-bartr-style:

**Metric** (`/metrics` endpoint on the webv pod):

```
# HELP webv_request_duration_seconds Time from client request emit to response received
# TYPE webv_request_duration_seconds histogram
webv_request_duration_seconds_bucket{method="GET",path="/api/movies",status="200",le="0.001"} 0
webv_request_duration_seconds_bucket{method="GET",path="/api/movies",status="200",le="0.0025"} 412
webv_request_duration_seconds_bucket{method="GET",path="/api/movies",status="200",le="0.005"} 28401
webv_request_duration_seconds_bucket{method="GET",path="/api/movies",status="200",le="0.01"} 29882
...
webv_request_duration_seconds_count{method="GET",path="/api/movies",status="200"} 29940
webv_request_duration_seconds_sum{method="GET",path="/api/movies",status="200"} 95.21
```

**Log line, TSV mode** (current movies-bartr format —
fixed columns, one line per request, scannable in a
terminal):

```
2026-06-08T13:14:15Z	PASS	200	0.642ms	GET	/api/movies	1572	application/json
```

**Log line, JSON mode** (what the `--json` flag should
produce — same fields, structured for a log platform):

```json
{
  "ts": "2026-06-08T13:14:15.123Z",
  "level": "info",
  "msg": "request_complete",
  "request_id": "7d4a9c0e-2b13-4f8a-9e6d-1c4a5b6e7f80",
  "method": "GET",
  "path": "/api/movies",
  "status": 200,
  "duration_ms": 0.642,
  "bytes": 1572,
  "content_type": "application/json",
  "pass": true
}
```

Same data. Two consumers, two ergonomic shapes. The
TSV is what you read when you're tailing the pod log
from a terminal during a release. The JSON is what
Loki indexes when the operator queries "every webv
request in the last hour where `duration_ms > 100`
for `path=/api/movies`."

The server emits a matching JSON log line on the same
`request_id` (it's already on JSON — servers never
benefit from TSV at scale). A log-query join produces
the full timeline of one request through both sides.

And the PromQL panel that earns its keep:

```promql
# p95 latency outside the server
histogram_quantile(
  0.95,
  sum(rate(webv_request_duration_seconds_bucket{path="/api/movies",status="200"}[5m])) by (le)
)
-
histogram_quantile(
  0.95,
  sum(rate(http_server_request_duration_seconds_bucket{job="movies-api",path="/api/movies",status="200"}[5m])) by (le)
)
```

One number on the dashboard. Steady-state value is the
topology's baseline overhead. When it jumps, the network
/ underlay changed, not the service.

#### Lab

*(This lab anticipates the enhancement — movies-bartr's
webv has TSV today but no `--json` mode and no metrics
yet. The lab walks the operator through adding both,
which is the right way to internalize the pattern.)*

1. **Add `--json` to the existing writer.** Refactor
   `writer.emit` in `src/cmd/webv/runner.go` so the
   formatter is pluggable: keep the current TSV
   formatter as the default, add a JSON formatter
   selected by `--json`. Both must produce the same
   field set with the same names — only the encoding
   changes. (Use `log/slog` with `NewJSONHandler` to
   avoid hand-writing the encoder.)
2. **Generate a `request_id` per request.** Set it as
   `X-Request-Id` on the outgoing HTTP request. Include
   it in both TSV (as a column, e.g. position 2) and
   JSON (as the `request_id` field). Confirm the server
   logs it on receipt — if it doesn't, that's a
   movies-api gap to fix in the same PR.
3. **Add a `/metrics` HTTP server to `webv`.** Use
   `prometheus/client_golang` with a per-router registry
   (mirror the pattern from
   [study-guide-go.md](study-guide-go.md) Module 7).
   Register one histogram: `webv_request_duration_seconds`
   with the bucket set above.
4. Wire `runner.doOne` to observe duration into the
   histogram with `(method, path, status)` labels.
5. Add a Prometheus `ServiceMonitor` for the webv
   deployment (cross-reference
   [study-guide-k8s-core.md](study-guide-k8s-core.md)
   Module 10).
6. Add a Grafana panel for the delta PromQL above.
7. Run two `webv` deployments simultaneously:
   - One in `namespace: movies` (same cluster, likely
     same node as movies-api), `--json` enabled,
     scraped by the cluster's Prometheus.
   - One on a separate VM outside the cluster, hitting
     the public LB, also `--json`, scraped by the same
     Prometheus over the public endpoint or shipped via
     remote-write.
   Compare the delta panel for the two. The cross-VM
   webv's delta should be measurably larger — and
   noticeably more variable.
8. (Optional, advanced.) Run a third webv in a different
   region or different cloud. Compare all three. The
   p99 delta and the variance both grow with distance;
   confirm the table above against your real numbers.
9. **Operator sanity check on output choice.** Run a
   single-pass test from your laptop without `--json`
   — the TSV output should be more pleasant to read
   than the JSON would be. Re-run with `--json` and
   pipe to `jq` — confirm it's more pleasant to query
   than the TSV would be. Both ergonomics are real;
   neither format is universally better.
10. **TSV → Excel ad-hoc analytics.** Route a 60-second
    TSV run to `run.xls`:
    ```sh
    $ kubectl exec -n movies deploy/webv -- /webv \
        --url=http://movies-api.movies.svc.cluster.local:8080 \
        --files=/webv-suites/benchmark.yaml \
        --duration=60s --verbose > run.xls
    ```
    Open it in Excel (click through the format-warning
    dialog) or upload to Google Sheets. Add a header row
    if needed. Build a pivot table: `path` as rows,
    average `dur` as values, `status` as a slicer.
    Notice how much faster this is than writing the
    equivalent PromQL or LogQL. This is the third TSV
    consumer the format earns its keep for.

#### Knowledge check

- TSV with a fixed column order is structured. So is
  JSON. Name *three* consumers the TSV format serves
  well that JSON serves poorly — and one consumer
  where the reverse is true.
- Why does the client-side histogram need the *same
  bucket boundaries* as the server-side histogram for
  the subtraction to be meaningful?
- The server-side p95 is 0.6 ms. The client-side p95
  (from outside the cluster) is 12 ms. A teammate says
  "the service is slow." What's the correct framing?
- Cross-cloud p99 jumped from 80 ms to 600 ms over the
  last hour. The server-side p99 is unchanged. What's
  happening, and what dashboard panel proves it?
- Why is `X-Request-Id` set by the *client*, not
  generated by the server?
- A teammate proposes dropping the JSON log mode and
  keeping only TSV — "we have the metric, and TSV is
  easier to read." What's lost when the test runs at
  scale in the cluster?

---

### Module 6 — Load generation tools: when each is right (I4)

#### Concept

Four tools cover the load-generation surface; each has
a job:

| Tool | What it's right for | What it's wrong for |
|---|---|---|
| **Web Validate (`webv`)** | Contract suites + sustained baseline; validation assertions; in-cluster loop | Burst load tests; sub-ms latency measurement |
| **`k6`** | Scripted scenarios with logic (login → fetch → mutate); browser-like sessions; ramp profiles | Simple "hammer this endpoint" |
| **`vegeta`** | Sustained, precise rate-controlled HTTP load with rich histogram output | Anything that needs login flow or scripting |
| **`hey`** | One-off "is this thing serving" smoke shots from a terminal | Anything you need to script or repeat |

The right move is usually **one of each layer**:

- `webv` running in-cluster for the 24/7 baseline.
- `vegeta` from a workstation when you need a precise
  rate sweep ("does it hold at 1000 RPS, 1500, 2000").
- `k6` when the scenario has logic (rare for a GET-only
  API; common for anything with auth or state).
- `hey` for `hey -n 100 -c 10 http://...` smoke when
  you just want to see if the LB answers.

Don't try to make one tool do all four jobs. They each
have a small surface and are pleasant to use when
applied to their actual purpose.

#### Example

movies-spec §10.4 target: 500 RPS sustained, p95 ≤ X,
0 % 5xx. Three tools, three jobs, on the same target:

**`hey` — does the service answer at all:**
```sh
$ hey -n 100 -c 10 http://movies.local/api/movies
```

**`vegeta` — does it hold at 500 RPS for 60 seconds:**
```sh
$ echo 'GET http://movies.local/api/movies' | \
    vegeta attack -rate=500 -duration=60s | \
    vegeta report -type=hdrplot
```

**`webv` (in-cluster) — does it hold *continuously*
without anyone watching:**
```yaml
# Already running 24/7 from the Deployment above.
# The Grafana panel shows the answer.
```

Three different questions, three different tools, none
of them duplicating the others' work.

#### Lab

1. Install `vegeta` and `hey`. Run each against the
   movies-bartr LB.
2. Pick the §10.4 target (500 RPS, 60 s). Run vegeta at
   that rate and capture the report. Compare the
   reported numbers to the Grafana dashboard's view of
   the same window.
3. Run `vegeta` at 2× the target (1000 RPS, 60 s).
   Predict what happens — error rate climbs?  latency
   climbs? Confirm. Use this as the input to Module 8's
   methodology lab.

#### Knowledge check

- The §10.4 target is 500 RPS. Why is the in-cluster
  `webv` running at ~584 RPS, not 500 exactly?
- A teammate proposes replacing the in-cluster `webv`
  with a `k6` job that runs every 5 minutes. What's
  lost?
- `hey -n 100000 -c 100` finishes in 30 seconds, no
  errors. The service is up to spec — true or false?

---

### Module 7 — Reading benchmarks: p50/p95/p99 vs mean (I5)

> Mean lies. Always.

#### Concept

The single most important benchmark literacy: **don't
quote the mean for latency.** Latency distributions are
heavy-tailed; a handful of slow requests pull the mean
up and tell you nothing useful about the median user
experience.

The four numbers that matter:

| Number | What it tells you | When it's the right number |
|---|---|---|
| **p50** (median) | What half your users experience | The "typical" experience |
| **p95** | What 5% of users hit (worst-1-in-20) | SLO target; if p95 is bad, real users are seeing bad |
| **p99** | What 1% of users hit (worst-1-in-100) | Tail-latency tracking; outlier behavior |
| **Mean** | The arithmetic average | **Almost never.** Useful only when comparing to median to detect tail-skew. |

Two more that matter:

- **RPS** (throughput) — does the service serve at the
  target rate without queueing?
- **Error rate** — what % of requests returned 5xx?
  4xx is sometimes correct behavior (the test
  *should* fail input validation) — but for a happy
  path, both should be 0.

The relationship between latency and load:

- At low load, p50 and p95 are close.
- As load climbs, p95 (and p99) climb *much* faster
  than p50 — that's the "knee" of the curve.
- The knee is where the service is starting to queue.
  Past the knee, every metric degrades together.

#### Example

A real `vegeta` report on a movies-api at 500 RPS:

```
Requests      [total, rate]       30000, 500.02
Duration      [total, attack]     60.001s, 60.000s
Latencies     [mean, 50, 95, 99, max]  0.7ms, 0.6ms, 1.1ms, 2.4ms, 18.3ms
Bytes In      [total, mean]       45.0 MB, 1572
Bytes Out     [total, mean]       0, 0
Success       [ratio]             100.00%
Status Codes  [code:count]        200:30000
```

Reading this:

- p50 = 0.6 ms — typical request finishes very fast.
- p95 = 1.1 ms — almost all requests finish quickly.
- p99 = 2.4 ms — outliers exist but aren't extreme.
- max = 18.3 ms — single worst request (probably GC).
- mean = 0.7 ms — close to p50, which means the tail
  isn't pulling things up much.
- 0 errors, full success rate, exactly 500 RPS as
  requested.

Compare to a degraded service at the same rate:

```
Latencies     [mean, 50, 95, 99, max]  47ms, 0.8ms, 240ms, 920ms, 4.1s
Success       [ratio]             99.7%
Status Codes  [code:count]        200:29910, 504:90
```

p50 is still 0.8 ms (most requests are fine). Mean is
47 ms — pulled up by a slow tail. p95 is 240 ms —
*5% of users are hitting a quarter of a second.* p99
is 920 ms — *1 in 100 users is waiting nearly a
second*. And 0.3% are timing out (504s).

If you only looked at the mean, you'd think things
were a little slow. If you only looked at p50, you'd
think everything was fine. The story is in the p95 /
p99 / error-rate.

#### Lab

1. Run a `vegeta` attack at the §10.4 target rate
   against movies-bartr. Capture the report.
2. Run another at 2× the target. Compare the four
   latency numbers. Where's the "knee"?
3. Stop one of the movies-api replicas (`kubectl
   scale deploy/movies-api --replicas=1`). Re-run.
   Note where the numbers change — and how the
   ratio of p95 / p50 shifts as resources get tighter.
4. Find the same window in the Grafana dashboard.
   Confirm the dashboard's percentile panels agree
   with vegeta's numbers (they should, within the
   limits of sampling).

#### Knowledge check

- A teammate says "the service runs at 50 ms mean."
  What's the next question you ask?
- p50 = 5 ms, p95 = 5.2 ms, p99 = 5.5 ms. What does
  the shape tell you?
- p50 = 5 ms, p95 = 50 ms, p99 = 800 ms. What does
  the shape tell you, and what's the most likely
  cause?

---

### Module 8 — Performance methodology: baseline → change one → measure (I6)

#### Concept

The discipline that separates "we improved performance"
from "we think performance is better":

1. **Establish baseline.** Run the same load with the
   same configuration. Record the four numbers (p50 /
   p95 / p99 / RPS) and the error rate. Write them
   down with the date, git SHA, and any non-default
   settings.
2. **Change exactly one thing.** Not two, not three —
   one. If you changed two things and the numbers
   improved, you don't know which change did it.
3. **Re-measure.** Same load, same config, same
   timing window. Record the same numbers.
4. **Record the delta.** Did it improve? By how much?
   Was the change worth the cost (complexity,
   maintenance, risk)?
5. **Decide.** Keep, revert, or iterate.

The trap: skipping step 2 and changing five things at
once "because it'll be faster." Numbers might
improve, but you've lost the ability to attribute
*which* change drove the improvement. Next time the
system degrades, you have no way to roll back the bad
change without rolling back the good ones.

The methodology is identical at every scale: tuning a
GC parameter, tuning a deployment's resource limits,
tuning a Postgres connection pool, tuning a kernel
sysctl — same five steps, same discipline, same
record-keeping.

#### Example

From movies-bartr's `docs/PERFORMANCE.md`, the math
for why the in-cluster `webv` runs at `--threads=2
--sleep=3ms`:

```
Goal:        500 RPS sustained against movies-api
Constraint:  webv pod has 250m CPU limit
Observation: single-thread RPS jumps in big steps
             because Go's time.Sleep is quantized
             by the kernel timer slice (~1 ms)
             Sleep=1ms → 800 RPS, Sleep=2ms → 428 RPS,
             Sleep=3ms → 293 RPS. No setting hits ~500.
Change:      Two threads × sleep=3ms = 584 RPS sustained
Result:      584 RPS at 0% 5xx, p95=0.6ms — meets
             §10.4, doesn't saturate the pod.
```

That's the methodology in one paragraph. Baseline
(single thread, every sleep value tried, none works).
Change one thing (add a second thread). Measure (584
RPS, 0 errors, p95 = 0.6 ms). Record. Decide (keep).

Note what's *not* there: nobody changed
`--threads=2`, `--sleep=3ms`, *and* the pod's CPU
limit, *and* the keepalive setting, all at once.

#### Lab

1. Pick one tuning parameter in movies-bartr (e.g.
   the SQLite cache size, the HTTP server's
   `ReadTimeout`, the worker pool size). Establish
   the baseline at 500 RPS.
2. Change one value. Re-measure. Record the delta in
   a markdown file in `docs/`.
3. Now make a *deliberately bad* change (drop the
   memory limit to 32Mi, or set `MaxConnsPerHost: 1`).
   Re-measure. Confirm the methodology surfaces the
   regression cleanly.
4. Revert. Confirm the numbers return to baseline.

#### Knowledge check

- A PR says "improved throughput by 30%." What two
  questions do you ask before reviewing?
- You change one parameter, p95 improves 20%, p99
  degrades 100%. Did performance improve?
- The baseline number you recorded six months ago
  isn't reproducible today on the same code. Three
  possible causes — what are they?

---

### Module 9 — Continuous testing in production (capstone)

> The 48-hour smoke test isn't the lesson. The lesson
> is that the baseline runs forever, and you know what
> its dashboard looks like.

#### Concept

Pull the previous seven modules together. The pieces:

- **Contract suite** (Module 3) that hits the live
  service over the real LB.
- **In-cluster runner** (Module 4) that runs that
  suite continuously, generating a known signal in
  the metrics.
- **Dashboard signature** (Module 4 + observability
  guide) that shows the signal as a chart shape
  everyone has memorized.
- **Negative-path suite** (Module 3) that, on
  demand, produces a known dashboard *delta* —
  proving the alerting infrastructure responds.

These four together give you a production system
that is **always being tested**, in a way that's
visible to anyone glancing at the dashboard. That's
the structural property the Helium 48-hour smoke
test was built around.

The release pattern this enables:

1. **Bake-time before promotion.** Don't release
   straight into the ring above without letting the
   new version run against the baseline for a
   period — long enough that the dashboard
   signature would have shown any regression. At
   Helium that was 48 hours. In your environment
   it might be 4 hours, or 2 weeks. The number
   isn't the point; the *practice* is.
2. **The dashboard is the gate.** "Did the signature
   hold during bake?" Yes → promote. No → fix or
   revert. Combine with the rings + git revert
   pattern from [study-guide-gitops.md](study-guide-gitops.md)
   Module 3.
3. **No release ever runs without an in-cluster
   baseline already exercising the new version.**
   If the baseline isn't running, the release isn't
   tested in production — it's just deployed.

The trap: confusing "we have CI" with "we have
production testing." CI tested the artifact. The
in-cluster baseline tests the *deployment* —
including the manifest, the config, the secrets, the
network policy, the service mesh, the cluster
itself. CI cannot test any of those.

#### Example

A release of movies-api 1.4.2 through the rings, with
the baseline running in each:

| Ring | Bake | Signal | Decision |
|---|---|---|---|
| dev | 30 min | webv baseline holds 584 RPS, 0 errors, p95=0.6ms | Promote to test |
| test | 2 hr | Same signature in test cluster | Promote to staging |
| staging | 24 hr | Same signature, including overnight | Promote to prod |
| prod | continuous | Baseline runs forever; signature is the SLO | — |

If at any ring the signature shifts (RPS drops, p95
climbs, error rate moves off 0), the decision is
fix-or-revert, *before* promotion. The dashboard is
the gate. Nobody has to read 47 alerts; everyone has
to glance at one chart.

This is also the answer to "how do you know your
alerting works." The negative-path suite is the
test of the dashboard. Run it monthly in staging.
If the 4xx panel doesn't twitch, the alerting is
broken — fix it before you find out in prod.

#### Lab

1. In movies-bartr, document the baseline dashboard
   signature: take a screenshot, write down the
   steady-state values (RPS, p50, p95, 5xx rate,
   4xx rate).
2. Release a deliberate regression (a handler that
   sleeps 100ms on every request). Watch the
   signature shift on the dashboard. Note how long
   it took to notice without anyone reading an
   alert.
3. Revert. Confirm the signature returns.
4. Run the negative-path suite (`test.yaml`).
   Confirm the 4xx panel responds, then returns to
   0. Time the response.
5. Disable the in-cluster `webv` deployment for one
   day. Confirm: no baseline signal in the
   dashboard. This is the "are we actually testing
   in production" smoke test for the smoke test.

#### Knowledge check

- Why does this guide insist that the in-cluster
  baseline is part of the *production system*, not
  part of the test suite?
- A teammate says "we ran the contract suite in CI,
  it's green, we're good to ship." What's missing?
- The baseline has been running for six months.
  Nobody has looked at the dashboard in three
  weeks. Is the baseline still serving its purpose?
- You inherit a service with no baseline running.
  What's the first move?

---

## Per-release review

Per the curriculum-wide template in
[study-guide-observability.md](study-guide-observability.md):
every release runs the cold-cluster read (K8s-core M12),
the security capstone (security M7), the image-size
budget check (containers per-release), the inner-loop
signal check (dev-loop per-release), and **this guide's
residual: the baseline-still-signaling check.**

Before promoting any release: confirm the in-cluster
`webv` deployment is running, the dashboard signature
is at its known shape, and a negative-path run on
demand produces the expected dashboard delta. If any
of the three is missing, the release is not ready —
not because the artifact is broken, but because the
*test of the deployment* isn't running.

## What this guide is

- The integration module for the testing pyramid plus
  the four pieces that make "continuous testing in
  production" structural (contract suite, in-cluster
  runner, dashboard signature, negative-path suite).
- Anchored in movies-bartr's actual in-cluster `webv`
  deployment, which is a curriculum port of the
  Helium / Web Validate pattern (Microsoft, MIT).
- The capstone for the "we don't stop testing when CI
  goes green" posture every other guide in the
  curriculum assumes.

## What this guide is not

- Not a full Web Validate reference — the canonical
  one is at https://github.com/microsoft/webvalidate.
- Not a tutorial on writing unit or integration tests
  in Go — that's covered in
  [study-guide-go.md](study-guide-go.md) Module 8.
- Not a comprehensive perf-engineering text — the
  methodology is one module; deep performance work
  is its own discipline.

## Open questions

1. Module 4 frames Helium's validation-feature
   evolution as reactive (each feature came from an
   outage). Is that the right teaching pattern, or
   should the guide front-load a complete validation
   taxonomy?
2. Module 9's "bake time" is left as a "you decide,"
   ranging from 30 min to 48 hr. Should the guide
   give a heuristic (e.g. "long enough that two full
   diurnal traffic cycles complete in prod, or 4 hr
   in pre-prod")?
3. The negative-path suite is currently a manual
   on-demand run. Should the curriculum push toward
   running it on a cron in non-prod (e.g. nightly
   at 03:00) so the alerting infrastructure is
   continuously confirmed?

## Status

DRAFT. Not promoted to `methodology/` until at least
one operator runs the in-cluster baseline through a
release cycle and confirms the dashboard-signature
gate works as described — the bake-time and
promotion-gate parts are the most procedurally
load-bearing and need lived experience to validate.
