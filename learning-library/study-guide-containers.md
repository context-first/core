# Study Guide — Container Fundamentals (movies-spec §9, domain A)

> **DRAFT — NOT FOR PUBLICATION.** Sixth instance of the study-guide
> format. Scoped from domain A of
> [skills-inventory.md](skills-inventory.md). Anchored in the
> movies-bartr `src/Dockerfile` and cross-referenced with
> [study-guide-kustomize.md](study-guide-kustomize.md) Module 6
> (Dockerfile in the deploy pipeline),
> [study-guide-go.md](study-guide-go.md) Module 10 (build flags), and
> [study-guide-security.md](study-guide-security.md) Module 3 (image
> security).

## Why this exists

Containers are the layer the operator touches a hundred times a day
without thinking. The agent will happily generate a `Dockerfile`
that *runs*. The same Dockerfile may be a 1.2GB image, run as root,
re-download dependencies on every build, and never cache a layer.
None of that fails a build.

This guide makes the operator literate in what a container actually
is, why a 15MB distroless image is the floor (not the ceiling), and
how to read a Dockerfile and predict the image's size, layer count,
cache behavior, and attack surface before building it.

**What this guide is anchored in:**

- Spec §9 — single multi-stage Dockerfile, distroless or minimal
  runtime, non-root.
- Spec §13 — non-root, read-only FS, no secrets in image layers.
- movies-bartr `src/Dockerfile` — a real, working, ~25MB, two-binary,
  multi-stage build using `golang:1.26-bookworm` and
  `gcr.io/distroless/static-debian12:nonroot`.
- [study-guide-security.md](study-guide-security.md) Module 3 —
  re-reads the same Dockerfile with security eyes. This guide
  covers what's *under* the Dockerfile (the container model
  itself).

## How to use this guide

Same protocol as the other study guides. One curriculum-level rule
specific to this one: **every lab uses `docker history`,
`docker image inspect`, or `dive` to read the image.** The image
is the artifact. If the operator can't read it, they're shipping
something they don't understand.

## Modules

### Module 1 — What a container actually is

> **Most "Docker is slow" complaints trace to not knowing this.**
> The mental model matters more than the commands.

#### Concept

A container is **a Linux process with three things bolted on**:

1. **Namespaces** — kernel features that isolate what the process
   *sees*. Seven of them: PID (own process tree), NET (own network
   stack), MNT (own filesystem view), UTS (own hostname), IPC,
   USER (own UID mapping), CGROUP. Each namespace is cheap; a
   container is the bundle.
2. **cgroups (control groups)** — kernel features that limit what
   the process *consumes*. CPU shares, memory caps, IO weight,
   PID counts. This is where K8s `resources.limits` ultimately
   lands.
3. **A filesystem (rootfs) handed to it via overlay** — the
   "image" is a stack of read-only layers + one writable layer on
   top. The container sees a single merged tree; the kernel does
   copy-on-write under it.

What it is **not**:

- **Not a VM.** No guest kernel, no hypervisor. The container
  shares the host kernel. A container on Linux 5.15 *is* a Linux
  5.15 process; there is no Linux running inside.
- **Not a security boundary equivalent to a VM.** A kernel
  vulnerability is a container escape. Use `seccomp`, capability
  dropping, non-root UIDs (Module 5; see also
  [study-guide-security.md](study-guide-security.md) Module 1).
- **Not Docker.** Docker is a daemon + CLI that wraps `containerd`,
  which wraps `runc`, which actually creates the namespaces and
  cgroups via Linux syscalls. K8s skips Docker entirely (talks to
  `containerd` directly via CRI). On macOS / Windows, "Docker" is a
  *Linux VM* (LinuxKit / WSL2) running `containerd` plus a daemon
  proxy on the host. **The host OS is not running containers.**

The four things that follow from the model:

1. A container starts in microseconds because no kernel boots.
2. A container's memory footprint is the memory of one process,
   not the memory of a guest OS.
3. A container has no `init` by default. PID 1 is your binary. If
   your binary forks and abandons children, you have zombies; if it
   doesn't handle SIGTERM, K8s will kill it after the grace
   period.
4. The image is *layers*, and every line in your `Dockerfile` that
   changes the filesystem creates one. Layer ordering is the
   difference between a 30-second rebuild and a 5-minute one.

#### Example

The actual call graph when you type `docker run alpine sh`:

```
docker CLI
  ↓ HTTP over /var/run/docker.sock
dockerd
  ↓ gRPC
containerd
  ↓ exec
runc
  ↓ syscalls: clone(CLONE_NEWPID|CLONE_NEWNET|CLONE_NEWNS|...),
              setns, pivot_root, prctl, etc.
Linux kernel
  ↓ creates process, attaches namespaces, joins cgroups
your binary (`sh`) — PID 1 inside its PID namespace, real PID
                     <something> on the host
```

On macOS / Windows, insert a "Linux VM" between `docker CLI` and
`dockerd` (covered in [skills-inventory.md](skills-inventory.md)
domain B).

#### Lab

1. On a Linux host (or in WSL2): `docker run -d --name lab1
   alpine sleep 3600`. Then on the **host**: `ps -ef | grep sleep`
   and `pstree -p <pid>`. The `sleep` process is a host process.
2. Inside the container: `docker exec -it lab1 sh`, then `ps`.
   Notice `sleep` is PID 1 (inside the PID namespace) — and your
   `sh` is PID 6 or so. Different number; same actual process the
   host sees.
3. `cat /proc/<host-pid-of-sleep>/cgroup` on the host. Notice the
   cgroup path includes the container ID.
4. `docker run --rm --memory=64m alpine sh -c 'cat
   /sys/fs/cgroup/memory.max'`. Watch the cgroup limit appear.
