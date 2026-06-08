# Study Guide — Security (movies-spec §8.1 + §13, domain J)

> **DRAFT — NOT FOR PUBLICATION.** Fifth instance of the study-guide
> format. Scoped from domain J of
> [skills-inventory.md](skills-inventory.md). Anchored in the
> movies-bartr security baseline (`deploy/movies/base/deployment.yaml`,
> `deploy/movies/base/networkpolicy.yaml`) and the secrets module of
> [study-guide-gitops.md](study-guide-gitops.md).

## Why this exists

Security is the domain where the agent's defaults are most likely to
look correct and *not be*. A `securityContext:` block with the right
fields present passes a code review. The same block with one wrong
value (`runAsNonRoot: false`, missing `drop: ALL`, a wrong
namespace selector in a NetworkPolicy) ships an exploitable container
without ever failing a build.

This guide closes that gap. It does not teach offensive security; it
teaches the operator to *read* a security baseline and know whether
it's the real thing or theater.

**What this guide is anchored in:**

- Spec §8.1 — `runAsNonRoot`, `readOnlyRootFilesystem`, dropped caps,
  no privilege escalation.
- Spec §13 — non-root, no caps, no PE, NetworkPolicy ingress
  restrictions, no secrets in repos / images / ConfigMaps.
- movies-bartr `deploy/movies/base/deployment.yaml` — the real,
  spec-compliant `securityContext` block.
- movies-bartr `deploy/movies/base/networkpolicy.yaml` — the
  default-deny + narrow-allow pattern.
- [study-guide-gitops.md](study-guide-gitops.md) Module 6 — secrets
  in GitOps (ESO / SOPS / Sealed Secrets). Referenced here, not
  re-explained.

## How to use this guide

Same protocol as the other study guides. One curriculum-level rule
specific to this one: **every lab includes a "break it and see what
fails" step.** Security baselines that have never been tested
end-to-end are theater. The point of the labs is to confirm the
controls actually do what the YAML claims.

## Modules

### Module 1 — Pod-level securityContext (spec §8.1, §13)

#### Concept

`securityContext` exists at two levels: **pod** (applies to every
container) and **container** (overrides pod-level for one container).
The spec requires both to be set explicitly; the agent's default is
often only one or the other.

The seven controls the spec cares about:

1. **`runAsNonRoot: true`** — the kubelet refuses to start the pod if
   the image's `USER` is root. Belt-and-suspenders on top of the
   Dockerfile's `USER 1000` directive.
2. **`runAsUser: 1000` / `runAsGroup: 1000`** — explicit UID/GID. If
   the image baked a different user, this overrides it (and may break
   the image — coordinate with the Dockerfile).
3. **`fsGroup: 1000`** — pod-level. Files on mounted volumes are
   chowned to this group on mount.
4. **`allowPrivilegeEscalation: false`** — disables `setuid` /
   `setgid` / capabilities-add via syscall. *Container-level only.*
5. **`readOnlyRootFilesystem: true`** — the container's `/` is
   mounted read-only. Anything that wants to write needs an explicit
   `emptyDir` mount (`/tmp` is the common one).
6. **`capabilities.drop: [ALL]`** — start from zero Linux
   capabilities, add back only what the workload needs. For a
   GET-only HTTP API: nothing.
7. **`seccompProfile.type: RuntimeDefault`** — applies the
   runtime's default seccomp profile (containerd's default blocks
   ~64 syscalls). Free hardening; no reason not to set it.

Also relevant, often missed:

- **`automountServiceAccountToken: false`** at the pod spec level.
  A workload that doesn't talk to the Kubernetes API doesn't need
  a token; mounting one by default is a credential the attacker
  doesn't need to be handed.

#### Example

From `deploy/movies/base/deployment.yaml`, the real movies-bartr
spec-compliant block:

```yaml
spec:
  template:
    spec:
      automountServiceAccountToken: false       # don't hand out a token
      securityContext:                          # pod-level
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: movies-api
          image: movies-api:1.0.0
          securityContext:                      # container-level
            runAsNonRoot: true
            runAsUser: 1000
            runAsGroup: 1000
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            seccompProfile:
              type: RuntimeDefault
            capabilities:
              drop:
                - ALL
          volumeMounts:
            - name: tmp
              mountPath: /tmp                   # writable scratch
      volumes:
        - name: tmp
          emptyDir: {}
```

