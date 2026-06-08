# Study Guide — Go for K8s-Native Data Services (canonical language, domain M)

> **DRAFT — NOT FOR PUBLICATION.** Third instance of the study-guide
> format established by
> [study-guide-observability.md](study-guide-observability.md) and
> [study-guide-kustomize.md](study-guide-kustomize.md). Scoped from
> domain M of
> [skills-inventory.md](skills-inventory.md) — Go is the canonical
> language for every example in the curriculum; this guide is the
> language layer the other guides assume.

## Why this exists

The other guides (observability, Kustomize, security, GitOps) use Go
for their canonical examples. This one teaches the Go you need to
*read* and *modify* those examples without the agent doing it for you.
It is deliberately not a Go textbook; it is the slice of Go that
maps onto the movies-spec requirements and the
[skills-inventory.md](skills-inventory.md) rows M1–M17.

The Matt-v1 gap (see
[sessions-and-skill-compounding.md](sessions-and-skill-compounding.md))
applies here too: the agent will write `signal.NotifyContext` +
`http.Server.Shutdown` correctly the first time. The operator who
can't read it can't tell when the agent's next refactor breaks
graceful shutdown.

**What this guide is anchored in:**

- The real movies-bartr Go code: `cmd/movies-api/main.go`,
  `internal/httpapi/*`. Every example references a file in the repo.
- The Prometheus client library (`prometheus/client_golang`) and
  `log/slog` (Go 1.21+ stdlib) — the libraries the spec implicitly
  requires for §7.1 and §7.2.
- Spec §9 build / packaging (single multi-stage Dockerfile, distroless
  static, non-root) — covered for the *language* side in this guide;
  the Dockerfile side is in
  [study-guide-kustomize.md](study-guide-kustomize.md) Module 6.

## How to use this guide

Same protocol as the other study guides:

- Pick the module mapping to upcoming work; don't read front-to-back.
- Run the lab with your own hands.
- At each tag, run the per-release review.

**One curriculum-level rule for this guide specifically:** every lab
asks you to *read existing movies-bartr code*, then *modify it
deliberately*, then *put it back*. The reading half is non-negotiable.
The agent will write Go for you; the question this guide answers is
"can you read what it wrote and tell whether it's correct."

## Modules

### Module 1 — Project layout: `cmd/`, `internal/`, `pkg/` (inventory M1)

#### Concept

Go has a project-layout convention that's nearly universal in modern
codebases:

- **`cmd/<binary-name>/main.go`** — the entry point for each binary.
  A repo can have many `cmd/*` directories; each becomes a separate
  binary. movies-bartr has `cmd/movies-api/` and `cmd/webv/`.
- **`internal/`** — packages importable *only* from within this
  module. The Go toolchain enforces this. It's how you signal "this
  is an implementation detail, not API."
- **`pkg/`** — packages explicitly intended for external import.
  Optional convention; many small services skip it. movies-bartr does.

The lever `internal/` provides is real: if a future change moves a
package into `pkg/`, you've made a public-API commitment that's hard
to take back. Starting in `internal/` keeps options open.

#### Example

The movies-bartr tree:

```
src/
├── go.mod                           # module github.com/bartr/bartr-movies
├── Dockerfile
├── cmd/
│   ├── movies-api/main.go           # the service binary
│   └── webv/                        # the validation tool (separate binary)
└── internal/
    ├── config/                      # flag/env parsing
    ├── httpapi/                     # router, handlers, middleware, metrics
    ├── store/                       # in-memory data + loaders
    └── version/                     # the version constant
```

`main.go` imports from `internal/*`; nothing outside this module can.

#### Lab

```bash
cd repos/movies-bartr/src
cat go.mod | head -5
ls cmd/ internal/
```

Then:

1. Try to import `github.com/bartr/bartr-movies/internal/store` from a
   *new module* (create a scratch `go.mod` in `/tmp`, write a one-liner
   that imports it, run `go build`). Read the error.
2. Look at `cmd/movies-api/main.go` — confirm every import either
   comes from the stdlib, an external module (`github.com/...` listed
   in `go.mod`), or `github.com/bartr/bartr-movies/internal/...`. No
   relative imports, no `pkg/`.
3. Move one file from `internal/version/` to a new `pkg/version/`
   directory and update the import in `main.go`. Build. Note that it
   still works (`pkg/` has no enforcement). Put it back.

#### Knowledge check

1. What's the difference between `internal/foo/` and `pkg/foo/`?
   What does the toolchain actually enforce?
2. Why does movies-bartr have two `cmd/*` directories? What would
   one big `main.go` cost you?
3. The module path is `github.com/bartr/bartr-movies`. If you fork
   and rename, what breaks if you *don't* update the module path?
4. When would you create a `pkg/` directory in this codebase?

---

### Module 2 — HTTP server idioms: `net/http`, chi, middleware (inventory M2)

#### Concept

Go's stdlib HTTP server is genuinely production-grade. Two layers:

- **`net/http`** — the server (`http.Server`), the router
  (`http.ServeMux`, upgraded in Go 1.22 to support method + path
  patterns), and the handler interface (`http.Handler` with one method,
  `ServeHTTP(w, r)`).
- **`chi`** (`github.com/go-chi/chi/v5`) — a thin router with richer
  pattern matching, URL params, sub-routers, and a built-in
  middleware stack. Builds on `net/http`, doesn't replace it.

A **handler** is anything with `ServeHTTP(w, r)`. A **middleware** is
a function that wraps one handler and returns another:

```go
type Middleware func(http.Handler) http.Handler
```

That signature is the entire abstraction. Logging, metrics, auth,
recovery, request IDs — all the same shape. Compose them by
chaining: `mw1(mw2(mw3(realHandler)))`.