5. `time docker run --rm alpine echo hi` and compare to `time
   docker run --rm ubuntu echo hi`. The startup time *isn't*
   dominated by the image — it's the daemon round-trip + runc.
6. **Falsify "container = VM."** `docker run --rm alpine uname
   -r`. The kernel version is *your host's kernel*, not the
   image's. Repeat with `ubuntu`, `debian:bookworm`. Same kernel
   every time.

#### Knowledge check

1. Name the three primitives a container is built from. Which one
   isolates *what the process sees* and which one limits *what it
   consumes*?
2. Why does a container start in microseconds where a VM takes
   seconds-to-minutes?
3. You run `docker run alpine uname -r` on a Linux 5.15 host. What
   kernel version does it print? Why?
4. A container's PID 1 receives SIGTERM and ignores it. What does
   K8s do? After how long?
5. "Docker on Mac" — what is actually running where? Why does that
   matter for bind-mount performance?
6. A kernel CVE drops. Are your containers vulnerable? Why or why
   not?

---

### Module 2 — Dockerfile authoring: the directives that matter

#### Concept

A `Dockerfile` is a script. Every directive either (a) creates a
new image layer, (b) sets metadata, or (c) controls the build
context. Knowing which is which is the first half of writing one
that caches well.

**Directives that create layers** (each adds to image size):

- `FROM` — picks the base; the base's layers come with you.
- `COPY` / `ADD` — copy files into the image. `COPY` is the safe
  default; `ADD` does URL fetching and tar extraction (mostly
  avoid).
- `RUN` — execute a command at build time. The resulting filesystem
  changes are baked into a new layer.

**Directives that set metadata** (no layer, but visible in
`docker inspect`):

- `ENV` — environment variables, persist at runtime.
- `ARG` — build-time variables. Do **not** persist at runtime
  unless you re-export to `ENV`. *Do not put secrets here* — they
  end up in image history.
- `LABEL` — arbitrary key/value metadata. Use OCI labels
  (`org.opencontainers.image.source`, `.version`, `.revision`).
- `USER` — sets the runtime user. Crucial for non-root (see
  Module 5).
- `WORKDIR` — sets the working directory. Creates the directory if
  missing.
- `EXPOSE` — documentation only. Doesn't open a port. K8s ignores
  it.

**Entry behavior:**

- `ENTRYPOINT ["bin", "arg1"]` (exec form) — the binary that runs
  as PID 1. **Always use the JSON array form.** The string form
  wraps in `/bin/sh -c` and breaks signal forwarding.
- `CMD ["arg2"]` — default args appended to `ENTRYPOINT`.
  Overridable by `docker run <image> <override>`.
- The relationship: `ENTRYPOINT` is the verb, `CMD` is the default
  noun.

**Build-context directives:**

- `.dockerignore` — exclude files from the build context. Without
  it, `COPY .` ships your `.git/`, `node_modules/`, and test
  output into the build. Slow and leaky.
- `# syntax=docker/dockerfile:1.7` — opt into newer build features
  (heredocs, secrets mount, cache mount). Should be the first
  line.

#### Example

A minimal Go service Dockerfile (single-stage, for contrast with
Module 3):

```dockerfile
# syntax=docker/dockerfile:1.7
FROM golang:1.26-bookworm
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -o /movies-api ./cmd/movies-api
USER 1000:1000
EXPOSE 8080
ENTRYPOINT ["/movies-api"]
```

This **works** and is **wrong**. The resulting image is ~1.2GB
because the Go toolchain, source, and module cache all live in the
final image. Module 3 fixes it with multi-stage.

The exec-form trap:

```dockerfile
# WRONG — string form, wraps in /bin/sh -c, breaks SIGTERM forwarding
ENTRYPOINT /movies-api --port 8080

# RIGHT — exec form, your binary is PID 1
ENTRYPOINT ["/movies-api", "--port=8080"]
```

A `.dockerignore` worth shipping:

```
.git
.gitignore
*.md
README.md
**/node_modules
**/__pycache__
**/*.test
**/testdata
coverage.out
.vscode
.idea
```

#### Lab

1. In `repos/movies-bartr/src/`: copy `Dockerfile` aside and write
   the minimal single-stage version above. Build it as
   `movies-single:lab`.
2. `docker images movies-single:lab` — note the size.
3. Now build the real Dockerfile as `movies-multi:lab`. Compare.
   Expect roughly a **50× ratio**.
4. `docker history movies-single:lab` and `docker history
   movies-multi:lab`. Read each layer's `SIZE`. Find the layers
   that account for the bulk.
5. **Falsify the string-form trap.** Edit the single-stage to use
   `ENTRYPOINT /movies-api`. Build, run. `docker stop <id>` and
   time how long it takes to die. Switch to `ENTRYPOINT
   ["/movies-api"]`. Repeat. *The string form takes the full
   10s SIGKILL timeout; the exec form exits in milliseconds.*
6. Add a `.dockerignore` that excludes `.git`. Rebuild. Note
   build-context size in the output. Compare to without.

#### Knowledge check

1. Which directives create a new image layer? Which only set
   metadata?
