# Study Guide — Linux + HTTP Floor (domain K, reduced form)

> **DRAFT — NOT FOR PUBLICATION.** Twelfth and final
> instance of the study-guide format, in **reduced form** —
> a checklist + labs + reading per module, not a full
> concept-and-example exposition. Scoped from domain K of
> [skills-inventory.md](skills-inventory.md).

## Why this exists — and why it's reduced

Every other guide in the curriculum assumes the operator
has Linux + HTTP fundamentals. This guide doesn't *teach*
those fundamentals from scratch — there are excellent
books for that (TLPI, *High Performance Browser Networking*,
*HTTP: The Definitive Guide*). It does three things the
books don't:

1. **Names the specific Linux + HTTP topics the rest of
   this curriculum hits.** Most "things that bite at scale"
   live in 6 narrow areas; this guide names them.
2. **Gives a concrete self-assessment lab for each.** If
   you can do the lab in 10 minutes without looking
   anything up, you have the floor for that topic. If you
   can't, the lab tells you what to read.
3. **Lists the canonical reading** for each gap, so the
   operator can close it deliberately rather than
   stumbling.

**Reduced form means:** each module is a single section,
not Concept / Example / Lab / Knowledge-check. The lab
is the assessment; the reading is the remediation.

**What this guide is anchored in:**

- movies-spec §6 (HTTP semantics), §8.1 (probes +
  graceful shutdown), §9 (containers), §10 (testing).
- movies-bartr's Dockerfile + Deployment as the worked
  example for each topic.
- [study-guide-go.md](study-guide-go.md) Module 4
  (graceful shutdown) and Module 7 (HTTP server +
  metrics) as the curriculum's existing depth on the
  application side.
- [study-guide-k8s-core.md](study-guide-k8s-core.md)
  Module 6 (probes) and Module 7 (resources/QoS) for
  the kubelet's side of the same conversation.

## How to use this guide

Read one module. Run the lab. If it goes cleanly: you
have the floor for that topic; move on. If it doesn't:
the lab tells you what's broken; the reading list closes
the gap. Total time for a passing operator: ~90 minutes
to confirm all six.

## Modules

### Module 1 — Process model + signals + graceful shutdown (K1)

**Why it matters in this curriculum:** Kubernetes sends
`SIGTERM` to your process and waits up to
`terminationGracePeriodSeconds` (default 30s) before
sending `SIGKILL`. If your process doesn't handle SIGTERM,
in-flight requests die mid-flight on every rollout. This
is **the cause of 90% of "why did my pod die uncleanly"
tickets** — every spec-floor service handles it correctly.

**The minimum to know:**

- `SIGTERM` vs `SIGKILL`: TERM is catchable, KILL is not.
- The shutdown sequence Kubernetes uses: kubelet sends
  SIGTERM → grace period → SIGKILL.
- Why a process needs to (a) stop accepting new
  connections, (b) drain in-flight requests, (c) close
  external resources (DB, files), in that order.
- Why setting `terminationGracePeriodSeconds` longer than
  your slowest request's natural duration is the default,
  not the exception.
- What `PID 1` means in a container, why `ENTRYPOINT
  ["binary"]` (exec form) is right and `ENTRYPOINT
  "binary"` (shell form) is wrong — only PID 1 receives
  SIGTERM from the kubelet; a shell wrapper swallows it.

**Lab (10 min):**

1. In movies-bartr, find the SIGTERM handler in
   `src/cmd/movies-api/main.go`. Read it. Confirm it
   stops the HTTP server with a graceful-shutdown timeout
   matching `terminationGracePeriodSeconds`.
2. `kubectl exec` into a movies-api pod and send the
   process SIGTERM by hand: `kill -TERM 1`. Watch the
   pod logs. Confirm the process logs "shutting down" and
   exits cleanly.
3. Now make a bad version: edit the deployment's command
   to `["/bin/sh", "-c", "/movies-api"]` (shell form).
   Re-deploy. Repeat the SIGTERM test. Confirm the shell
   wrapper swallows the signal and the pod gets SIGKILL'd
   after the grace period.
4. Revert.

**If the lab is hard, read:**

- TLPI (Kerrisk), Chapters 20–22 (signals, process
  termination).
- Kubernetes docs: "Termination of Pods" —
  https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination
- [study-guide-go.md](study-guide-go.md) Module 4
  (graceful shutdown) for the application-side pattern.

---

### Module 2 — File descriptors, ulimit, the things that bite at high RPS (K2)

**Why it matters in this curriculum:** A Go HTTP server
holds one file descriptor per concurrent connection. The
default per-process FD limit on most distros is 1024. At
2000 concurrent connections (not unusual for a service at
500 RPS with keepalive), the server starts returning
`too many open files` — and the failure mode looks like
a 500/503 spike at exactly the wrong load level. The
spec's §10.4 target (500 RPS) doesn't naturally hit this
on a single pod, but **any time you're benchmarking
above that, you need to know.**

**The minimum to know:**

- What a file descriptor is and why a TCP socket is one.
- `ulimit -n` and where it's set on a Linux box vs in a
  container vs in a Kubernetes Pod (it's not the same
  place).
- Why containers usually inherit a sensible FD limit
  from the container runtime but Kubernetes can override
  it.
- `/proc/<pid>/limits` is the truth — that's what the
  process actually sees, regardless of what `ulimit`
  says in the shell that started it.
- The other "bites at scale" things in the same family:
  `net.core.somaxconn` (the listen-backlog cap),
  `net.ipv4.tcp_max_syn_backlog`, ephemeral port
  exhaustion on the *client* side.

**Lab (10 min):**

1. In a movies-bartr pod, run `cat /proc/1/limits`. Find
   the "Max open files" line. Note the soft + hard
   limits.
2. Run `ulimit -n` in the same pod. Compare to (1). If
   they differ, understand why (the shell sets its own;
   the process inherits from the runtime).
3. Find `net.core.somaxconn` on the host: `sysctl
   net.core.somaxconn`. If it's the kernel default
   (4096 on modern Linux; 128 on older systems), note
   it. A high-RPS service may need it raised — but
   that's a host setting, not a Pod setting.
4. (Optional, if you have time.) Run `vegeta` at 5000
   RPS against movies-api. Watch `/proc/<pid>/fd` count
   climb. Confirm it stays well under the FD limit.

**If the lab is hard, read:**

- TLPI Chapter 36 (resource limits).
- "Tuning Linux for high RPS" — the Cloudflare blog
  series is the canonical practitioner reference.
- `man 2 setrlimit` for the syscall behind `ulimit`.

---

### Module 3 — DNS inside a pod (K3)

**Why it matters in this curriculum:** Every in-cluster
service call goes through CoreDNS, and every "why is my
service flapping" investigation eventually arrives at
DNS. The default `/etc/resolv.conf` inside a pod has
search-path behavior that surprises people: a query for
`movies-api` (no dots) gets *5 lookups* before resolving,
because of the search list. Move to fully-qualified
names (`movies-api.movies.svc.cluster.local.` — note the
trailing dot) and you skip the search.

**The minimum to know:**

- How a pod's `/etc/resolv.conf` is constructed by the
  kubelet — `nameserver` is CoreDNS's cluster IP,
  `search` is the namespace search list, `options
  ndots:5` is the surprise.
- The four DNS forms for an in-cluster Service:
  `<svc>` (works only in the same namespace),
  `<svc>.<ns>` (works across namespaces),
  `<svc>.<ns>.svc` (full short form),
  `<svc>.<ns>.svc.cluster.local.` (FQDN, skips
  search). The trailing dot matters at scale.
- Headless Services (`clusterIP: None`) return one A
  record per backend pod — used for stateful
  workloads (Postgres, Kafka) that need direct pod
  addressing.
- ExternalName Services map an in-cluster name to a
  CNAME outside the cluster.
- `ndots: 5` interacts with `search` to multiply
  failed lookups. The classic "DNS is slow in our
  cluster" trace ends with "your code is doing 5
  lookups per request because the name has fewer than
  5 dots."

**Lab (10 min):**

1. In a movies-bartr pod, `cat /etc/resolv.conf`.
   Confirm `nameserver`, `search`, `ndots: 5`.
2. From the same pod, run `nslookup movies-api`. Count
   queries: `time nslookup movies-api`. Compare to
   `time nslookup movies-api.movies.svc.cluster.local.`
   (with the trailing dot). Same answer, fewer
   queries.