Matt-v1 used the stdlib `http.ServeMux` (Go 1.22+). movies-bartr uses
chi. Both are correct; chi wins when you want URL params
(`/api/movies/{id}`), sub-routers, and pre-built middleware. Stdlib
wins when you want zero external deps.

#### Example

The movies-bartr router setup (simplified from
`internal/httpapi/router.go`):

```go
r := chi.NewRouter()
r.Use(requestLogger())       // middleware 1
r.Use(m.middleware())        // middleware 2 (metrics)
r.Get("/healthz", handleHealthz)
r.Get("/readyz", handleReadyz(readyFn))
r.Get("/version", handleVersion(ver))
r.Get("/metrics", m.handler().ServeHTTP)

r.Route("/api", func(r chi.Router) {
    r.Get("/movies", h.listMovies)
    r.Get("/movies/{id}", h.getMovie)
    r.Get("/actors", h.listActors)
    r.Get("/actors/{id}", h.getActor)
    r.Get("/genres", h.listGenres)
})
```

A handler:

```go
func handleVersion(ver string) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "text/plain")
        fmt.Fprintln(w, ver)
    }
}
```

A middleware (the stdlib pattern, from `internal/httpapi/middleware.go`):

```go
func requestLogger() func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()
            rec := &statusRecorder{ResponseWriter: w, status: http.StatusOK}
            next.ServeHTTP(rec, r)
            slog.LogAttrs(r.Context(), slog.LevelInfo, "http_request",
                slog.String("path", r.URL.Path),
                slog.Int("status", rec.status),
                slog.Int64("duration_ms", time.Since(start).Milliseconds()),
            )
        })
    }
}
```

Notice: the middleware wraps `next` and calls `next.ServeHTTP(rec, r)`.
The `rec` is a thin shim that captures the status code — because
`http.ResponseWriter` doesn't expose what was written. This pattern
shows up in every middleware that cares about the response.

#### Lab

In `repos/movies-bartr/src/`:

1. Read `internal/httpapi/router.go` end to end. For each `r.Get`,
   identify the file containing the handler function.
2. Read `internal/httpapi/middleware.go`. Trace what happens to a
   request to `/api/movies` — which middlewares run, in what order,
   and what fields the log line ends up with.
3. Add a new handler `r.Get("/whoami", ...)` that returns the
   request's `User-Agent` header as plain text. Build. Curl it.
4. Add a one-line middleware that adds an `X-App-Version` response
   header with the current version. Build, curl, confirm.

#### Knowledge check

1. What is the *exact* type signature of `http.Handler`? What about
   `http.HandlerFunc`? How is `HandlerFunc` related to `Handler`?
2. Middleware order matters. If `requestLogger` runs *before* the
   metrics middleware in `r.Use(...)`, which one wraps the other?
3. Why does the logging middleware use a `statusRecorder` wrapper
   instead of reading the status off the `ResponseWriter`?
4. The handlers in `internal/httpapi/handlers.go` take a `*Handlers`
   receiver. What's the alternative (closure-returns-HandlerFunc) and
   when does each one read more clearly?
5. movies-bartr uses chi. Matt-v1 used stdlib `http.ServeMux`. For
   `r.Get("/api/movies/{id}", ...)`, what's the stdlib-1.22+
   equivalent and what's the readability trade-off?

---

### Module 3 — `context.Context` and cancellation (inventory M3)

> **The lever for Module 4 (graceful shutdown), Module 7
> (Prometheus), and almost every K8s-aware behavior in this guide.**
> Get this one right and the rest gets easier.

#### Concept

`context.Context` is Go's standard way to carry three things through
a call chain: a **deadline**, a **cancellation signal**, and
**request-scoped values**. Every HTTP request you handle, every
database call you make, every shutdown sequence you write — they all
hang off a `Context`.

Three primitives:

- **`context.Background()`** — the root. Used at program start.
- **`context.WithCancel(parent)`** — returns a derived context and a
  `cancel()` function. Calling `cancel()` causes `ctx.Done()` to fire
  on this context *and every context derived from it*.
- **`context.WithTimeout(parent, d)` / `WithDeadline(parent, t)`** —
  same, but cancels automatically after the duration / at the time.

A `Context` is **always the first parameter** by convention:

```go
func (s *Store) FindMovie(ctx context.Context, id string) (*Movie, error) { ... }
```

The function should:

1. Check `ctx.Err()` (or `ctx.Done()`) at points where it can
   meaningfully abort.
2. Pass `ctx` to any downstream call (`http.NewRequestWithContext`,
   `db.QueryContext`, etc.).
3. **Never store a `Context` in a struct.** Always pass it as a
   parameter.

In an HTTP handler, the request's context (`r.Context()`) is **already
cancelled when the client disconnects.** That's the lever for "stop
doing expensive work when the user navigated away."

#### Example

The shutdown-context pattern from `cmd/movies-api/main.go`:

```go
// Root context that cancels on SIGINT or SIGTERM.
ctx, stop := signal.NotifyContext(context.Background(),
    syscall.SIGINT, syscall.SIGTERM)
defer stop()

// ... wait for signal ...
<-ctx.Done()

// Derived context with a hard deadline for shutdown.
shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()
if err := srv.Shutdown(shutdownCtx); err != nil {
    return fmt.Errorf("graceful shutdown: %w", err)
}
```