2. Why does `ENTRYPOINT ["bin"]` matter vs `ENTRYPOINT bin`?
   What's the failure mode of the latter under K8s `kubectl
   delete pod`?
3. `ARG` vs `ENV` — when is each appropriate? Why is `ARG SECRET=...`
   a security smell?
4. Without a `.dockerignore`, what gets shipped into your build
   context that probably shouldn't?
5. `EXPOSE 8080` — what does it actually do? What does it *not*
   do?

---

### Module 3 — Multi-stage builds: the spec §9 floor

> **Single-stage Dockerfiles for compiled languages are
> indefensible in 2026.** Multi-stage is one extra `FROM` line and
> a 50× size win. The spec requires it.

#### Concept

A multi-stage build has more than one `FROM`. Each `FROM` starts a
new stage with its own filesystem; you copy artifacts forward via
`COPY --from=<stage-name>`. The final stage is what ships; earlier
stages are discarded.

The pattern:

1. **Build stage**: rich base (full toolchain — `golang:1.26`,
   `rust:1.80`, `node:20`, `python:3.12`). Pull dependencies,
   compile, test. Big, fat, slow.
2. **Runtime stage**: minimal base (`distroless`, `alpine`,
   `scratch`). `COPY --from=build` the compiled binary. Nothing
   else.

Why this is the right shape:

- **Size.** Drop 1GB+ of build tooling from the runtime image.
- **Attack surface.** The runtime image has no compiler, no
  package manager, no shell (for distroless). An attacker who gets
  code execution has nothing to `apt install`.
- **Cache.** The build stage can cache aggressively (Module 4)
  because its base doesn't change.
- **Single source of truth.** One file describes the entire build
  end-to-end. Spec §9 forbids "build a binary in a sidecar repo
  then `COPY` it in" precisely because that splits the truth.

Variations that earn their complexity:

- **Multiple binaries from one tree** (movies-bartr does this).
  One build stage, multiple `go build` invocations, multiple
  `COPY --from=build` lines into the runtime — or two runtime
  stages with different entrypoints.
- **Test stage as a target** — a `FROM build AS test` stage that
  runs the test suite. `docker build --target=test .` runs tests
  in CI; default target ships the runtime.
- **Cross-compilation** — build for arm64 from an amd64 host using
  `--platform`, `GOARCH`, etc. Lab in Module 7.

#### Example

The actual movies-bartr Dockerfile, lightly annotated:

```dockerfile
# syntax=docker/dockerfile:1.7

# Build stage — full Go toolchain on Debian Bookworm.
FROM golang:1.26-bookworm AS build
WORKDIR /src

# Module cache layer (changes rarely → cached often).
COPY go.mod go.sum ./
RUN go mod download

# Source layer (changes per commit → invalidates everything below).
COPY cmd ./cmd
COPY internal ./internal

ARG VERSION=1.0.0
ENV CGO_ENABLED=0 GOOS=linux GOARCH=amd64

# Two binaries from one tree.
RUN go build -trimpath -ldflags "-s -w -X .../version.Version=${VERSION}" \
      -o /out/movies-api ./cmd/movies-api \
 && go build -trimpath -ldflags "-s -w -X .../version.Version=${VERSION}" \
      -o /out/webv ./cmd/webv

# Runtime stage — distroless, non-root, no shell, no libc.
FROM gcr.io/distroless/static-debian12:nonroot

COPY --from=build /out/movies-api /movies-api
COPY --from=build /out/webv /webv
COPY data /data                # JSON catalog baked in per spec §5.2
COPY webv /webv-suites         # webv test fixtures

USER 1000:1000
EXPOSE 8080
ENTRYPOINT ["/movies-api"]
```

The build stage is ~1GB. The runtime stage is ~25MB. The 975MB
delta is the Go toolchain plus everything Debian ships — never in
the runtime image, never in production, never on an attacker's
shell prompt.

Notice **two binaries from one image** — the same image runs as
`movies-api` (default entrypoint) or as `webv` (the load
generator in `deploy/webv/`, which sets a different entrypoint at
the pod level). One build, two roles, one set of dependencies to
audit.

#### Lab

1. In `repos/movies-bartr/src/`: build the real Dockerfile as
   `movies-real:lab`.
2. Find each stage's intermediate image: `docker images --filter
   "label=stage=build"` (or use `docker build --target=build -t
   movies-build:lab .` to materialize it). Compare sizes.
3. Use `dive movies-real:lab` (install if needed). Click through
   each layer; identify the binary copy, the data copy, the
   webv-suites copy.
4. **Run as the second binary.** `docker run --rm
   --entrypoint=/webv movies-real:lab --help`. The same image
   runs as either binary depending on the override.
5. Add a test stage: `FROM build AS test` with `RUN go test
   ./...`. Build with `docker build --target=test .`. Confirm
   tests run. Build with the default target. Confirm the test
   stage doesn't ship.
6. Try `FROM scratch` as the runtime (instead of distroless).
   Build. Run. *It works for a static Go binary.* Note the
   trade-off: no `/etc/passwd`, no `/etc/ssl/certs`, no `nsswitch`,
   no nothing. Distroless gives you the minimum reasonable runtime
   environment; `scratch` gives you nothing at all.

#### Knowledge check

1. Why is single-stage indefensible for a compiled-language
   service?
2. What does `COPY --from=build` actually do? Where does the
   build stage's filesystem live?
3. The movies-bartr Dockerfile produces **two** binaries from one
   build. What's the operational benefit? What's the risk?
4. A `FROM build AS test` stage that runs `go test` — why use a
   stage at all instead of running tests in CI's own job?
5. `FROM scratch` vs `FROM distroless/static:nonroot` — what does
   distroless give you that scratch doesn't?

---

### Module 4 — Layer caching: ordering for cache hit rate

> **The skill that turns a 5-minute rebuild into a 5-second one.**
> The agent's defaults are wrong on this one most of the time.

#### Concept

The build cache is per-layer. A layer hits the cache when:

1. The base image hash matches.
2. The directive is byte-identical.
3. For `COPY` / `ADD`: the *contents* of the copied files are
   identical (computed as a checksum).
4. Every previous layer also hit the cache.

The fourth rule is the load-bearing one. **Cache miss cascades.**
The moment any layer misses, every layer below it is rebuilt.

The ordering rule that follows: **put what changes rarely above
what changes often.**

For a typical service:

| Change frequency | Goes in layers... |
|---|---|
| Almost never | `FROM`, `WORKDIR`, system packages |
| Rarely | Dependency manifests (`go.mod`, `package.json`, `requirements.txt`) |
| Rarely | Dependency install (`go mod download`, `npm ci`, `pip install`) |
| Every commit | Source code |
| Every commit | Compile / build |
| Every commit | Final binary copy into runtime stage |

If you reverse "source" and "dependency install," every code change
re-downloads every dependency. The agent will sometimes write
`COPY . .` *before* the dependency download because it's "simpler";
this is the single most common Dockerfile mistake.

**BuildKit cache mounts** (the modern lever, requires `# syntax=...`
header):