Notice the pattern: the pod-level block sets defaults, the
container-level block re-states the critical ones (defense in depth),
and an `emptyDir` mount on `/tmp` exists *only* because
`readOnlyRootFilesystem: true` would otherwise break anything that
writes scratch data.

#### Lab

In `repos/movies-bartr/`:

1. Read `deploy/movies/base/deployment.yaml` in full. For each of the
   seven controls above, find it. Note where pod and container
   blocks duplicate (intentional defense in depth) vs where they
   differ.
2. **Break and observe (1).** Change `runAsNonRoot` to `false` at
   the pod level. Apply. Does the pod start? Why?
3. **Break and observe (2).** Set `readOnlyRootFilesystem: false`,
   remove the `/tmp` emptyDir. Apply. Does the pod start? Does the
   workload behave the same? (The Go binary writes very little; the
   answer might surprise you.)
4. **Break and observe (3).** Remove `capabilities.drop: [ALL]`.
   Apply. The pod starts and the service works *exactly the same*.
   The control is invisible until it matters. **This is what
   "security theater" looks like when it's *not* theater — the
   absence of an exploit you can't see.**
5. Restore.
6. **Recognize the agent's typical mistakes.** Ask Claude to
   "generate a Deployment YAML for an HTTP service." Read the
   generated `securityContext`. Score it against the seven controls.
   Where did it cut corners?

#### Knowledge check

1. Why is `securityContext` at both pod and container levels?
2. `runAsNonRoot: true` in K8s vs `USER 1000` in the Dockerfile —
   which is the real control? What does each one catch that the
   other doesn't?
3. `readOnlyRootFilesystem: true` requires an `emptyDir` for any
   scratch directory. What breaks if you forget the `emptyDir`? How
   would you discover the breakage?
4. `capabilities.drop: [ALL]` is invisible to functional testing.
   What's the methodology that catches the *absence* of this control
   before it bites you?
5. `automountServiceAccountToken: false` — what attack does it
   foreclose? When would you set it `true` deliberately?

---

### Module 2 — NetworkPolicy: default-deny + narrow-allow (spec §13)

> **The control most teams set once and never test.** Easy to write
> a NetworkPolicy that *looks* restrictive and isn't. This module
> teaches the pattern that actually works and the test that proves
> it.

#### Concept

By default, every pod in a Kubernetes cluster can reach every other
pod. `NetworkPolicy` is the lever that changes that — but only if
the cluster's CNI implements it (most do; some default-disable it).

The pattern that works:

1. **Default-deny.** A NetworkPolicy with `podSelector: {}` (matches
   every pod in the namespace) and `policyTypes: [Ingress, Egress]`
   with no `ingress:` or `egress:` rules. Result: everything is
   denied unless another policy explicitly allows it.
2. **Narrow-allow.** A second NetworkPolicy that matches the
   workload by label and allows *exactly* the traffic it needs.

The narrow-allow policy must specify both peer (`from:` / `to:`) and
port. Common selectors:

- **`namespaceSelector` with `kubernetes.io/metadata.name`** — the
  kube-apiserver sets this label automatically on every Namespace,
  so it's portable.
- **`podSelector`** — match peers by label inside the same namespace.
- **`ipBlock`** — match by CIDR. Use sparingly; pod IPs change.