Notice: `shutdownCtx` is derived from `context.Background()`, not from
`ctx`. That's deliberate — by the time we call `srv.Shutdown`, the
parent `ctx` is *already* cancelled (that's why we got here). Deriving
from it would give us a context that's instantly done. We want
shutdown to have its own 10-second budget.

A logging middleware that uses the request context:

```go
slog.LogAttrs(r.Context(), slog.LevelInfo, "http_request", ...)
```

Passing `r.Context()` lets `slog` honor any request-scoped values or
deadlines on it.

#### Lab

1. In `cmd/movies-api/main.go`, change the shutdown timeout from 10s
   to 100ms. Build. Run. `kill -TERM` the process while a slow request
   is in flight. Observe what happens — `srv.Shutdown` returns an
   error because not all requests drained. Read the error.
2. Change it to 30s. Run again. Confirm clean shutdown.
3. Put it back to 10s.
4. Write a 5-line program that uses `context.WithTimeout(ctx, 50*time.Millisecond)`
   and calls an HTTP endpoint with `http.NewRequestWithContext`. Point
   it at a deliberately slow endpoint (you can `time.Sleep` inside a
   handler). Watch the request error out with a context-deadline
   error, not a network error.

#### Knowledge check

1. Why is `shutdownCtx` derived from `context.Background()` and not
   from the cancelled signal context?
2. If a handler ignores `r.Context()` entirely and runs a 30-second
   computation, what happens when the client disconnects after 1
   second? What *should* happen?
3. The rule "never store a `Context` in a struct" — what does the
   alternative look like in practice? (Hint: every method takes
   `ctx context.Context` as its first arg.)
4. `ctx.Err()` returns one of three things. What are they and when
   does each appear?
5. `signal.NotifyContext` was added in Go 1.16. What was the
   pre-1.16 idiom for the same thing, and why is the new one less
   error-prone?

---

### Module 4 — Graceful shutdown (inventory M4)

> **20 lines of code, 80% of the "why did my pod die uncleanly"
> tickets.** Matt-v1 had a partial version of this; movies-bartr has
> the complete pattern. Read the difference.

#### Concept

When Kubernetes wants to stop your pod, it:

1. Sends `SIGTERM` to the process.
2. Waits up to `terminationGracePeriodSeconds` (default 30s).
3. Sends `SIGKILL`.

If your process exits immediately on `SIGTERM`, in-flight requests die
with TCP RSTs and the client sees errors. If your process *ignores*
`SIGTERM`, every rolling deploy hard-kills you after 30s. The correct
behavior is:

1. Trap `SIGTERM`.
2. Stop accepting new connections.
3. Wait for in-flight requests to complete (with a deadline).
4. Exit cleanly.

`http.Server.Shutdown(ctx)` does steps 2–3. `signal.NotifyContext`
does step 1. You write the glue.

The shutdown deadline should be **less than `terminationGracePeriodSeconds`**.
If grace is 30s and your shutdown deadline is 60s, k8s wins and you
get `SIGKILL`'d mid-drain. movies-bartr uses 10s shutdown vs the k8s
default 30s grace — comfortable margin.

#### Example

The full pattern from `cmd/movies-api/main.go`:

```go
// Trap signals: ctx cancels on SIGINT or SIGTERM.
ctx, stop := signal.NotifyContext(context.Background(),
    syscall.SIGINT, syscall.SIGTERM)
defer stop()

// Run the server in a goroutine so we can wait on ctx in main.
errCh := make(chan error, 1)
go func() {
    if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
        errCh <- err
    }
}()

// Wait for either a signal or a server error.
select {
case <-ctx.Done():
    logger.Info("shutdown signal received")
case err := <-errCh:
    return err
}

// Drain in-flight requests with a hard deadline.
shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()
if err := srv.Shutdown(shutdownCtx); err != nil {
    return fmt.Errorf("graceful shutdown: %w", err)
}
```

The `errors.Is(err, http.ErrServerClosed)` is the line everyone
forgets: `ListenAndServe` returns `http.ErrServerClosed` when
`Shutdown` is called. That's not an error, it's the expected exit.
Without the `errors.Is` check you log spurious "server crashed"
messages on every clean shutdown.

#### Lab

1. Run movies-bartr locally (in the cluster, not bare).
2. Open two terminals: one runs a slow `curl` (`while true; do curl
   -s http://localhost:8080/api/movies > /dev/null; done`); the other
   runs `kubectl delete pod -n movies <pod-name>`.
3. Watch the request loop. Do you see errors during the rolling
   delete?
4. Now sabotage `cmd/movies-api/main.go`: delete the
   `srv.Shutdown(shutdownCtx)` block and just `return nil` on
   `ctx.Done()`. Build, redeploy, rerun the test. Read the error
   pattern in the client.
5. Restore. Confirm clean again.

#### Knowledge check

1. What does `http.Server.Shutdown(ctx)` actually do, step by step?
   What does it *not* do?
2. The shutdown deadline (10s in movies-bartr) needs to be less than
   what k8s value? What happens if it's larger?
3. `ListenAndServe` returns `http.ErrServerClosed` on graceful exit.
   What happens if you log this as an error?
4. What's the role of `defer stop()` after `signal.NotifyContext`?
   What goes wrong without it?
5. If your service has a long-running background goroutine (not an
   HTTP request — say a periodic data refresh), what changes about
   the shutdown logic?

---

### Module 5 — Error handling discipline (inventory M5)

#### Concept

Go's error story is values, not exceptions. Three rules cover ~90%:

1. **Return errors, don't panic.** `panic` is for "the program is in
   an unrecoverable state" — nil pointer on a corrupt data structure,
   etc. Bad input is not a panic.
2. **Wrap with `%w` to preserve the chain.** `fmt.Errorf("loading
   movies: %w", err)` lets callers walk the chain with `errors.Is`
   and `errors.As`.
3. **Check at the boundary, log once, return once.** Don't log *and*
   return — the caller will log too, and you end up with duplicate
   noise.

The library you reach for is `errors` (stdlib):

- `errors.Is(err, target)` — does this error chain contain `target`?
  Used for sentinels like `http.ErrServerClosed`, `io.EOF`.
- `errors.As(err, &target)` — does this chain contain something
  assignable to `target`? Used for typed errors with extra fields.
- `fmt.Errorf("ctx: %w", err)` — wrap, preserve the chain.

#### Example

From `cmd/movies-api/main.go`:

```go
if err := srv.Shutdown(shutdownCtx); err != nil {
    return fmt.Errorf("graceful shutdown: %w", err)
}
```

Wrapped: the caller can still detect the underlying cause:

```go
if err := run(args); err != nil {
    if errors.Is(err, context.DeadlineExceeded) {
        // shutdown timed out — we know exactly which one
    }
    fmt.Fprintln(os.Stderr, "movies-api:", err)
    os.Exit(2)
}
```

The flag-handling pattern (also from `main.go`):

```go
cfg, err := config.Load(args, os.Stderr)
if err != nil {
    if errors.Is(err, flag.ErrHelp) {
        return nil           // not an error — user asked for --help
    }
    return err
}
```

Notice the `errors.Is(err, flag.ErrHelp)` distinguishing "user asked
for help" (exit 0) from "user gave a bad flag" (exit non-zero).
That's the discipline.

#### Lab

1. Add a deliberate error to `config.Load` (e.g. require a flag
   that doesn't exist). Run `movies-api`. Confirm the exit code is 2.
2. Wrap an error with `fmt.Errorf("loading config: %w", err)`. Run
   again. Confirm the message shows both layers.
3. Replace the `%w` with `%v`. Run again. Note that the printed
   message looks identical, but now `errors.Is`/`errors.As` won't
   walk the chain. Restore.
4. Find a `panic(` in the codebase (there are very few). Read why
   it's safe to panic at that exact spot.

#### Knowledge check

1. `fmt.Errorf("...: %w", err)` vs `fmt.Errorf("...: %v", err)` —
   the printed output looks the same. What's the actual difference?
2. When is `panic` the right answer? Give two cases.
3. The "log once, return once" rule — what does the alternative
   (log and return) look like in real codebases? Why is it noisy?
4. `errors.Is` vs `errors.As` — when do you reach for each?
5. The `main.go` pattern `if err := run(args); err != nil { exit }`
   keeps `main` itself tiny. What does this buy you for testing?

---

### Module 6 — Structured logging with `log/slog` (inventory M6)

> **Spec §7.2 says JSON on stdout with `debug|info|warn|error` and a
> few required fields. `log/slog` (stdlib, Go 1.21+) is built for
> exactly this. Don't reach for `logrus` or `zap` unless you have a
> specific reason — stdlib is enough.**

#### Concept

`log/slog`:

- **Handlers** — `slog.NewJSONHandler(w, &slog.HandlerOptions{...})`
  for JSON-to-stdout, `slog.NewTextHandler` for human-readable.
- **Levels** — `LevelDebug` (-4), `LevelInfo` (0), `LevelWarn` (4),
  `LevelError` (8). Configurable at handler level.
- **Attrs** — typed key/value pairs: `slog.String("path", p)`,
  `slog.Int("status", 200)`, `slog.Int64("duration_ms", ms)`. Use
  `LogAttrs` for the fast path (no allocations for the variadic).
- **Default logger** — `slog.SetDefault(logger)` so package-level
  `slog.Info(...)` works everywhere.

The spec field discipline (§7.2):

- A timestamp (slog adds `time` automatically).
- A level (slog adds `level` automatically).
- A message (the first non-attr arg).
- Request-scoped attrs for HTTP logs.
- No PII, no bodies.

#### Example

The setup from `main.go`:

```go
func newLogger(level string) *slog.Logger {
    var lvl slog.Level
    switch strings.ToLower(level) {
    case "debug":
        lvl = slog.LevelDebug
    case "warn":
        lvl = slog.LevelWarn
    case "error":
        lvl = slog.LevelError
    default:
        lvl = slog.LevelInfo
    }
    h := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: lvl})
    return slog.New(h)
}
```

The request-logging middleware from `internal/httpapi/middleware.go`:

```go
slog.LogAttrs(r.Context(), lvl, "http_request",
    slog.String("method", r.Method),
    slog.String("path", r.URL.Path),
    slog.String("query", r.URL.RawQuery),
    slog.Int("status", rec.status),
    slog.Int("bytes", rec.bytes),
    slog.Int64("duration_ms", time.Since(start).Milliseconds()),
    slog.String("remote", r.RemoteAddr),
    slog.String("user_agent", r.UserAgent()),
)
```

Notice:

- `LogAttrs` instead of `Info` — typed, allocation-free, faster.
- Status-driven level: `>=500` → `Error`, `>=400` → `Warn`, else
  `Info`. Operators can filter on `level=error` and get only
  actionable lines.
- `/metrics` requests are explicitly skipped to keep the request log
  signal-heavy (Prometheus scrapes every 30s).
- The query string is logged; the body is not (spec §7.2).

Output:

```json
{"time":"2026-05-05T10:14:22Z","level":"INFO","msg":"http_request","method":"GET","path":"/api/movies","query":"","status":200,"bytes":18234,"duration_ms":0,"remote":"10.42.0.5:42138","user_agent":"curl/8.4.0"}
```

#### Lab

1. Run movies-bartr and tail the logs.
   ```bash
   kubectl logs -n movies deploy/movies-api -f | jq .
   ```
2. Hit `/api/movies?q=batman`. Confirm the query string appears in
   the log.
3. Hit `/api/movies/does-not-exist`. Confirm the level is `WARN`
   (because status is 404).
4. Set `MOVIES_LOG_LEVEL=debug` via the deployment env. Redeploy.
   Note the volume change.
5. **Anti-pattern recognition:** in a scratch program, call
   `slog.Info("user logged in", "password", pw)`. Run it. Note the
   password landed in the log. *This is why PII discipline lives in
   the middleware, not in trusting handlers to be careful.*

#### Knowledge check

1. `slog.Info(...)` vs `slog.LogAttrs(...)` — what's the difference
   and when do you pick `LogAttrs`?
2. Why does the request middleware skip `/metrics`? What problem
   does that prevent?
3. The level for a 404 is `WARN`, not `ERROR`. Why? When would you
   set it to `ERROR` instead?
4. movies-bartr emits one log line per request. For 10K RPS this
   becomes a lot. What's the standard trade-off — sample, increase
   level, both? What does the spec say?
5. `slog.SetDefault(logger)` makes the package-level `slog.Info`
   work. What's the downside of relying on the default logger
   everywhere?

---

### Module 7 — Prometheus client library (inventory M7)

> **Maps directly onto observability Module 1.** Read that one first
> for what the metrics *mean*; this one is how Go emits them.

#### Concept

The library is `github.com/prometheus/client_golang/prometheus` plus
`promhttp` for the HTTP handler. The three types:

- **`CounterVec`** — counter with labels. `requests.WithLabelValues("GET",
  "/api/movies", "200").Inc()`.
- **`HistogramVec`** — histogram with labels and explicit buckets.
- **`Gauge`** — single up/down value. No labels variant needed for
  movies-spec.

Two registry choices:

1. **Default global registry** (`prometheus.MustRegister(...)`) —
   convenient, but two routers in the same process panic with
   "duplicate metrics collector registration" on registration.
2. **Per-router registry** (`prometheus.NewRegistry()` +
   `promhttp.HandlerFor(reg, ...)`) — what movies-bartr does. Lets
   tests spin up multiple routers, lets the registry include Go
   runtime + process collectors alongside your app metrics.

The per-router choice is opinionated and worth defending.

**Label cardinality is the footgun.** A label like `user_id` or
`request_id` will blow up your timeseries count. Stick to bounded
labels — method (5 values), route template (~10 values), status code
(~10 values).

#### Example

The registry setup from `internal/httpapi/metrics.go`:

```go
m := &metrics{
    registry: prometheus.NewRegistry(),
    requests: prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total HTTP requests received, labeled by method, route template, and status code.",
        },
        []string{"method", "route", "code"},
    ),
    durations: prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "http_request_duration_seconds",
            Help: "HTTP request latency in seconds, labeled by method, route template, and status code.",
            Buckets: []float64{
                0.0001, 0.00025, 0.0005, 0.001, 0.0025,
                0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10,
            },
        },
        []string{"method", "route", "code"},
    ),
    inFlight: prometheus.NewGauge(...),
}
m.registry.MustRegister(
    collectors.NewGoCollector(),
    collectors.NewProcessCollector(collectors.ProcessCollectorOpts{}),
    m.requests, m.durations, m.inFlight,
)
```

The recording middleware (simplified):

```go
m.inFlight.Inc()
defer m.inFlight.Dec()

start := time.Now()
rec := &statusRecorder{ResponseWriter: w, status: http.StatusOK}
next.ServeHTTP(rec, r)

route, ok := apiRouteLabel(r.URL.Path)
if !ok {
    return
}
code := strconv.Itoa(rec.status)
m.requests.WithLabelValues(r.Method, route, code).Inc()
m.durations.WithLabelValues(r.Method, route, code).Observe(time.Since(start).Seconds())
```

The /metrics handler:

```go
return promhttp.HandlerFor(m.registry, promhttp.HandlerOpts{
    Registry: m.registry,
})
```

Sub-ms histogram buckets are specific to movies-spec — typical
service time is 20–100 µs. The default Prometheus buckets start at
5ms and would put every measurement in the first bucket, making p95
useless.

#### Lab

1. Read `internal/httpapi/metrics.go` end to end. For each bucket
   boundary, name the latency it's tuned to surface.
2. Curl `/api/movies` 10 times. Then curl `/metrics`. Find your 10
   requests in the counter output. Find them in the histogram
   buckets.
3. **Cardinality lab:** add a `query` label to the `requests` vector
   so it records the raw query string. Build, send 1000 requests
   with random `?q=` values, curl `/metrics`. Count the
   `http_requests_total` lines. Now revert — and never do this in
   production.
4. Change the histogram buckets to the Prometheus defaults
   (`prometheus.DefBuckets`). Build, run, curl `/metrics`. Note that
   every observation lands in the first bucket. This is what an
   un-tuned histogram looks like.

#### Knowledge check

1. Why does movies-bartr use a per-router registry? Name two
   problems the default global registry would cause here.
2. Bucket choice is workload-specific. What metric do you look at
   to decide buckets are wrong?
3. Cardinality: why does adding `user_id` as a label kill
   Prometheus, but adding `status code` does not?
4. The middleware does `m.requests.WithLabelValues(method, route,
   code).Inc()`. What does `WithLabelValues` return, and why is the
   `.Inc()` separate?
5. `collectors.NewGoCollector()` adds Go runtime metrics
   (heap, goroutines, GC). Why are those worth having on
   `/metrics` even though they're not in the spec?

---

### Module 8 — Testing: table-driven, `httptest`, coverage (inventory M8–M9)

#### Concept

Three test patterns cover almost everything in a service like
movies-bartr:

1. **Table-driven unit tests.** A slice of test cases, one loop.
   Idiomatic Go.
2. **`httptest.NewServer(handler)`.** Spin up a real HTTP server in
   the test process. Hit it with `http.Get` / `http.NewRequest`.
   This is integration-style — exercises the full middleware stack —
   without standing up a cluster.
3. **`httptest.NewRecorder()`.** In-memory `ResponseWriter`. Test a
   handler in isolation without a server. Faster, narrower.

Coverage:

- `go test -cover ./...` — per-package summary.
- `go test -coverprofile=cover.out ./...` then
  `go tool cover -html=cover.out` — line-by-line HTML report.
- Spec target: ≥80% on data and HTTP layers.

#### Example

A table-driven test (idiomatic shape):

```go
func TestAPIRouteLabel(t *testing.T) {
    cases := []struct {
        path string
        want string
        ok   bool
    }{
        {"/api/movies", "/api/movies", true},
        {"/api/movies/tt0111161", "/api/movies/{id}", true},
        {"/api/actors", "/api/actors", true},
        {"/api/actors/nm0000206", "/api/actors/{id}", true},
        {"/api/genres", "/api/genres", true},
        {"/healthz", "", false},
        {"/nonexistent/path", "", false},
    }
    for _, tc := range cases {
        t.Run(tc.path, func(t *testing.T) {
            got, ok := apiRouteLabel(tc.path)
            if got != tc.want || ok != tc.ok {
                t.Errorf("apiRouteLabel(%q) = (%q, %v), want (%q, %v)",
                    tc.path, got, ok, tc.want, tc.ok)
            }
        })
    }
}
```

A handler-level integration test:

```go
func TestVersionEndpoint(t *testing.T) {
    router := httpapi.NewRouter("0.9.9", func() bool { return true }, nil)
    srv := httptest.NewServer(router)
    defer srv.Close()

    resp, err := http.Get(srv.URL + "/version")
    if err != nil {
        t.Fatal(err)
    }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    if got := strings.TrimSpace(string(body)); got != "0.9.9" {
        t.Errorf("got %q, want %q", got, "0.9.9")
    }
}
```

#### Lab

1. `cd repos/movies-bartr/src && go test ./...`. Watch the output.
2. `go test -cover ./...` — find the per-package coverage numbers.
   Which package is highest, which is lowest, why?
3. `go test -coverprofile=cover.out ./internal/httpapi/ &&
   go tool cover -html=cover.out`. Open in browser, find an
   uncovered branch.
4. Write a new test for one uncovered branch. Run again, confirm
   coverage rose.
5. Run `go test -race ./...`. Note the runtime cost. Note that on a
   clean codebase it should pass cleanly.

#### Knowledge check

1. What's the difference between `httptest.NewServer(h)` and
   `httptest.NewRecorder()`? When do you reach for which?
2. Table-driven tests use `t.Run(name, fn)` for sub-tests. What
   does that buy you over a bare loop?
3. Coverage ≥80% is a *guide-rail*, not a goal. What's the spec's
   actual phrase, and what does it imply about *which* 80%?
4. `go test -race` adds significant runtime cost. When is it worth
   enabling in CI vs only locally?
5. movies-bartr tests live next to the code (`handlers_test.go`
   next to `handlers.go`). What does that buy over `tests/`-as-a-
   separate-tree?

---

### Module 9 — The toolchain: `gofmt`, `vet`, `staticcheck`, `golangci-lint` (inventory M14)

#### Concept

Go's default toolchain is opinionated, fast, and non-negotiable. Four
tools you run every session:

- **`gofmt`** — formatter. There is no style debate; gofmt is correct.
  Modern editors run it on save. CI runs `gofmt -l .` and fails if
  any file would change.
- **`go vet`** — built-in static analysis. Catches printf-format
  mismatches, copy-by-value-of-mutex, unreachable code. Fast, no
  config.
- **`staticcheck`** — third-party. The next level up: dead code,
  inefficient patterns, common bugs. Worth running.
- **`golangci-lint`** — meta-linter that runs many tools at once.
  Configurable via `.golangci.yml`. The CI default for most modern
  Go projects.

The cost of *not* running these is paid forever; the cost of running
them is seconds.

#### Example

A `.golangci.yml` snippet:

```yaml
run:
  timeout: 5m
linters:
  enable:
    - gofmt
    - govet
    - staticcheck
    - errcheck
    - ineffassign
    - unused
```

A `Makefile` target:

```makefile
.PHONY: lint
lint:
	gofmt -l . | tee /dev/stderr | (! read)
	go vet ./...
	golangci-lint run
```

#### Lab

1. `cd repos/movies-bartr/src && gofmt -l .`. Confirm no output
   (clean).
2. Deliberately misformat a file (add tabs in random places). Run
   `gofmt -l .` again. Note it lists the file. Run `gofmt -w .` to
   fix. Confirm clean.
3. `go vet ./...`. Read the output (likely nothing on a clean
   codebase).
4. Introduce a `printf` mismatch: `fmt.Printf("%d", "a string")`.
   Run `go vet`. Read the warning.
5. Install `staticcheck` (`go install honnef.co/go/tools/cmd/staticcheck@latest`)
   and run it. Note anything it finds.

#### Knowledge check

1. Why is the gofmt-is-the-style-guide convention worth defending
   in a code review?
2. `go vet` is in the toolchain; `staticcheck` is not. What's the
   trade-off Google made there?
3. `golangci-lint` runs many linters. What's the risk of enabling
   *too many* in CI?
4. When would you commit a `gofmt`-unclean file deliberately?
   (Hint: trick question.)
5. CI should fail on `gofmt -l . | wc -l` ≠ 0. Why is that the
   right check shape vs `gofmt -d .`?

---

### Module 10 — Build flags for tiny static binaries (inventory M11)

> **The lever for the 15MB-image story in
> [study-guide-kustomize.md](study-guide-kustomize.md) Module 6.**
> Three flags, ~40% size reduction, real stripped-down behavior.

#### Concept

A default `go build` produces a fat binary with symbol tables, debug
info, and the path-to-your-Go-source baked in. For a service that
runs in a container, you want it stripped, reproducible, and statically
linked.

The flags:

- **`CGO_ENABLED=0`** — disables cgo. Forces pure-Go networking,
  which means a fully static binary with no glibc dependency. Lets
  you `FROM scratch` or `FROM distroless/static`.
- **`-trimpath`** — strips the local filesystem path from compiled
  binaries. Makes builds reproducible across machines.
- **`-ldflags="-s -w"`** — strips symbol table (`-s`) and DWARF
  debug info (`-w`). ~25% size reduction. **Trade-off:** stack
  traces in panics no longer include line numbers in the binary.
  Acceptable for production where you ship the source-mapped binary
  to a debugger separately; not acceptable for development.
- **`-ldflags="-X main.version=..."`** — sets a string variable at
  link time. The standard way to bake the version into the binary
  without a generated `_version.go` file.

#### Example

The full build invocation (from movies-bartr Dockerfile):

```dockerfile
ARG VERSION=dev
RUN CGO_ENABLED=0 GOOS=linux go build \
      -trimpath \
      -ldflags="-s -w -X main.version=${VERSION}" \
      -o /out/movies-api ./cmd/movies-api
```

Building locally for size comparison:

```bash
cd repos/movies-bartr/src

# Fat default
go build -o /tmp/movies-api-fat ./cmd/movies-api
ls -lh /tmp/movies-api-fat

# Lean production
CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /tmp/movies-api-lean ./cmd/movies-api
ls -lh /tmp/movies-api-lean
```

Expect ~12MB → ~7-8MB.

#### Lab

1. Build both the fat and lean versions side by side. Record the
   sizes.
2. Run each. Confirm they behave identically (`./movies-api --help`,
   curl `/version`).
3. Cause a panic in each (add a `panic("test")` in a handler).
   Compare the stack traces. Lean has function names but no line
   numbers; fat has both.
4. Try `-ldflags="-s -w" -X main.version=$(git describe --tags)`.
   Run, curl `/version`. Confirm the version reflects your git tag.
5. Add `CGO_ENABLED=1` to the lean build. Run `file
   /tmp/movies-api-lean`. Note the dynamic linking — this is what
   keeps you from running on `FROM scratch`.

#### Knowledge check

1. Why does `CGO_ENABLED=0` matter for distroless/static base
   images?
2. `-trimpath` makes builds reproducible. What broke without it?
   (Hint: build attestation, two-machine binary diff.)
3. `-ldflags="-s -w"` is a trade-off. What's the cost, and how do
   real teams pay for it in production debugging?
4. The `-X main.version=...` pattern bakes a string at link time.
   What's the alternative (e.g. `_version.go` from a `go:generate`
   step), and why is `-X` usually cleaner?
5. The lean binary on movies-bartr is ~7-8MB. The runtime image is
   ~15MB. Where's the other 7MB?

---

### Module 11 — `go.mod`, `go.sum`, dependency hygiene (inventory M13)

#### Concept

`go.mod` declares the module path, the Go version, and direct
dependencies (with explicit versions). `go.sum` is the cryptographic
manifest — every dep at every version it's ever seen.

Three rules:

1. **`go mod tidy`** before every commit that touched imports.
   Removes unused deps, adds missing ones. Idempotent.
2. **Commit both `go.mod` and `go.sum`.** Never `.gitignore` them.
3. **Don't vendor unless you have a reason.** The module proxy
   (`proxy.golang.org`) plus `go.sum` gives you reproducibility.
   `vendor/` adds noise to PR diffs and burden to upgrades.

The `go` directive in `go.mod` declares the Go toolchain version
(e.g. `go 1.22`). It's a *minimum*, not an exact pin. The minimum-version-
selection algorithm picks the lowest version that satisfies all
constraints — the opposite of npm's most-recent-wins.

#### Example

The movies-bartr `go.mod`:

```
module github.com/bartr/bartr-movies

go 1.26.2

require (
    github.com/go-chi/chi/v5 v5.2.5
    github.com/prometheus/client_golang v1.23.2
)
```

Direct deps listed cleanly; indirect deps (the transitive closure) are
under a second `require ( // indirect )` block that `go mod tidy`
maintains.

#### Lab

1. `cd repos/movies-bartr/src && go mod graph | head`. See the full
   dependency tree.
2. `go list -m all | wc -l` — total dep count. Compare to a Node
   project of similar scope (typically 10-100×).
3. `go mod tidy`. Should be a no-op on a clean repo.
4. Add an import you don't use: `import _ "github.com/some/pkg"`.
   Run `go build`. Build fails (unused dep not yet in go.mod). Add
   it via `go get github.com/some/pkg`. Build succeeds. Run `go mod
   tidy`. Note that the dep is removed (because you didn't *use*
   it). The exact lifecycle.
5. Run `go list -m -u all` — shows available upgrades.

#### Knowledge check

1. Why are both `go.mod` and `go.sum` committed?
2. Go's "minimum version selection" picks the lowest version that
   satisfies all constraints. npm picks the highest. What's the
   trade-off, and which one bites you on transitive breakage?
3. The `go` directive (e.g. `go 1.22`) — minimum or maximum?
4. When *would* you vendor? Give one defensible case.
5. `go mod tidy` is idempotent. What scenario produces a diff every
   time you run it, and how do you fix it?

---

### Module 12 — Concurrency: goroutines, channels, when not to (inventory M15–M16)

> **Movies-spec barely needs concurrency primitives.** The handlers
> are already concurrent (each request runs in a goroutine — the
> HTTP server does that for you). This module is the floor for
> recognizing when an *added* goroutine is correct.

#### Concept

Three building blocks:

- **`go fn()`** — start a goroutine. Free. Cheap. The runtime
  multiplexes goroutines onto OS threads.
- **Channels (`chan T`)** — typed pipes between goroutines.
  Unbuffered = synchronous; buffered = up-to-N slots.
- **`sync.Mutex`, `sync.RWMutex`, `sync.WaitGroup`, `atomic.*`** —
  the lower-level primitives for shared-state access.

The slogan "do not communicate by sharing memory; share memory by
communicating" pushes you toward channels — but `sync.Mutex` and
`atomic` are correct, fast, and idiomatic for many real cases.
movies-bartr uses both: `atomic.Bool` and `atomic.Pointer[Store]` for
the `ready` flag and store reference (no contention, single writer),
no channels at all in the main code path.

The race detector (`go test -race`, `go run -race`) instruments your
code to catch data races at runtime. ~2-20× slower; worth running in
CI on at least one job.

**When *not* to reach for concurrency:**

- "Speeding up" a sub-millisecond handler with goroutines — overhead
  dwarfs the work.
- Per-request worker pools when the HTTP server already gives you one.
- `sync.Mutex` around something that should be immutable.

#### Example

The dataset-load goroutine from `cmd/movies-api/main.go`:

```go
var ready atomic.Bool
var storeRef atomic.Pointer[store.Store]
go func() {
    s, err := store.Load(cfg.DataDir)
    if err != nil {
        logger.Error("dataset load failed", slog.String("err", err.Error()))
        return
    }
    storeRef.Store(s)
    ready.Store(true)
}()
```

The `/healthz` endpoint returns 200 immediately. `/readyz` reads
`ready.Load()` — false until the load goroutine finishes. The handlers
read `storeRef.Load()` — nil until ready, return 503 problem+json
until then. **No mutex, no channel, no race.**

This is the right shape because there's exactly one writer (the load
goroutine) and N readers (every request). `atomic.Pointer[T]` is
designed for this case.