3. Find CoreDNS in the cluster: `kubectl get pods -n
   kube-system -l k8s-app=kube-dns`. Tail its logs
   while you run a fresh resolution. Confirm you see
   the query land at CoreDNS.
4. Delete CoreDNS pods (`kubectl delete pod -n
   kube-system -l k8s-app=kube-dns`). Confirm a
   restart happens within seconds. Notice: in-cluster
   DNS is briefly broken; existing TCP connections
   keep working but new ones fail. This is why CoreDNS
   runs ≥ 2 replicas in production.

**If the lab is hard, read:**

- Kubernetes docs: "DNS for Services and Pods" —
  https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/
- CoreDNS docs on its plugin chain — particularly the
  `kubernetes` and `forward` plugins.
- "DNS lookups in Kubernetes" — the Datadog and Lyft
  engineering blogs both have widely-cited write-ups
  of the `ndots: 5` trap.

---

### Module 4 — HTTP status codes + idempotency (K4)

**Why it matters in this curriculum:** Spec §6 fixes the
status code semantics for movies-api: 200 / 400 / 404 /
405 / 500. The trap is **4xx-vs-5xx confusion** —
returning 500 for "the client sent garbage" is
diagnostic noise that wastes operator attention. Every
4xx is "the client did something wrong, no point
retrying without a change." Every 5xx is "I did
something wrong, retry might help." Getting this right
matters for retries, alerting, log signal, and downstream
caching.

**The minimum to know:**

- The five categories: 1xx informational, 2xx success,
  3xx redirect, 4xx client-fault, 5xx server-fault.
  The hinge is "whose fault."
- The status codes a curriculum-spec API actually
  uses: 200, 204, 301/308, 400, 401, 403, 404, 405,
  409, 422, 429, 500, 502, 503, 504. Know each one's
  meaning.
- **Idempotency.** GET / PUT / DELETE are idempotent
  (calling N times has the same effect as calling
  once). POST and PATCH are not. This is the
  controlling property for whether a retry is safe.
- RFC 7807 `application/problem+json` — the standard
  envelope for 4xx/5xx responses. Spec §6 mandates it.
  movies-bartr's `test.yaml` validates that 400s
  return `application/problem+json` content type.