For an HTTP service inside a cluster, the egress list is usually
just *DNS* (UDP+TCP 53 to kube-system's CoreDNS). The ingress list
names every legitimate caller: the ingress controller, the metrics
scraper, any in-namespace load generator.

**The failure mode that bites everyone:** writing the policy with
the wrong namespace label, the wrong port, or the wrong protocol —
and never testing the *deny* path. The policy applies, traffic
flows, and you assume the policy is working. It's not; the policy
just happens to be too loose.

#### Example

From `deploy/movies/base/networkpolicy.yaml` — the real, working,
two-policy pattern:

```yaml
# Policy 1 — default-deny across the namespace.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: movies
spec:
  podSelector: {}                # every pod in the namespace
  policyTypes:
    - Ingress
    - Egress
---
# Policy 2 — narrow-allow for movies-api.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: movies-api
  namespace: movies
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: movies-api
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Traefik (lives in kube-system on k3s).
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - { protocol: TCP, port: 8080 }
    # Prometheus operator scrape.
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - { protocol: TCP, port: 8080 }
    # In-namespace load generator.
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: webv
      ports:
        - { protocol: TCP, port: 8080 }
  egress:
    # DNS only — the service makes no other outbound calls.
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - { protocol: UDP, port: 53 }
        - { protocol: TCP, port: 53 }
```

Three real allowed ingress sources, one real allowed egress
destination. Everything else is denied.

#### Lab

1. Read both policies in `networkpolicy.yaml`. For each ingress
   rule, identify *which real caller* it allows.
2. **Prove default-deny works.** Spin up a debug pod
   (`kubectl run debug --image=nicolaka/netshoot --rm -it --restart=Never
   -n movies -- /bin/sh`). From it, try `curl
   http://movies-api:8080/healthz`. The pod has no `app.kubernetes.io/name:
   webv` label; the request should *fail* (timeout, not a 404). If
   it succeeds, your default-deny isn't working.
3. **Prove narrow-allow works.** Re-run the debug pod with the
   `webv` label: `kubectl run debug --image=nicolaka/netshoot
   --labels=app.kubernetes.io/name=webv ...`. Same `curl`. Should
   succeed.
4. **Break and observe (1).** Change the namespace selector for the
   Prometheus rule from `monitoring` to `monitoringg` (typo). Apply.
   Wait for the next Prometheus scrape. Does Prometheus still see
   `movies-api`? Read the symptom: no metrics, no error message in
   the policy itself.
5. **Break and observe (2).** Comment out the egress rule. Apply.
   Does the service still resolve `kube-dns`? What error do you get
   in the logs?
6. Restore everything. Re-run steps 2–3 to confirm the baseline.

#### Knowledge check

1. Why does the pattern use *two* policies instead of one big one?
2. `podSelector: {}` is the load-bearing line in the default-deny
   policy. What does it select?
3. A NetworkPolicy without `ingress:` or `egress:` (but with
   `policyTypes` listing both) — what does that *do*?
4. The Prometheus selector is `kubernetes.io/metadata.name: monitoring`.
   Why is that label specifically the safe choice vs picking your
   own custom label?
5. The egress allows only DNS. What happens to outbound HTTPS calls
   from the service? When would you want to allow them, and how
   would you write the rule?
6. The CNI must implement NetworkPolicy for any of this to work. How
   do you verify your cluster's CNI does, *before* you ship the
   policy?

---

### Module 3 — Image security: distroless, non-root, scanned (spec §9, §13)

> **The Dockerfile half of the security story.** Tied tightly to
> [study-guide-kustomize.md](study-guide-kustomize.md) Module 6
> (canonical Dockerfile) and [study-guide-go.md](study-guide-go.md)
> Module 10 (build flags). This module is the security read on the
> same code.

#### Concept

Three image-side controls the spec requires:

1. **Minimal base.** `distroless/static` (Google) or `alpine` for Go;
   `distroless/cc-debian12` or `chiseled` images for other stacks.
   The principle: an image that doesn't contain a shell, a package
   manager, or libc cannot run an attacker's interactive payload.
2. **Non-root user.** `USER 65532:65532` (distroless's `nonroot`
   UID) or an explicit `USER 1000` for alpine. Pairs with
   `runAsNonRoot: true` (Module 1).
3. **No build tooling in the runtime image.** Multi-stage; the
   builder stage has the Go toolchain, the runtime stage doesn't.

Plus, increasingly table-stakes:

4. **Vulnerability scanning.** `trivy image movies-api:1.0.0`,
   `grype movies-api:1.0.0`, GHCR's built-in scan, Snyk, etc.
   Catches `CVE-2024-1234` in a transitively pulled library.
5. **SBOM generation.** `syft movies-api:1.0.0 -o spdx-json` produces
   a Software Bill of Materials — an inventory of every package in
   the image. Required by many enterprise procurement processes
   and by recent US executive orders.
6. **Image signing.** `cosign sign --keyless ghcr.io/<org>/movies-api:1.0.0`
   produces a signature attesting to who built the image. Verifying
   the signature at admission time blocks unsigned images from
   reaching the cluster.

For the curriculum, controls 1–4 are spec-bar; 5–6 are
enterprise-bar.

#### Example

The canonical Dockerfile from
[study-guide-kustomize.md](study-guide-kustomize.md) Module 6,
re-read with security eyes:

```dockerfile
FROM golang:1.22-alpine AS builder         # toolchain stage
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
ARG VERSION=dev
RUN CGO_ENABLED=0 GOOS=linux go build \
      -trimpath \
      -ldflags="-s -w -X main.version=${VERSION}" \
      -o /out/movies-api ./cmd/movies-api

FROM gcr.io/distroless/static:nonroot      # runtime — no shell, no libc, no nothing
WORKDIR /app
COPY --from=builder /out/movies-api /app/movies-api
USER 65532:65532                            # distroless nonroot uid/gid
EXPOSE 8080
ENTRYPOINT ["/app/movies-api"]
```

Security read:

- `distroless/static:nonroot` — no shell to `kubectl exec` into,
  no package manager to `apk add curl` from inside the container.
- `USER 65532:65532` — the runtime can't write anywhere outside
  what's explicitly mounted as `nonroot`-writable.
- `CGO_ENABLED=0` — no dynamic linking, no glibc surface area.
- `-trimpath` + `-ldflags="-s -w"` — strips reproducibility-breaking
  metadata and debug info from the binary. (Tradeoff covered in
  [study-guide-go.md](study-guide-go.md) Module 10.)

Scan + SBOM:

```bash
trivy image --severity HIGH,CRITICAL movies-api:1.0.0
syft movies-api:1.0.0 -o spdx-json > sbom.spdx.json
```

#### Lab

1. Build movies-bartr's image: `cd repos/movies-bartr/src && docker
   build -t movies-api:lab .`
2. **Prove there's no shell.** `docker run --rm -it movies-api:lab
   /bin/sh`. Read the error. Try `/bin/bash`. Try `/sh`. Try
   `sh`. None work — there's nothing to exec.
3. **Compare to a fat base.** Edit the runtime stage to
   `FROM golang:1.22-alpine`. Rebuild as `movies-api:fat-lab`.
   `docker run --rm -it movies-api:fat-lab /bin/sh` — you're in.
   Type `apk list --installed | wc -l`. Note the attack surface.
4. **Scan both.** `trivy image movies-api:lab` and `trivy image
   movies-api:fat-lab`. Compare CVE counts.
5. **Generate an SBOM.** `syft movies-api:lab -o spdx-json |
   jq '.packages | length'` — count the packages. Compare to the
   fat image.
6. Restore the Dockerfile.

#### Knowledge check

1. Why does `distroless/static` make `kubectl exec` into the
   container effectively useless? What does that buy you?
2. `USER 65532:65532` is the distroless `nonroot` UID. Why use that
   exact number? What happens if the runtime needs to write to a
   mount owned by UID 1000?
3. A CVE is published in a library your image transitively pulls.
   What's the spec-bar response? What's the enterprise response?
4. SBOM generation is enterprise-bar. Name one concrete
   procurement/compliance scenario where it's a hard requirement.
5. Image signing (cosign) catches a class of attacks that scanning
   doesn't. What class?

---

### Module 4 — Secrets at runtime (spec §13, points to GitOps Module 6)

#### Concept

The git-side of secrets is covered in
[study-guide-gitops.md](study-guide-gitops.md) Module 6 (SOPS /
Sealed Secrets / ESO). This module is the **runtime** side: once a
secret exists in the cluster, how does it get into the container
without leaking?

Three injection patterns, in increasing order of leak risk:

1. **`envFrom: - secretRef`** — every key in the Secret becomes an
   environment variable. Leak risk: environment vars are visible
   to anyone with `kubectl exec` *or* anyone who can read
   `/proc/<pid>/environ` from inside the container.
2. **`env: - valueFrom: secretKeyRef`** — same as above but
   per-key. Cleaner because you opt-in to each var.
3. **Projected volume / `volumeMounts`** — the secret is mounted as
   a file at a path. Leak risk: only what you `cat` into a log.
   Best for files (TLS keys, kubeconfigs, API keys with newlines).

Spec §13 calls out: "delivered via a native Kubernetes `Secret`
referenced by `envFrom` / `valueFrom.secretKeyRef` or a projected
volume." All three are spec-compliant; the trade-off is per-secret.

**Anti-patterns the agent will reach for if you don't watch:**

- **`ConfigMap` for "non-sensitive" config that turns out sensitive.**
  ConfigMaps have no encryption-at-rest in etcd by default;
  Secrets do (on most managed K8s distros; opt-in on some). A
  ConfigMap holding a Grafana admin password is a credential.
- **Logging the env vars on startup.** "Effective configuration logged
  once at info level" (spec §11) — *with secret values redacted.*
  Easy to miss. movies-bartr's config uses a `Redacted()` method for
  exactly this.
- **Baking the secret into the image at build time.** `--build-arg`
  with a secret puts it in image layer history forever.

#### Example

`envFrom` from a `Secret` (the simplest pattern):

```yaml
spec:
  template:
    spec:
      containers:
        - name: movies-api
          envFrom:
            - secretRef:
                name: movies-api-secrets
```

`valueFrom` for one key (the per-key pattern):

```yaml
env:
  - name: GRAFANA_ADMIN_PASSWORD
    valueFrom:
      secretKeyRef:
        name: grafana-admin
        key: password
```

Projected volume (the file-mount pattern):

```yaml
volumeMounts:
  - name: tls
    mountPath: /etc/tls
    readOnly: true
volumes:
  - name: tls
    secret:
      secretName: movies-api-tls
      defaultMode: 0400
```

#### Lab

1. Create a scratch Secret: `kubectl create secret generic lab-secret
   --from-literal=password=hunter2 -n movies`.
2. **Try all three injection patterns.** Patch the Deployment to
   inject the secret via `envFrom`, redeploy, `kubectl exec` into
   the pod (oh wait — distroless. Use a debug ephemeral container
   or temporarily switch to alpine). Confirm the env var is set.
3. **Read it from `/proc`.** Inside the pod (or via debug
   container), `cat /proc/1/environ | tr '\0' '\n'`. There's your
   secret in plain text. *This is what `envFrom` looks like to an
   attacker who got code execution.*
4. **Compare to the projected-volume pattern.** Replace `envFrom`
   with a projected volume. Re-run step 3 — the env var is gone.
   The secret lives only in `/etc/tls/password` with mode 0400.
5. **Recognize the agent's mistake.** Ask Claude to "generate a
   Deployment that uses a Grafana admin password from a Secret."
   Read what it generates. Score the choice — `envFrom`,
   `valueFrom`, or projected volume? Why that choice?
6. Clean up the scratch secret.

#### Knowledge check

1. `envFrom` vs `valueFrom` vs projected volume — what's the
   leak-risk ordering and why?
2. ConfigMaps have no encryption-at-rest by default. Why does this
   matter for a "non-sensitive" config that turns out to hold a
   credential?
3. The spec says effective configuration is logged once at startup.
   What discipline keeps this from leaking secrets?
4. `defaultMode: 0400` on a projected volume — what attack does it
   foreclose?
5. The `--build-arg SECRET=...` anti-pattern: why doesn't `docker
   build --secret` (the right answer) appear in most tutorials?

---

### Module 5 — Dependency scanning + supply chain (spec §13)

#### Concept

Three layers of supply-chain hygiene, increasing in scope:

1. **Language-level dependency audit.** `go list -json -m all | nancy`,
   `npm audit`, `pip-audit`, `cargo audit`. Catches CVEs in your
   direct + transitive dependencies. Should run on every commit (or
   at least every PR) and on a schedule (CVEs land continuously
   after your build).
2. **Image scan.** `trivy`, `grype`, `snyk container`. Catches CVEs
   in the OS layer + the dependencies your language audit missed.
   Should run on every image build and on a schedule against the
   registry.
3. **SBOM + signing.** `syft` for SBOM, `cosign` for signing,
   `kyverno` / `sigstore-policy-controller` for admission-time
   verification. The "supply chain" controls in spec §13 sit here.

The methodology piece the spec doesn't spell out:

- **Cadence matters as much as the tool.** A scan that runs once a
  quarter is theater. A scan that runs on every PR + nightly is a
  control.
- **Severity threshold + waiver workflow.** Block on HIGH+CRITICAL,
  warn on MEDIUM, document waivers (with expiration dates) for
  anything you accept.
- **Remediation, not just detection.** Most teams have a scanner;
  fewer have a *who-fixes-what-by-when* workflow. The scanner
  finding a CVE is the start of the work, not the end.

#### Example

A minimal `Makefile` security target:

```makefile
.PHONY: sec sec-deps sec-image sec-sbom

sec: sec-deps sec-image

sec-deps:
	@echo "==> Go dependency audit"
	go list -json -m all | docker run --rm -i sonatypecommunity/nancy sleuth

sec-image:
	@echo "==> Image vulnerability scan"
	trivy image --severity HIGH,CRITICAL --exit-code 1 movies-api:$(VERSION)

sec-sbom:
	@echo "==> Generate SBOM"
	syft movies-api:$(VERSION) -o spdx-json > sbom-$(VERSION).spdx.json
```

A GitHub Actions workflow that runs the above on every PR plus
nightly is the cadence half. The detection-to-remediation half is
the workflow your team uses to triage the findings.

#### Lab

1. Run a dependency audit on movies-bartr: `cd repos/movies-bartr/src
   && go list -json -m all | docker run --rm -i
   sonatypecommunity/nancy sleuth`. Read the output. Count HIGH+
   findings.
2. Run an image scan: build the movies-bartr image, then `trivy
   image --severity HIGH,CRITICAL movies-api:lab`.
3. **Introduce a CVE on purpose.** Find an old version of a Go
   library known to have a CVE (the trivy findings from step 2 are
   a starting point, *if* there are any; otherwise pin
   `github.com/dgrijalva/jwt-go v3.2.0+incompatible` for an
   easy-to-find one). Re-run the audit. Read the finding.
4. Revert. Confirm clean.
5. Generate an SBOM: `syft movies-api:lab -o spdx-json | jq
   '.packages | length'`. Look at the first few entries —
   confirm they're real packages with real versions.

#### Knowledge check

1. Three layers of supply-chain hygiene — name them.
2. A scanner that runs once a quarter is theater. Why? What
   cadence is the minimum for a real control?
3. CVE severity thresholds — what's the trade-off between blocking
   on HIGH+ vs blocking on CRITICAL only?
4. A CVE waiver workflow — what fields belong on every waiver?
5. SBOM is required by some procurement processes. What's the
   smallest workflow change that makes "ship an SBOM with every
   release" routine?

---

### Module 6 — OWASP top-10 for GET-only HTTP APIs

> **Yes, even GET-only services have an OWASP attack surface.**
> Smaller than a full app, but not zero. This module names what to
> watch for in movies-spec-class services.

#### Concept

Most of OWASP's classic top-10 (A03 injection, A04 insecure design,
A07 auth failures) presumes write operations. movies-spec is GET-only.
The relevant slice:

1. **A01: Broken Access Control.** Even GET-only, "which IDs can
   *this* caller see?" matters if any per-tenant scoping exists.
   For movies (public catalog data), this is moot. For *any* real
   service, it's not.
2. **A02: Cryptographic Failures.** TLS termination at the edge,
   strong ciphers, no plaintext anywhere a captured packet helps.
   Spec defers TLS to the Ingress layer; the service speaks plain
   HTTP inside the cluster.
3. **A03: Injection.** Even GET endpoints take query parameters. Any
   query that ends up in a SQL string, a shell command, an LDAP
   filter, or an HTML template needs escaping. movies-spec uses
   in-memory maps and never reaches a SQL planner — but the agent
   may add caching layers or downstream calls that change this.
4. **A05: Security Misconfiguration.** *The category that bites
   movies-spec.* Most of Modules 1–4 above are spec-bar countermeasures
   for A05.
5. **A06: Vulnerable & Outdated Components.** Module 5 above.
6. **A09: Security Logging & Monitoring Failures.** Module 6 of the
   observability guide is the relevant counter; the security-side
   read is "are we logging the events that would let us detect an
   attack." For a GET-only API, that's 4xx/5xx rates by source.
7. **A10: SSRF.** Server-Side Request Forgery — a service that takes
   a URL parameter and fetches it. movies-spec doesn't have this
   shape. If you add a "fetch poster art" endpoint, you do.

Two service-specific risks that aren't in the top-10 but matter:

- **Information disclosure via verbose errors.** The spec's
  `problem+json` (RFC 7807) format is fine; the danger is including
  internal paths, stack traces, or DB schema in the `detail` field.
- **Rate-limit / abuse.** GET endpoints get hammered by scrapers,
  bots, and the occasional accidental DoS from a misconfigured
  client. The spec doesn't require rate limiting; the platform
  (Traefik IngressRoute middleware, or a sidecar) usually does it.

#### Example

A `problem+json` response that leaks too much:

```json
{
  "type": "https://movies-api.example.com/errors/not-found",
  "title": "movie not found",
  "status": 404,
  "detail": "movie id 'tt9999999' not found in /data/movies.json (line 42819)",
  "instance": "/api/movies/tt9999999"
}
```

`/data/movies.json (line 42819)` is a gift. The same response, safe:

```json
{
  "type": "https://movies-api.example.com/errors/not-found",
  "title": "movie not found",
  "status": 404,
  "detail": "movie id 'tt9999999' not found",
  "instance": "/api/movies/tt9999999"
}
```

A Traefik middleware rate limit (overlay-level config):

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: movies-rate-limit
spec:
  rateLimit:
    average: 100        # req/s sustained
    burst: 200
```

#### Lab

1. Curl every movies-bartr endpoint with a deliberately malformed
   input (long IDs, special characters, SQL-like fragments). Read
   the responses. Score each for information disclosure.
2. Read `internal/httpapi/problem.go` (or equivalent error path).
   Identify what *would* leak if the operator hadn't been careful.
3. **A03 lab.** Add a query parameter that gets concatenated into a
   downstream call (a scratch handler is fine). Demonstrate the
   injection. Fix it with proper escaping / parameterization.
4. **A10 lab.** Add a `/api/preview?url=...` endpoint that fetches
   the URL. Demonstrate SSRF (point it at
   `http://169.254.169.254/` on a cloud VM, or at an internal IP).
   Add an allow-list. Re-test.
5. Remove the lab handlers. Restore.

#### Knowledge check

1. Which OWASP top-10 categories actually apply to a GET-only API?
2. The `problem+json` `detail` field is freeform. What's the
   discipline that keeps it from leaking?
3. A03 injection on a GET endpoint — give a real example specific
   to movies-spec (something the agent might write that opens it).
4. SSRF (A10) is invisible until someone adds a URL-taking endpoint.
   What's the methodology check that catches it during code review?
5. Rate limiting at Traefik vs in the application — what's the
   trade-off? When would you want both?

---

### Module 7 — Putting it together: read a security baseline cold

> **The capstone.** No new content; entirely review-driven. The test
> of whether modules 1–6 stuck is whether you can read a security
> baseline cold and tell whether it's the real thing or theater.

#### Concept

The skill: hand the operator a `Deployment` + `NetworkPolicy` +
`Dockerfile` they've never seen, and have them score it against
modules 1–6 in under 15 minutes. Output: a one-page review with
findings (must-fix / should-fix / consider) and a verdict (ship,
fix-then-ship, do-not-ship).

The format mirrors what a real security review looks like — except
the operator is doing it, not waiting for a security team. That's the
point: the team owns its own security baseline.

#### Example

A review template:

```markdown
# Security baseline review — <service> <tag>

Reviewer: <name>
Date: <date>
Verdict: <ship | fix-then-ship | do-not-ship>

## Pod / container (Module 1)
- runAsNonRoot:                <set | not-set | misconfigured>
- runAsUser:                   <set | not-set | misconfigured>
- readOnlyRootFilesystem:      <set | not-set | misconfigured>
- capabilities.drop ALL:       <set | not-set | misconfigured>
- allowPrivilegeEscalation:    <false | true | not-set>
- seccompProfile:              <RuntimeDefault | other | not-set>
- automountServiceAccountToken false: <set | not-set>

## NetworkPolicy (Module 2)
- default-deny in place:       <yes | no>
- narrow-allow for workload:   <yes | no>
- egress restricted:           <yes | no>
- selectors verified portable: <yes | no>

## Image (Module 3)
- distroless / minimal base:   <yes | no>
- multi-stage build:           <yes | no>
- non-root USER in Dockerfile: <yes | no>
- vulnerability scan run:      <yes | no, HIGH+ count>
- SBOM available:              <yes | no>

## Secrets (Module 4)
- secrets-in-repo:             <none | found>
- injection pattern:           <envFrom | valueFrom | projected | mixed>
- ConfigMap holding creds:     <none | found>
- secrets logged at startup:   <no | yes>

## Supply chain (Module 5)
- dependency audit cadence:    <every PR | nightly | quarterly | none>
- image scan cadence:          <every build | nightly | quarterly | none>
- CVE waiver workflow:         <documented | informal | none>

## App-layer (Module 6)
- problem+json discipline:     <safe | leaks | not-applicable>
- rate limiting:               <Traefik | app | none>
- SSRF-shape endpoints:        <none | <list>>

## Findings
1. [must-fix] ...
2. [should-fix] ...
3. [consider] ...
```

#### Lab

This module's lab *is* the capstone:

1. Pick a service that is **not** movies-bartr (one from your work,
   a public reference repo, or a Claude-generated scratch service).
2. Run the review using the template. Time yourself: 15 minutes
   target.
3. Compare your findings to what Claude finds when asked the same
   question. Where do you disagree? Why?
4. Now do the same exercise on movies-bartr. *You should find no
   must-fix findings.* If you do, you've found a real baseline gap
   and should file it.
5. Re-run on a deliberately-broken movies-bartr (uncomment one
   wrong control). Confirm the review catches it.

#### Knowledge check

1. The "ship / fix-then-ship / do-not-ship" verdict — what's the
   smallest must-fix that should trigger "do-not-ship"?
2. The review template has six sections (one per module). What
   would you cut for a 5-minute review? What would you cut to make
   it not-worth-doing?
3. The agent and the human review the same baseline and disagree.
   What's the right way to resolve?
4. The methodology says "the team owns its own security baseline."
   What concretely does that mean? Who runs the review at each
   release?
5. The review is a snapshot. What's the cadence that keeps a
   baseline from rotting? Tie this back to per-release review
   from the observability guide.

---

## Per-release review

Same template as the observability guide; see the
[Per-release review template section](study-guide-observability.md#per-release-review-template).
Security-specific addition: every release runs the **Module 7
capstone review** against the touched workload. The review is short
(15 min) but mandatory; a release without a passing baseline review
is fix-then-ship, not ship.

This is the lever that prevents "the agent generated something that
looks right" from becoming "we shipped something exploitable."

## What this guide is and is not

- **Is:** the slice of security that maps onto movies-spec §8.1 +
  §13 + the GitOps secrets story + the OWASP categories that apply
  to GET-only HTTP APIs.
- **Is not:** an offensive-security curriculum. The operator learns
  to *read* and *score* baselines; pen-testing is its own
  inventory.
- **Is not:** a Kubernetes security textbook. PodSecurity admission,
  Open Policy Agent / Kyverno, runtime security (Falco), service
  mesh mTLS, Pod Security Standards profiles — all real and all out
  of scope here. Each is its own future module if the spec grows
  that way.
- **Is not:** an opinion on a specific scanner. Trivy is the
  reference because it's free, fast, and widely-deployed; Snyk /
  Grype / Anchore are all defensible alternatives.

## Open questions

- **Module 6 (OWASP) is the lightest module.** It's intentionally
  narrow because the spec is narrow. If the curriculum grows to
  cover services with write endpoints / auth / sessions, OWASP
  deserves its own guide rather than one module here.
- **Module 7 (capstone review) might deserve its own ceremony in
  the close ritual.** Right now it lives under per-release review;
  worth considering whether it should be promoted to a
  standalone close-ritual step (alongside green tests and FF-merge).
- **Admission control (Kyverno / Pod Security Standards) is a
  notable gap.** The capstone review catches misconfigurations at
  review time; admission control catches them at apply time. Worth
  a module if/when the spec grows multi-tenant cluster requirements.
- **Runtime security (Falco etc.) and service mesh mTLS** are not
  covered. Both are real enterprise needs at scale and arguably
  belong in the curriculum eventually. Currently out of scope to
  keep this guide bounded by what movies-spec requires.

## Status

- Not yet run end-to-end with any operator.
- Anchored in movies-bartr's real `deployment.yaml` and
  `networkpolicy.yaml` — every spec-bar control cited here is set
  correctly in the working repo.
- Unlocks for promotion to `methodology/` once at least one full
  run has used Module 7's capstone review at release and
  reported honestly on whether it caught real findings or only
  theater.