#### Lab

1. Read `cmd/movies-api/main.go`'s startup. Trace how `/readyz`
   transitions from 503 to 200.
2. Add a deliberate race: in a scratch file, increment a plain `int`
   from two goroutines without synchronization. Run with `go run -race`.
   Read the race report.
3. Fix it with `atomic.AddInt64`. Re-run. Race gone.
4. In a separate scratch program, spawn 1M goroutines that each
   `time.Sleep(10*time.Second)`. Measure memory. Note that Go
   goroutines are cheap (start at ~2KB stacks), but a million is
   still a million.

#### Knowledge check

1. Why does movies-bartr use `atomic.Pointer[Store]` instead of a
   `sync.RWMutex` around a `*Store` field?
2. When is `sync.Mutex` the right primitive vs `chan`?
3. The race detector adds 2-20× runtime cost. Should it run in
   every CI job, one CI job, or only locally?
4. Channels: unbuffered vs buffered — what behavior does each give
   you on send?
5. The HTTP server already runs each request in its own goroutine.
   When (if ever) do you need to spawn *more* goroutines inside a
   handler?

---

## Per-release review

Same template as the observability guide; see the
[Per-release review template section](study-guide-observability.md#per-release-review-template).
Go-specific addition: when a release touches Module 3 (context) or
Module 4 (shutdown), the hands-on check **must** include a
deliberate-`kubectl delete pod` test with concurrent `curl` traffic.
"It compiles" is a much weaker signal here than "it drains cleanly
under signal."

## Translation notes (per inventory domain M17)

This guide is the canonical-language guide for the curriculum. Every
other study guide references its modules. If you're working in a
non-Go stack, the equivalent primitives:

| Concept (Go module) | Rust + axum | Node + fastify | .NET minimal API | Python + FastAPI |
|---|---|---|---|---|
| `context.Context` (M3) | `tower::Service` extensions + `tokio::time::timeout` | `AbortController` / `signal` | `CancellationToken` | `asyncio.CancelledError` |
| Graceful shutdown (M4) | `tokio::signal` + `axum::Server::with_graceful_shutdown` | `app.close()` on `SIGTERM` | `IHostApplicationLifetime.ApplicationStopping` | `lifespan` events |
| `log/slog` (M6) | `tracing` + `tracing-subscriber` JSON | `pino` | `Microsoft.Extensions.Logging` + JSON formatter | `structlog` |
| Prom client (M7) | `prometheus` crate | `prom-client` | `prometheus-net` | `prometheus_client` |
| Table-driven tests (M8) | `rstest` or hand-rolled | `tap` / `vitest` `describe.each` | `[Theory]` + `[InlineData]` | `pytest.mark.parametrize` |
| Static lint (M9) | `cargo clippy` | `eslint` + `@typescript-eslint` | `dotnet format` + analyzers | `ruff` |
| Tiny binary (M10) | `cargo build --release` + `strip` | n/a (Node ships a runtime) | `dotnet publish -p:PublishTrimmed=true` | n/a (Python ships a runtime) |

The "translate this to my stack" prompt for the agent:

> *"Translate Module N's example from
> [study-guide-go.md](study-guide-go.md) to <stack>. Keep the spec
> contract identical. Don't introduce a framework I haven't named."*

## What this guide is and is not

- **Is:** the slice of Go that maps onto movies-spec plus what the
  other guides assume.
- **Is not:** a Go language tutorial. If the operator doesn't know
  what `:=` does, start with the Go Tour, then come back.
- **Is not:** a microservices-patterns book. Goroutine pools, sagas,
  CQRS, event sourcing — out of scope.
- **Is not:** an opinion on Go vs Rust / Node / .NET / Python. That
  argument is made (and decided for the curriculum) in
  [skills-inventory.md](skills-inventory.md) domain M.

## Open questions

- Module 12 (concurrency) is right at the line of "the spec doesn't
  need it." Worth keeping for floor-knowledge, or trim?
- Module 9 (toolchain) overlaps with the dev-loop guide (domain H).
  Cleaner cut: `gofmt`/`go vet`/`staticcheck` stays here because
  it's *language*; `make`/`git`/`jq` moves to the dev-loop guide
  when written.
- The translation table is the canonical home for the
  cross-language mapping. Does it belong here, or as its own
  doc that every guide links into? Current call: here, until it
  outgrows a single table.

## Status

- Not yet run end-to-end with any operator.
- Anchored in real movies-bartr code; every example cites a file or
  function that exists today.
- Unlocks for promotion to `methodology/` once at least one full
  run has used it and reported honestly. Note the Go-confound
  named in
  [skills-inventory.md](skills-inventory.md): standardizing the
  curriculum on Go does not let us strip "is Go just easy" from
  "did the methodology generalize" until a non-Go participant has
  run.