- **Where the codes actually fire in a real stack.**
  4xx returned by *your* code = your validation. 4xx
  returned by Traefik = the controller (rate limit,
  IP block, auth). 5xx returned by *your* code =
  bug. 5xx returned by Traefik = backend unreachable.
  `502 Bad Gateway` always means "the controller
  can't reach the backend pod"; `503 Service
  Unavailable` from Traefik usually means
  "no backend pods are Ready." Reading the source
  matters: a 502 in the Traefik log is a *cluster*
  problem; a 502 in your app log is a *downstream*
  problem (a service you call timing out, etc.).

**Lab (10 min):**

1. In movies-bartr, find every status code your
   handlers return. `grep -rn "WriteHeader\|.Status("
   src/`. Map each to one of the categories.
2. Run `test.yaml` against the cluster. Confirm every
   negative-path request returns the spec'd 4xx with
   `application/problem+json`.
3. Stop the movies-api Deployment (`kubectl scale
   deploy/movies-api --replicas=0`). Curl the LB
   immediately. Predict the exact code Traefik
   returns. Confirm. (Hint: it's 503, not 502,
   because there are *no* backends — the difference
   matters.)
4. Set replicas back to 1; scale to 5 immediately
   while curl-ing. Watch for 502s during the
   transition (some backends drained, others not yet
   Ready). That's the difference between "service
   unavailable" and "specific backend unreachable."

**If the lab is hard, read:**

- RFC 9110 (HTTP Semantics, 2022) — the modern
  canonical text. Pay attention to §15 (status codes)
  and §9.2.2 (idempotency).
- RFC 7807 (Problem Details for HTTP APIs).
- Mozilla's HTTP docs are a faster lookup if you're
  not sure which code to use:
  https://developer.mozilla.org/en-US/docs/Web/HTTP/Status

---

### Module 5 — HTTP keep-alive, connection reuse, why benchmarks lie (K5)

**Why it matters in this curriculum:** The single most
common "benchmark looks fast in dev but bad in prod"
trace ends with "the load generator wasn't using
keep-alive, so every request paid a TCP handshake +
TLS handshake." A handshake is 1 RTT minimum (for TCP;
TLS adds another 1–2 RTTs without 0-RTT). On a local
network that's 0.5 ms; cross-region it's 50 ms — and
suddenly your "p95 = 50 ms" is *all handshake*. The
real service responds in 1 ms once a connection is
warm. Without keep-alive you are benchmarking the
network, not the service.

**The minimum to know:**

- HTTP/1.1's `Connection: keep-alive` is the default
  in modern HTTP; the *server* can close
  (`Connection: close`) but rarely should. Reuse a
  connection for many requests = pay the handshake
  once.
- HTTP/2 multiplexes streams over a single
  connection — keep-alive is implicit and per-stream
  cost is near zero. This is why HTTP/2 collapsed the
  "concurrent connection limit" problem.
- HTTP/3 (QUIC) is HTTP/2's semantics over UDP with
  built-in TLS 1.3 and 0-RTT resumption — solves
  head-of-line blocking at the transport layer. Not
  always supported by ingress controllers; check
  before you bet on it.
- Connection pooling on the client side. Go's
  `http.Client` reuses connections by default *if*
  you read the response body to EOF and `Close()`
  it. Leaking a response body = leaking a
  connection = serial handshakes on the next
  requests.
- `vegeta` and `webv` reuse connections by default.
  `curl` reuses across `--next` requests in one
  invocation; one `curl` per request is one
  handshake per request — the slow way to benchmark.
- The "warmup" step in performance methodology
  (testing guide Module 8): run the load for 10s
  before recording, so connection pools are warm and
  TCP windows have ramped.

**Lab (10 min):**

1. Curl movies-bartr's LB twice with `-v --next`:
   ```sh
   $ curl -v --next http://localhost/api/movies --next http://localhost/api/actors 2>&1 | grep -E "Connected|Re-using|Connection #"
   ```
   Confirm "Re-using existing connection" on the
   second.
2. Now run two separate curls. Confirm two separate
   TCP connects.
3. Run `vegeta` at 500 RPS for 60s twice: once with
   `-keepalive=true` (default), once with
   `-keepalive=false`. Compare p50, p95, p99. The
   second run will show meaningfully higher latency
   *and* higher variance — most of which is
   handshake cost.
4. (Optional, advanced.) Use `tcpdump -i any -nn
   'host <pod-ip> and tcp'` on a node and watch
   connection establishment during each scenario.
   The keepalive=true run shows one SYN per worker
   thread; keepalive=false shows one per request.

**If the lab is hard, read:**

- *High Performance Browser Networking* (Grigorik) —
  Chapters 1–4 cover TCP, TLS, HTTP/1.1, HTTP/2 with
  the right level of network-level detail. Free
  online at https://hpbn.co/.
- RFC 9110 §9 (HTTP connection management).
- Go's `net/http` Transport docs on connection
  pooling.

---

### Module 6 — TLS basics: handshake, certs, SNI (K6)

**Why it matters in this curriculum:** Module 5 of the
ingress guide covers TLS *operationally* (cert-manager,
Let's Encrypt, the staging-then-prod issuer flip). This
module covers the floor underneath: what's actually
happening on the wire when "TLS terminates at the
ingress." Without it, debugging certificate problems is
guesswork.

**The minimum to know:**

- The TLS 1.3 handshake: ClientHello (with SNI →
  ServerHello (with cert) → key exchange → encrypted
  application data. Minimum 1 RTT; 0-RTT possible on
  resumption. TLS 1.2 is 2 RTTs and still common on
  older clients.
- **SNI (Server Name Indication).** The client sends
  the hostname *in the clear* in ClientHello so the
  server (or LB) can pick the right cert for that
  hostname. Without SNI, one IP can serve only one
  HTTPS cert. With SNI, one IP can serve hundreds.
- The chain of trust: leaf cert → intermediate(s) →
  root CA. The root CA is in your client's trust
  store (OS, browser, Go's `crypto/x509`). Missing
  intermediates = "unknown certificate authority"
  even though the leaf is valid.
- **mTLS (mutual TLS).** The *server* also verifies
  the *client's* cert. Used for service-mesh
  pod-to-pod, partner-to-partner B2B, and zero-trust
  network access (ZTNA). Not the default for public
  web traffic.
- The certificate fields that matter: Subject
  Alternative Name (SAN) is the modern "what
  hostnames does this cert cover" field; Common Name
  (CN) is deprecated for hostname matching since
  2017 (modern browsers ignore it).
- Cert expiry: cert-manager renews automatically
  (see ingress M5); your job is to know that **a
  cert expiring at 02:00 UTC on a Sunday is the
  number-one cause of "the site is down" outages**
  in environments without auto-renewal.

**Lab (10 min):**

1. Inspect a public TLS cert from your laptop:
   ```sh
   $ echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null \
       | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
   ```
   Identify the subject, issuer, validity dates, SAN
   list.
2. Drop the `-servername` flag. Some sites return a
   different (default) cert; some refuse. That's
   SNI in action.
3. Curl a site whose cert has a missing intermediate
   (one good test: `https://incomplete-chain.badssl.com/`):
   ```sh
   $ curl -v https://incomplete-chain.badssl.com/ 2>&1 | grep -iE "certificate|verify"
   ```
   Confirm `curl` fails. Run with `--insecure` —
   succeeds. That's the difference between "cert is
   wrong" and "cert is missing intermediates."