```dockerfile
RUN --mount=type=cache,target=/root/.cache/go-build \
    --mount=type=cache,target=/go/pkg/mod \
    go build -o /out/svc ./cmd/svc
```

The cache mount **persists across builds** at the daemon level,
not inside the image. Go's build cache (often hundreds of MB) is
re-used between builds without bloating the image. Same idea for
npm (`/root/.npm`), pip (`/root/.cache/pip`), apt
(`/var/cache/apt`).

**Secrets mount** (the right answer to "I need a token at build
time"):

```dockerfile
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci
```

The secret is available during the RUN, **not baked into the
layer**. Pass at build time: `docker build --secret
id=npmrc,src=$HOME/.npmrc .`

#### Example

The **wrong** ordering (every code change refetches modules):

```dockerfile
FROM golang:1.26-bookworm AS build
WORKDIR /src
COPY . .                       # ← every commit invalidates this
RUN go mod download            # ← so this re-runs every time
RUN go build ...
```

The **right** ordering (movies-bartr's actual approach):

```dockerfile
FROM golang:1.26-bookworm AS build
WORKDIR /src
COPY go.mod go.sum ./          # ← only changes when deps change
RUN go mod download            # ← cached unless go.{mod,sum} changes
COPY cmd ./cmd                 # ← only invalidates from here down
COPY internal ./internal
RUN go build ...
```

The same idea for other ecosystems:

```dockerfile
# Node
COPY package.json package-lock.json ./
RUN npm ci
COPY . .

# Python
COPY requirements.txt ./
RUN pip install -r requirements.txt
COPY . .

# Rust (with cargo-chef or the recipe pattern)
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo 'fn main(){}' > src/main.rs && cargo build --release
COPY src ./src
RUN touch src/main.rs && cargo build --release
```

#### Lab

1. In `repos/movies-bartr/src/`: build the real Dockerfile.
   `docker build -t movies:cache-lab .` and watch the output.
2. Build it again with no changes. Every layer should print
   `CACHED`. Time it.
3. Touch a file under `cmd/`: `touch cmd/movies-api/main.go`.
   Build again. Observe which layers hit cache and which don't.
   The `go mod download` layer should still be cached.
4. Touch `go.mod`: `touch go.mod`. Build. Observe `go mod
   download` re-runs (and everything below).
5. **Write the broken version.** Edit the Dockerfile to put
   `COPY cmd ./cmd` *before* `COPY go.mod go.sum`. Build. Touch
   anything under `cmd/`. Build again. *`go mod download`
   re-runs every time.* That's the failure mode.
6. Add a BuildKit cache mount for `/root/.cache/go-build` and
   `/go/pkg/mod`. Time a clean rebuild before and after.
7. Restore the real Dockerfile.

#### Knowledge check

1. The four conditions for a layer cache hit — name them.
2. "Cache miss cascades." What does that mean concretely?
3. Why does `COPY go.mod go.sum ./` and `RUN go mod download` go
   *above* `COPY cmd ./cmd` and `COPY internal ./internal`?
4. BuildKit cache mounts vs image layers — what's the difference,
   and what's the operational consequence?
5. You need a private registry token at `npm ci` time. Three
   approaches: `ARG NPM_TOKEN=...`, `COPY .npmrc /root/`, or
   `--mount=type=secret`. Rank them by leak risk and explain.

---

### Module 5 — Distroless, non-root, read-only: the spec §13 floor

> **Cross-references [study-guide-security.md](study-guide-security.md)
> Module 3.** That guide reads the Dockerfile with *security* eyes;
> this module reads it with *container* eyes (what the choices buy
> you, how they interact with the runtime).

#### Concept

Three runtime-image choices the spec requires, plus the K8s-side
fields that complete them:

**1. Minimal base.**

| Base | Size | Has | Use when |
|---|---|---|---|
| `scratch` | 0 bytes | Nothing. No `/etc`, no DNS resolver, no certs. | Static Go binary, no network calls beyond raw IP, no users to look up. Rare in practice. |
| `gcr.io/distroless/static-debian12:nonroot` | ~2MB | CA certs, `/etc/passwd` with `nonroot` user, `tzdata`, `nsswitch`. No shell, no libc, no package manager. | **The default for static Go binaries.** What movies-bartr uses. |
| `gcr.io/distroless/base-debian12:nonroot` | ~20MB | Above + glibc + libssl. No shell. | Dynamically linked binaries (CGO, some Rust crates). |
| `gcr.io/distroless/cc-debian12:nonroot` | ~25MB | Above + libgcc + libstdc++. | C/C++ apps. |
| `alpine:3.20` | ~8MB | musl libc, busybox shell, apk. | When you genuinely need a shell. Trade-off: musl vs glibc compatibility issues. |
| `chainguard/static` | ~2MB | Similar to distroless static, but maintained by Chainguard with daily CVE patching. | Same use as distroless, with stronger CVE response. |
| `ubuntu:24.04` / `debian:bookworm` | ~75-100MB | Full distro. | **Almost never the right answer for a runtime.** |

**2. Non-root user.** Two halves:

- **Image side**: `USER 1000:1000` (or `USER 65532:65532` for
  distroless's `nonroot` UID). Sets the default UID the container
  runs as.
- **K8s side**: `runAsNonRoot: true` + `runAsUser: 1000`. The
  kubelet **refuses to start** the pod if the image's effective
  user is root.

Both because: image-side prevents accidental `docker run` as root;
K8s-side prevents an image with a wrong `USER` from being deployed.

**3. Read-only root filesystem.**

- **K8s side**: `readOnlyRootFilesystem: true`. The container's
  `/` is read-only at runtime.
- **Image side**: no equivalent — read-only is enforced by the
  runtime, not baked into the image. But the image must be
  *prepared* for it: any directory the app writes to must be a
  volume mount, not a normal directory.

The common writable directories an app might need:

- `/tmp` — `emptyDir` mount.
- `/var/cache/<app>` — `emptyDir` or PVC.
- Application data — PVC.

If you set `readOnlyRootFilesystem: true` without providing the
mounts, the app crashes on first write. The discovery is "ship
once, watch the logs, add the mount." Module 6 of
[study-guide-security.md](study-guide-security.md) has the lab
for this.

The fourth control that completes the picture:

**4. `automountServiceAccountToken: false`** at pod level. A
workload that doesn't talk to the K8s API doesn't need an SA
token; defaulting one in is a credential you didn't need to
hand out.

#### Example

The movies-bartr runtime stage, read for what it gives you:

```dockerfile
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/movies-api /movies-api
COPY --from=build /out/webv /webv
COPY data /data
COPY webv /webv-suites
USER 1000:1000
EXPOSE 8080
ENTRYPOINT ["/movies-api"]
```

What you get:

- ~25MB total (most of which is the two Go binaries + the JSON
  data).
- No shell. `kubectl exec -it pod -- /bin/sh` returns an error.
  Ephemeral debug containers (`kubectl debug`) work; manual shell
  access doesn't.
- No package manager. An attacker with code execution can't `apt
  install` anything.
- No libc. Most off-the-shelf exploit payloads don't run.
- Runs as UID 1000 by default. Pod-spec `runAsUser: 1000` lines up.
- Read-only-FS-compatible (the app writes only to where mounts
  cover it).

What you give up:

- Debugging via `kubectl exec` is gone. Use `kubectl debug node`
  or attach an ephemeral debug container with `kubectl debug pod
  --image=nicolaka/netshoot`.
- Crash investigation needs structured logs to be good — there's no
  way to `cat` a file from inside a crashed container.

This is the trade movies-spec makes deliberately: lose interactive
debugging, gain a runtime an attacker can't easily live in.

#### Lab

1. **Prove there's no shell.** `docker run --rm -it
   gcr.io/distroless/static-debian12:nonroot /bin/sh`. Error. Try
   `/bin/bash`, `/sh`, `sh`. All errors.
2. **Use an ephemeral debug container.** Deploy movies-bartr to a
   local cluster. `kubectl debug -it <pod> --image=nicolaka/netshoot
   --target=movies-api -n movies`. You get a shell — in a sidecar
   that joins the pod's namespaces but has its own image.
3. **Compare image sizes.** Build a `FROM ubuntu:24.04` variant of
   the movies Dockerfile. Compare with the distroless version.
   Note both the disk size and the count of files
   (`docker run --rm <image> find / | wc -l` won't work because no
   shell; use `dive` or `docker save | tar t | wc -l`).
4. **Test `readOnlyRootFilesystem`.** Deploy with
   `readOnlyRootFilesystem: true` but **without** the
   `/tmp` `emptyDir` mount. Run the app. Find what breaks (Go's
   default `os.TempDir()` is `/tmp`; many libraries assume it
   exists and is writable).
5. Add the `emptyDir`. Confirm it works.
6. **Verify the non-root chain.** `docker run --rm
   movies-api:1.0.0 id`. Expect `uid=1000(nonroot) gid=1000(nonroot)`.
   Then deploy with `runAsNonRoot: true`. Then deploy a *different*
   image (`USER root` in the Dockerfile) with `runAsNonRoot: true`.
   The pod fails to start. Read the error.

#### Knowledge check

1. Distroless `static` vs `base` vs `cc` — when do you need each?
2. `USER 1000` in the Dockerfile and `runAsNonRoot: true` in K8s
   — which is the real control? What does each one catch?
3. `readOnlyRootFilesystem: true` requires preparing the image.
   What does "preparing" mean concretely?
4. You can no longer `kubectl exec` into your service to debug.
   What replaces that workflow?
5. Three real costs of a distroless runtime image (not security
   benefits — costs). Name them.

---

### Module 6 — Tagging, versioning, and image identity

> **"`latest` is not a version."** This module fixes the discipline.

#### Concept

An image has three identities:

1. **A digest**: `sha256:abc123...`. Immutable. Always refers to
   *exactly the bytes you built*. The only safe identifier in a
   deploy manifest.
2. **A tag**: `movies-api:1.0.0`, `:latest`, `:dev`,
   `:pr-42-abc1234`. Mutable. Whoever can push to the registry
   can re-point the tag.
3. **A name**: `ghcr.io/bartr/movies-api`. The repository.

The spec §12 inner loop depends on tagging discipline:

- **Semver for releases**: `1.0.0`, `1.0.1`, `1.1.0`. Bumped per
  spec §12 step 1.
- **Never `:latest` in deploy manifests.** `:latest` is mutable;
  K8s caches images by tag-and-digest. Two pods can be running
  *different versions* of `:latest` depending on when they pulled.
- **`imagePullPolicy: IfNotPresent` for versioned tags.**
  `imagePullPolicy: Always` for `:latest` (which you're not using
  anyway). The spec uses semver tags, so `IfNotPresent` is the
  default.
- **Pin by digest in high-trust paths.** `image:
  ghcr.io/bartr/movies-api@sha256:abc...` cannot be re-pointed
  by anyone. This is what `cosign verify` + signed digests
  enable (see [study-guide-security.md](study-guide-security.md)
  Module 3).

OCI annotations that should be on every image:

```dockerfile
LABEL org.opencontainers.image.source="https://github.com/bartr/bartr-movies"
LABEL org.opencontainers.image.revision="${GIT_SHA}"
LABEL org.opencontainers.image.version="${VERSION}"
LABEL org.opencontainers.image.created="${BUILD_DATE}"
```

GHCR reads `.source` to link the image to the source repo on the
package page; renovate-bot / dependabot reads `.revision` and
`.version` to do its job; SBOM tooling reads everything.

The version-string discipline at build time:

```dockerfile
ARG VERSION=dev
ENV CGO_ENABLED=0
RUN go build \
    -ldflags="-X github.com/bartr/bartr-movies/internal/version.Version=${VERSION}" \
    -o /movies-api ./cmd/movies-api
```

Then in the binary:

```go
package version
var Version = "dev"  // overridden at build time via -ldflags
```

And the runtime endpoint per spec:

```
GET /version → {"version": "1.0.0", "revision": "abc1234"}
```

Now the image tag, the binary's reported version, and the git
commit are all linked. The operator can stand in front of `/version`
and answer "what's running?" with certainty.

#### Example

A `Makefile` target that wires the whole chain:

```makefile
VERSION  := $(shell cat VERSION)
GIT_SHA  := $(shell git rev-parse --short HEAD)
BUILT_AT := $(shell date -u +%Y-%m-%dT%H:%M:%SZ)
IMAGE    := ghcr.io/bartr/movies-api

.PHONY: build push

build:
	docker build \
	  --build-arg VERSION=$(VERSION) \
	  --label org.opencontainers.image.source=https://github.com/bartr/bartr-movies \
	  --label org.opencontainers.image.revision=$(GIT_SHA) \
	  --label org.opencontainers.image.version=$(VERSION) \
	  --label org.opencontainers.image.created=$(BUILT_AT) \
	  -t $(IMAGE):$(VERSION) \
	  -t $(IMAGE):$(GIT_SHA) \
	  src/

push:
	docker push $(IMAGE):$(VERSION)
	docker push $(IMAGE):$(GIT_SHA)
```

Two tags pushed for every build: the semver (`:1.0.0`) for human
deployments and the git SHA (`:abc1234`) for traceability and
emergency revert ("what was the prior commit running?").

**Never** `docker push $(IMAGE):latest` in this Makefile. If
operators want a "current dev" pointer, that's a CI job that
re-tags, not a deploy-time identifier.

#### Lab

1. In `repos/movies-bartr/`: read the `Makefile` and find how
   `VERSION` flows from the source file → `--build-arg` → ldflags
   → the binary's `version.Version` → the `/version` endpoint.
2. Build with `--build-arg VERSION=test-1` and the image tag
   `:test-1`. Deploy. Curl `/version`. Confirm the value matches.
3. **Demonstrate the `:latest` trap.** Tag two different builds
   both as `:latest` (locally is fine). Deploy a pod with
   `imagePullPolicy: IfNotPresent`. Scale up. Watch as pods may
   run different actual images.
4. **Demonstrate digest pinning.** Push the image, find the
   digest with `docker inspect --format='{{index .RepoDigests
   0}}' <image>`. Edit your deploy manifest to use
   `image: ...@sha256:...`. Re-push the tag pointing to a
   *different* build. Re-deploy. The digest-pinned manifest
   doesn't change; the tag-pinned one would.
5. **Read OCI labels.** `docker inspect --format='{{json
   .Config.Labels}}' movies-api:1.0.0 | jq`. Confirm the four
   labels exist.

#### Knowledge check

1. Three identities of an image — name them and which is
   immutable.
2. Why is `:latest` actively dangerous in a deploy manifest?
3. `imagePullPolicy: IfNotPresent` vs `: Always` — when does each
   apply?
4. Digest pinning (`@sha256:...`) — what threat does it foreclose
   that tag pinning doesn't?
5. The `VERSION` -> `ldflags` -> `/version` chain: what does it
   buy the operator?

---

### Module 7 — Multi-arch builds and `docker buildx`

> **Spec doesn't require this; enterprise does.** Apple Silicon
> dev laptops and arm64 cloud nodes (Graviton, Ampere, Azure
> Cobalt) make single-arch images a deploy-time landmine.

#### Concept

A normal `docker build` produces an image for the **host's
architecture**. Build on an amd64 laptop, get an amd64 image. Push
that to a registry, deploy to an arm64 node, and the pod fails
with `exec format error`.

**`docker buildx`** is the BuildKit-powered builder that handles
multi-arch. It uses one of two strategies:

1. **Emulation** (default, slow). QEMU emulates the target
   architecture on the host. Builds work but are 5–10× slower for
   the foreign arch. Fine for small builds, painful for large
   ones.
2. **Multiple native builders**. A "builder" that spans an
   amd64 node and an arm64 node. Each platform builds natively;
   buildx stitches the result into a multi-arch manifest. Fast,
   requires infra.

Either way, the output is an **OCI manifest list** (or "manifest
index") — a top-level manifest that points to per-arch manifests,
each pointing to per-arch image blobs. The registry stores all of
them under one tag. The runtime pulls the right one.

The minimal command:

```bash
docker buildx create --use --name multibuilder
docker buildx build \
    --platform linux/amd64,linux/arm64 \
    --tag ghcr.io/bartr/movies-api:1.0.0 \
    --push \
    .
```

Two things to know:

- **`--push` is required for multi-arch.** Local Docker daemons
  can't store manifest lists; the registry can. Without `--push`,
  buildx errors out.
- **Cross-compilation in the Dockerfile.** For Go, set `GOARCH` from
  the buildx-provided `TARGETARCH` build arg:

```dockerfile
FROM --platform=$BUILDPLATFORM golang:1.26-bookworm AS build
ARG TARGETOS TARGETARCH
ENV CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH
# ... rest of build
```

The `--platform=$BUILDPLATFORM` line is the key insight: **build
on the native architecture, cross-compile to the target.** A
Go build is fundamentally a cross-compile; running an emulated
amd64 toolchain on arm64 just to produce an arm64 binary is
~10× slower than running the native arm64 toolchain doing a
cross-compile.

#### Example

The movies-bartr Dockerfile, lightly modified for multi-arch:

```dockerfile
# syntax=docker/dockerfile:1.7

FROM --platform=$BUILDPLATFORM golang:1.26-bookworm AS build
WORKDIR /src

COPY go.mod go.sum ./
RUN go mod download

COPY cmd ./cmd
COPY internal ./internal

ARG TARGETOS TARGETARCH
ARG VERSION=1.0.0
ENV CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH

RUN go build -trimpath -ldflags "-s -w -X .../version.Version=${VERSION}" \
      -o /out/movies-api ./cmd/movies-api

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/movies-api /movies-api
USER 1000:1000
EXPOSE 8080
ENTRYPOINT ["/movies-api"]
```

Build:

```bash
docker buildx build \
    --platform linux/amd64,linux/arm64 \
    --tag ghcr.io/bartr/movies-api:1.0.0 \
    --push \
    src/
```

Result on the registry: one tag, manifest list pointing to two
images, each ~25MB.

#### Lab

1. `docker buildx ls` — see your current builders.
2. `docker buildx create --use --name lab-builder` — create a new
   one.
3. Build the movies-bartr Dockerfile single-arch (default).
   `docker image inspect <image>` and find `Architecture`.
4. Modify it as above; build multi-arch with `--platform
   linux/amd64,linux/arm64`. Push to a scratch registry (or use
   `--output type=oci,dest=out.tar` to avoid pushing).
5. **Demonstrate the emulation cost.** Time a multi-arch build
   *without* `--platform=$BUILDPLATFORM`. Then with. The first
   should be markedly slower for the foreign arch.
6. Inspect the manifest list with `docker buildx imagetools
   inspect <image>` — confirm both architectures listed.

#### Knowledge check

1. What is a manifest list (manifest index) and why does multi-arch
   need one?
2. Why does multi-arch require `--push` (or another non-local
   output)?
3. `BUILDPLATFORM` vs `TARGETPLATFORM` — what does each refer to?
4. For a Go service, why is `--platform=$BUILDPLATFORM` on the
   build stage *much* faster than letting buildx emulate?
5. Your team runs production on Graviton (arm64) but everyone
   develops on amd64 laptops. What single workflow change closes
   the "but it works on my machine" gap most cheaply?

---

### Module 8 — Reading images: history, inspect, dive

> **The capstone for this guide.** If the operator can't read an
> image, they ship images they don't understand. This module makes
> reading them routine.

#### Concept

Three tools, escalating in detail:

1. **`docker image inspect <image>`** — the metadata view. JSON
   blob with the config (env, user, entrypoint, labels, exposed
   ports), root filesystem layer hashes, architecture, OS,
   creation date. Use `--format='{{json .Config}}' | jq` to slice.
2. **`docker history <image>`** — the layer view. One row per
   layer, with the directive that created it, the size delta, and
   the creation time. Reads top-to-bottom from newest to oldest.
   First-pass diagnosis for "why is this image so big?"
3. **`dive <image>`** — interactive layer browser (third party,
   excellent). Click into each layer; see the actual files added,
   modified, removed. Calculates "wasted space" (files added in
   one layer and removed in a later one — still bloating the image
   because layers are append-only).

Two questions every read should answer:

1. **What's in this image that shouldn't be?** Source code in a
   runtime image, build tooling, `.git/`, cache files, secrets.
2. **What's *missing* that should be?** OCI labels, version
   metadata, non-root user, expected entrypoint.

The "wasted space" trap is the subtle one. The intuitive fix —
`RUN apt-get install foo && apt-get remove foo` in two `RUN`
directives — *doesn't actually shrink the image* because the
first RUN's layer still contains foo. The right pattern is one
RUN that does install, use, and clean up:

```dockerfile
# WRONG — foo is removed in layer 2 but still lives in layer 1.
RUN apt-get update && apt-get install -y foo
RUN do-something-with foo
RUN apt-get remove -y foo && apt-get autoremove -y

# RIGHT — install, use, clean up in one layer.
RUN apt-get update && apt-get install -y foo \
 && do-something-with foo \
 && apt-get remove -y foo && apt-get autoremove -y \
 && rm -rf /var/lib/apt/lists/*
```

#### Example

Reading the movies-bartr image cold:

```bash
$ docker history movies-api:1.0.0
IMAGE          CREATED         CREATED BY                                      SIZE
abc123def456   2 minutes ago   ENTRYPOINT ["/movies-api"]                      0B
<missing>      2 minutes ago   EXPOSE 8080                                     0B
<missing>      2 minutes ago   USER 1000:1000                                  0B
<missing>      2 minutes ago   COPY webv /webv-suites                          15kB
<missing>      2 minutes ago   COPY data /data                                 4.2MB
<missing>      2 minutes ago   COPY --from=build /out/webv /webv               7.8MB
<missing>      2 minutes ago   COPY --from=build /out/movies-api /movies-api   12.3MB
<missing>      3 weeks ago     /bin/sh -c #(nop) ENTRYPOINT ["..."]            0B
<missing>      3 weeks ago     /bin/sh -c #(nop) USER nonroot                  0B
<missing>      3 weeks ago     /bin/sh -c #(nop) COPY file:... in /            ~2MB
# ... distroless base layers
```

What this tells you:

- Total image: ~26MB. Two binaries + data + suites + distroless
  base.
- The `/data` directory is **4.2MB** — that's the JSON movie
  catalog, baked into the image per spec §5.2.
- The two Go binaries are ~20MB combined. They use the same
  internal packages, so `-ldflags="-s -w"` (strip symbols, strip
  debug info — see [study-guide-go.md](study-guide-go.md) Module
  10) is doing its job.
- No `.git`, no source, no go.mod, no go.sum, no build tooling.
- USER set, EXPOSE set, ENTRYPOINT set.

A `dive` view would confirm no wasted space (no install-then-remove
in different layers) and no surprising files in any layer.

#### Lab

1. `docker history movies-api:1.0.0` in `repos/movies-bartr/`.
   For each layer, identify which Dockerfile line created it.
2. `docker image inspect movies-api:1.0.0 --format='{{json
   .Config}}' | jq`. Find: `User`, `Entrypoint`,
   `WorkingDir`, `ExposedPorts`, `Labels`.
3. Install `dive` (`brew install dive` or
   `go install github.com/wagoodman/dive@latest`). Run `dive
   movies-api:1.0.0`. Tab through layers; find the `/data`
   layer; confirm contents.
4. **Build a deliberately-bloated image.** Write a Dockerfile
   with `RUN apt-get install -y curl && curl ... && apt-get
   remove -y curl`. Build. Inspect with `dive`. Find the wasted
   space.
5. Rewrite with the single-RUN pattern. Rebuild. Compare image
   sizes and dive views.
6. **Capstone**: pick a random public image (any service you've
   used). Run `dive`. Score it: minimal base? non-root? OCI
   labels? wasted space? `.git` artifacts? Generate a one-page
   findings list as if you were reviewing it for production
   adoption.

#### Knowledge check

1. The three tools (inspect, history, dive) — when do you reach
   for each?
2. Why does `RUN apt install foo` + `RUN apt remove foo` in
   separate layers *not* shrink the image?
3. The movies-bartr image is ~26MB. What accounts for the bulk?
4. "Wasted space" in `dive` — what does it mean and what's the
   fix pattern?
5. What does a healthy `docker history` look like? What's the
   first sign something's wrong?

---

## Per-release review

Same template as the other guides; see the
[Per-release review template section](study-guide-observability.md#per-release-review-template).

Container-specific addition: **every release verifies the image
size and the layer count haven't drifted.** A `:1.0.0` image that
was 25MB and is now 80MB on `:1.1.0` is a question, even if it
deployed cleanly. The agent will sometimes slip a debugging package
into the build stage that ends up in the runtime; the size check
catches it.

A simple Makefile target:

```makefile
.PHONY: check-image-size
check-image-size:
	@size=$$(docker image inspect movies-api:$(VERSION) --format='{{.Size}}'); \
	limit=$$((40 * 1024 * 1024)); \
	if [ "$$size" -gt "$$limit" ]; then \
	  echo "FAIL: image size $$size > $$limit"; exit 1; \
	fi; \
	echo "OK: image size $$size"
```

40MB is movies-bartr's specific budget (25MB image + 15MB headroom).
Each service picks its own; the discipline is that *there is a
number*.

## What this guide is and is not

- **Is:** the slice of containers that maps onto movies-spec §9 +
  §13 + the inner loop that depends on cache-hit rebuilds. Mental
  model, Dockerfile fluency, multi-stage, cache ordering,
  minimal-base trade-offs, tagging, multi-arch, image reading.
- **Is not:** a Docker reference manual. `docker network`,
  `docker volume`, `docker compose`, Swarm — covered elsewhere
  (B for local platform, D/E for orchestration). This guide is
  about *images and their construction*.
- **Is not:** a Kubernetes guide. The pod-spec security fields
  (`runAsNonRoot`, `readOnlyRootFilesystem`,
  `automountServiceAccountToken`) are named here because the
  image and the pod-spec must agree. The K8s side itself is in
  [study-guide-security.md](study-guide-security.md).
- **Is not:** a runtime alternative survey. `containerd`, `cri-o`,
  `podman`, `buildah`, `nerdctl` — all OCI-compliant, all
  interoperable with the Dockerfiles in this guide. Tool choice
  is a B-domain concern.

## Open questions

- **`buildah` / `podman` for rootless builds.** Worth its own
  module if/when the curriculum needs to be daemonless. Currently
  Docker is assumed.
- **Image signing (cosign) and admission verification.** Covered
  in [study-guide-security.md](study-guide-security.md) Module 3
  but not here. The container side of signing (signing as part of
  the build, attestations attached to the image) might belong
  here — open whether it goes here, in security, or in CI/CD.
- **The `chainguard/` family of images** are a credible
  alternative to distroless with a stronger CVE-response cadence.
  Worth flagging in the curriculum if/when they become the
  default.
- **WASM as a runtime target.** `wasmtime`, `spin`, container
  runtimes that execute WASM modules. Real but young. Out of
  scope until the spec grows to mention them.

## Status

- Not yet run end-to-end with any operator.
- Anchored in movies-bartr's real `src/Dockerfile` — every
  spec-bar pattern cited here is set correctly in the working
  repo.
- Unlocks for promotion to `methodology/` once at least one full
  run has produced a non-trivial Dockerfile review using Module 8's
  capstone and reported on what it caught.