4. If movies-bartr has a TLS dev cert: `openssl
   x509 -in /path/to/cert.pem -noout -text` and read
   it top to bottom. If not, generate a self-signed
   cert with `openssl req -x509 -newkey ed25519
   -nodes -days 30 -out cert.pem -keyout key.pem` and
   read that. The point is: be comfortable looking
   at a cert as a structured document, not a black
   box.

**If the lab is hard, read:**

- *Bulletproof TLS and PKI* (Ristić) — the canonical
  practitioner reference. Long; chapters 1–3 are
  the floor.
- RFC 8446 (TLS 1.3). Dense, but the handshake
  diagrams are the clearest in any spec.
- Cloudflare's "How TLS works" blog series — most
  accessible practitioner intro.

---

## Per-release review

This guide is the **only one in the curriculum without
a per-release residual**, and that's deliberate. It's a
floor, not a process — the per-release rituals come from
the other guides (cold-cluster-read, security capstone,
image-size budget, inner-loop signal check, baseline-
still-signaling, entrypoint-pin audit). The floor either
holds or it doesn't; if it doesn't, the right move is
training, not a per-release check.

The closest thing to a residual: **on every new operator
joining the project, walk all six labs.** 90 minutes
the first week saves hours the rest of the year.

## What this guide is

- A self-assessment checklist for the Linux + HTTP
  topics the rest of this curriculum hits.
- A pointer-page to the canonical reading for each
  topic — TLPI, *High Performance Browser Networking*,
  RFC 9110, RFC 8446, *Bulletproof TLS and PKI*.
- The reduced-form template for "this domain doesn't
  need a full study guide, but does need to be named."

## What this guide is not

- Not a Linux administration text — TLPI exists.
- Not a complete HTTP reference — RFC 9110 exists.
- Not a TLS deep-dive — Ristić exists.
- Not a substitute for hands-on time on a real Linux
  box. The labs are confirmation, not instruction.

## Open questions

1. Is Module 2 (FD limits + sockets) too narrow? It
   covers what bites at the curriculum's RPS targets,
   but says little about NUMA, hugepages, or block-I/O
   tuning — topics that bite at much higher scales.
   Add an "advanced" section, or punt to a separate
   guide if/when the curriculum ever needs to address
   100k-RPS workloads?
2. Module 6 (TLS) skirts post-quantum cryptography
   entirely. Hybrid PQ key-exchange is being rolled
   out in TLS 1.3 stacks across 2024–2026. Does the
   curriculum need to address it, or wait for a real
   workload to force the question?
3. Should the curriculum standardize on Linux only,
   or address the WSL2 / macOS-via-Linux-VM cases
   that the local-platform guide already documents?
   This guide currently assumes "Linux" without
   qualification.

## Status

DRAFT. Reduced-form guides have a lower validation bar
than full guides — promotion criterion is: one operator
runs all six labs end-to-end and confirms the time
estimate (90 min total to pass all six) is realistic,
and that the reading-list remediations actually close
the gaps the labs surface. Failure mode to watch for:
labs that "pass" without the underlying floor actually
being there — a lab is only good if its failure mode
points at the right reading.
