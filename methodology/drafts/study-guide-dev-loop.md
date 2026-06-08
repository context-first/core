# Study Guide — Dev Loop and Tooling (domain H)

> **DRAFT — NOT FOR PUBLICATION.** Ninth instance of the
> study-guide format. Scoped from domain H of
> [skills-inventory.md](skills-inventory.md). Built as the
> integration guide for the dev-loop threads that already run
> through containers (A), local platform (B), K8s core (C),
> Kustomize (D), GitOps (E), and observability (G).

## Why this exists

Five of the seven inventory H rows are already covered, in
pieces, in other guides:

- **H1** §12 inner loop — bits live in observability M6
  (probes + `/version`), kustomize M8 (inner-loop integration),
  containers M6 (VERSION → ldflags → `/version` chain).
- **H2** `make` / task runners — examples in containers M6,
  security M5, GitOps M5. No module on Makefile *discipline*.
- **H3** versioning — covered in containers M6. **No gap.**
  Not re-explained here.
- **H4** shell fluency — companion tools listed in local-platform
  M6, nothing teaches `jq` / `yq` / `xargs` / `awk` patterns.
- **H5** `git` beyond basics — methodology references FF-merge
  but no module teaches it.
- **H6** commit granularity — named gap in
  [sessions-and-skill-compounding.md](sessions-and-skill-compounding.md),
  not taught anywhere.
- **H7** dev containers — full module in local-platform M5.
  **No gap.** Not re-explained here.

What's missing is **the integration layer**: the module that
makes §12 a coherent walkthrough rather than five pieces
scattered across five guides; the module on Makefile discipline
rather than Makefile examples; the modules on `jq` / `git` /
commit granularity that the curriculum currently assumes you
already know.

That's what this guide is.

**What this guide is anchored in:**

- The movies-spec §12 inner loop — bump → build → deploy →
  verify → validate → inspect.
- movies-bartr's `Makefile`, `VERSION`, session-log, and
  per-session commit history as the worked example.
- [sessions-and-rpi.md](sessions-and-rpi.md) and
  [sessions-and-skill-compounding.md](sessions-and-skill-compounding.md)
  for the methodology framing the close ritual.

## How to use this guide

Same protocol as the other study guides. One curriculum-level
rule specific to this one: **the operator does every lab in the
terminal, not in an IDE GUI.** The IDE will hide the exact
behavior the labs teach (`git rebase -i` in a graphical merge
tool is not the same skill as `git rebase -i` in `$EDITOR`; the
graphical tool is fine *after* the muscle memory is there).

## Modules

### Module 1 — The §12 inner loop end-to-end (H1)

> Five guides each cover a slice. This module is the slice
> that ties them together.

#### Concept

The movies-spec §12 inner loop is six steps:

1. **Bump** — `VERSION` file gets a new semver.
2. **Build** — container image gets built with that version
   baked in via ldflags + OCI labels.
3. **Deploy** — Kustomize overlay's image tag gets updated;
   `kubectl apply -k` (or Flux reconciles) into the cluster.
4. **Verify** — pod reaches `Ready`; `/version` returns the
   exact string from step 1.
5. **Validate** — contract / smoke tests pass against the
   live in-cluster service over its real LoadBalancer.
6. **Inspect** — metrics in Grafana, logs in the log pane;
   confirm nothing regressed.

Each step has a "loop closed" signal. If any step's signal is
missing, the loop is not actually closed and the change is not
actually verified — it's just *present*.

| Step | "Loop closed" signal | If missing |
|---|---|---|
| Bump | `git diff VERSION` shows the new value | You can't tell what you shipped. |
| Build | Image tag in registry matches `VERSION`; `docker inspect` shows the OCI labels | You shipped a different image than you think. |
| Deploy | `kubectl rollout status` returns success; `kubectl get pods` shows new ReplicaSet only | Old pods still serving traffic. |
| Verify | `curl http://<lb>/version` returns the new string | Image is wrong, or `/version` is faked, or you're hitting the wrong service. |
| Validate | Contract suite returns 0 | The endpoint loads but the behavior regressed. |
| Inspect | Grafana shows the request you just made; logs show it; no new error-rate spike | Metrics or logs aren't wired; you'd ship a regression without knowing. |

#### Example

The movies-bartr Makefile encodes the loop as composable
targets. The full loop is one command:

```
$ make release-local
```

which internally is:

```
$ make bump          # bumps VERSION
$ make image         # docker build with ldflags + labels
$ make deploy        # kustomize edit set image + kubectl apply -k
$ make verify        # curl /version, assert it matches VERSION
$ make validate      # web-validate against the LB
$ make inspect       # open Grafana + tail logs
```

The targets compose because each one is small and each one
fails loudly. `make verify` is two lines:

```make
verify:
	@expected=$$(cat VERSION); actual=$$(curl -fsS http://movies.local/version | jq -r .version); \
	  [ "$$expected" = "$$actual" ] || { echo "MISMATCH: expected=$$expected actual=$$actual"; exit 1; }
```

That two-line target is the difference between "I shipped a
release" and "I shipped the release I think I shipped."

#### Lab

1. In movies-bartr, run `make release-local`. Observe each
   step's signal.
2. Break step 4 deliberately: edit the overlay to pin the
   *previous* image tag, then re-run `make deploy verify`.
   Predict what fails. Confirm.
3. Break step 6: stop Prometheus (`kubectl scale -n monitoring
   sts/prometheus-prometheus --replicas=0`). Re-run `make
   inspect`. The inspect step is the weakest link in most
   teams' loops — see how silently it fails.

#### Knowledge check

- Why is `make verify` two lines rather than one (`curl
  /version`)?
- A teammate says "I shipped 1.4.2 to dev." You check the
  cluster: `kubectl get pods -n movies-dev` shows 1.4.1
  pods. What three things could have gone wrong, and which
  §12 step's signal would have caught each?

---

### Module 2 — `make` discipline: wrappers vs hiders (H2)

> Spec §9 says Makefile is *optional*. That's deliberate.
> A bad Makefile is worse than no Makefile.

#### Concept

A Makefile target is **good** when it satisfies all three:

1. **Composable** — small enough to be a step in a larger
   target, not a 200-line god target.
2. **Inspectable** — `make -n <target>` (dry-run) shows the
   exact commands; nothing important is hidden.
3. **Idempotent or loud** — running it twice is either safe
   or fails clearly the second time.

A target is **bad** when it:

- Wraps a single command without adding value (`make get-pods:
  kubectl get pods` — just type `kubectl get pods`).
- Hides the actual command (`make deploy:` that runs 40 lines
  of shell, half of which set state you'll need to debug).
- Silently catches errors (`-` prefix on a critical command).
- Depends on environment state nobody documents (`make
  release` only works if `GITHUB_TOKEN` and `KUBECONFIG` and
  three other things are set, with no `@which` checks).

The rule: **Makefile targets exist to encode the inner loop's
multi-step plays, not to alias single commands.** If the
target is one command, type the command.

#### Example

Good — composable, inspectable, idempotent:

```make
.PHONY: image
image: ## Build container image with VERSION + git SHA baked in
	docker build \
	  --build-arg VERSION=$(VERSION) \
	  --build-arg SHA=$(GIT_SHA) \
	  --label org.opencontainers.image.version=$(VERSION) \
	  --label org.opencontainers.image.revision=$(GIT_SHA) \
	  -t $(IMAGE):$(VERSION) \
	  -t $(IMAGE):latest \
	  .
```

Bad — hides state, wraps trivially:

```make
.PHONY: k
k:
	@kubectl --context=$$(cat ~/.current-ctx) --namespace=$$(cat ~/.current-ns) $(ARGS)
```

That second example trades two characters of typing
(`kubectl`) for an invisible context-and-namespace
side-channel that nobody else on the team can debug. Use
`kubectx` / `kubens` for that — they make context visible in
the prompt.

#### Lab

1. Read movies-bartr's `Makefile` top to bottom. Mark each
   target as Composable / Inspectable / Idempotent — or note
   which property is missing.
2. Find a target that wraps a single command. Either delete
   it or justify why it earns its keep.
3. Add a `help` target (if missing) that auto-generates from
   `##` comments:

   ```make
   help: ## Print this help
       @awk 'BEGIN {FS = ":.*?## "} /^[a-zA-Z_-]+:.*?## / {printf "  \033[36m%-20s\033[0m %s\n", $$1, $$2}' $(MAKEFILE_LIST)
   ```

#### Knowledge check

- When should an inner-loop step be a Makefile target vs a
  shell function in your `.zshrc` vs a checked-in script in
  `scripts/`?
- A teammate sends a PR adding `make all: image deploy
  verify validate inspect`. Why is `all` a worse name than
  `release-local`?

---

### Module 3 — Shell fluency for the curriculum (H4)

> The cost of not knowing `jq` is paid every session.

#### Concept

The four tools the curriculum keeps reaching for:

| Tool | What it does | Pays for itself when |
|---|---|---|
| `jq` | Filter / transform JSON | `kubectl get -o json`, `curl` against any JSON API, parsing GitHub API responses, reading Prometheus query results. |
| `yq` | Same shape as `jq`, for YAML | Reading Kustomize output, inspecting Helm-rendered manifests, editing a single field in a values file from a script. |
| `xargs` | Turn lines on stdin into command arguments | "Delete every pod in `Evicted` state"; "build every image in this list"; "scale every deployment in this namespace to zero." |
| `awk` | Field-based text processing | Anything tabular — `kubectl get pods` columns, `df`, `ps`, log lines with consistent shape. `awk '{print $2}'` is most of it. |

These four cover ~95% of the shell work the curriculum's
inner loop generates. The patterns to actually internalize:

**`jq` — the five patterns**

```sh
# 1. Pluck one field
curl -fsS http://movies.local/version | jq -r .version

# 2. Pluck many fields as TSV
kubectl get pods -o json | jq -r '.items[] | [.metadata.name, .status.phase] | @tsv'

# 3. Filter
kubectl get pods -o json | jq -r '.items[] | select(.status.phase != "Running") | .metadata.name'

# 4. Construct an object
echo '{"a":1,"b":2}' | jq '{sum: (.a + .b)}'

# 5. Compact for piping
kubectl get pods -o json | jq -c '.items[] | {name: .metadata.name, image: .spec.containers[0].image}'
```

**`yq` — same shape, for YAML**

```sh
yq '.spec.template.spec.containers[0].image' deploy/movies/base/deployment.yaml
yq '.spec.template.spec.containers[].name' deploy/movies/base/deployment.yaml
yq -i '.spec.replicas = 3' deploy/movies/overlays/dev/replicas.yaml
```

**`xargs` — the three forms**

```sh
# Simple substitution
echo "a b c" | xargs -n1 echo

# Explicit placeholder ({}) — required when the argument isn't at the end
kubectl get pods -o name | xargs -I{} kubectl describe {}

# Parallel — the killer feature
cat image-list.txt | xargs -P4 -n1 docker pull
```

**`awk` — the one-liner that earns its keep**

```sh
# Print the second column
kubectl get pods | awk '{print $2}'

# Filter then print
kubectl get pods | awk '$3 != "Running" {print $1, $3}'

# Sum a column
df | awk 'NR>1 {sum += $3} END {print sum}'
```

The rule: **if you find yourself reaching for Python to parse
a one-liner's output, you're missing one of these four
tools.** Python is right for 50-line scripts; for "what's the
image tag on every pod in this namespace," `jq` is right.

#### Example

Reading the cold-cluster state from K8s-core Module 12 is
mostly `jq` work:

```sh
# Every workload + its image + its replica count
kubectl get deploy -A -o json | \
  jq -r '.items[] | [.metadata.namespace, .metadata.name, .spec.template.spec.containers[0].image, .status.readyReplicas] | @tsv' | \
  column -t

# Every pod that isn't healthy
kubectl get pods -A -o json | \
  jq -r '.items[] | select(.status.phase != "Running" and .status.phase != "Succeeded") | [.metadata.namespace, .metadata.name, .status.phase] | @tsv'

# The exact image SHA Flux deployed (vs what the overlay claims)
kubectl get deploy -n movies movies-api -o json | \
  jq -r '.spec.template.spec.containers[0].image'
```

That's three queries answering three operator questions in one
shell, no scripts, no GUI.

#### Lab

1. From the movies-bartr cluster, write one `jq` one-liner
   that prints every Service's name + cluster IP + port.
2. Write one `yq` command that changes the image tag in the
   `dev` overlay's `kustomization.yaml` without opening an
   editor.
3. Write one `xargs -P4` command that pulls every image
   referenced by every Deployment in the cluster, in
   parallel.
4. Write one `awk` one-liner that, given `kubectl top pods
   -A`, prints the top five memory consumers.

#### Knowledge check

- What's the difference between `jq -r` and `jq` (no flag)?
  When does the difference bite you?
- `xargs` vs a `for` loop — when is each correct?
- Why is `kubectl get pods | awk '{print $1}'` fragile,
  and what's the structured alternative?

---

### Module 4 — `git` for the methodology (H5)

> The methodology's close ritual assumes FF-merge. Most
> operators don't know what FF-merge is, or how to keep it
> achievable across a multi-session arc.

#### Concept

Six git skills the curriculum assumes you have:

1. **`git rebase main` vs `git merge main`** — rebase
   replays your commits on top of `main`; merge creates a
   merge commit. Rebase is right *during* a feature branch
   (keeps history linear). Merge is right when integrating a
   long-lived shared branch.
2. **Fast-forward merge** — when your branch is a strict
   superset of `main`'s commits (no divergence), git can
   "fast-forward" `main`'s pointer to your branch's tip
   without a merge commit. **FF-merge is the curriculum's
   close ritual** — every session ends with `gh pr merge
   --rebase --delete-branch`, which replays + FFs.
3. **Tagging** — `git tag -a v1.4.2 -m "release 1.4.2"` then
   `git push --tags`. Tags are the artifact `make verify`
   compares `/version` against.
4. **`git bisect`** — when "it worked yesterday, it doesn't
   today," bisect finds the breaking commit in O(log n)
   steps. The reason commit granularity matters (Module 5):
   bisecting 50 tiny commits gives you a precise answer;
   bisecting 5 giant commits tells you which afternoon
   broke things.
5. **`--no-pager`** — `git --no-pager log --oneline -20`.
   Scripts and Makefiles need git output without the
   interactive pager. Every git command in a Makefile uses
   `--no-pager`.
6. **`git reflog`** — the safety net. "I rebased and lost a
   commit." `git reflog` shows every HEAD position for the
   last 90 days; `git reset --hard HEAD@{5}` recovers.
   Knowing reflog exists makes rebase fearless.

The rule the methodology builds on: **never `--squash` a
merge.** Squash destroys per-commit history, which kills
`bisect`, kills the signal "what shipped together," and
collapses the work down to one massive change nobody can
review. The curriculum-wide merge is `--rebase` (replays
commits + FFs), and `--merge` is the fallback when rebase
won't go cleanly. Squash is never the answer.

#### Example

A clean session arc, from branch creation to FF-merge:

```sh
# Branch from main
$ git switch -c feat/movies-readyz

# ...work, commit small, push as you go...
$ git add internal/healthz/readyz.go
$ git commit -m "readyz: return 503 when db unavailable"
$ git add internal/healthz/readyz_test.go
$ git commit -m "readyz: test 200 / 503 paths"

# Before opening PR, rebase onto current main so FF is possible
$ git fetch origin
$ git rebase origin/main
# (resolve conflicts if any)

# Push, open PR
$ git push -u origin feat/movies-readyz
$ gh pr create --fill

# After review/CI green: merge with rebase + delete branch
$ gh pr merge --rebase --delete-branch

# Tag the release if this closed a release-worthy arc
$ git switch main && git pull
$ git tag -a v1.4.2 -m "movies-api 1.4.2"
$ git push --tags
```

Six commits, FF-merged, tagged. `git log --oneline main` is a
flat history with one commit per logical change. `git bisect`
on this history pinpoints regressions to the exact line.

The anti-pattern:

```sh
$ git add .
$ git commit -m "wip session 3"
$ git push
$ gh pr merge --squash    # ← curriculum forbids this
```

One commit covers eight unrelated changes; bisect is useless;
the PR is unreviewable.

#### Lab

1. In a scratch repo, create a branch with four commits.
   Run `git log --graph --oneline --all` before and after `gh
   pr merge --rebase` (vs `--merge`, vs `--squash`).
   Compare the three resulting histories.
2. Practice `git bisect` on a deliberately-broken-then-fixed
   branch. Time how long it takes to find the bad commit on
   a 5-commit branch vs a 1-commit-squashed branch (you
   can't on the squash — that's the point).
3. Do `git rebase -i HEAD~5` and reorder, squash, and
   reword. Then `git reflog` and reset back to before the
   rebase. This is the safety net that makes (2) safe.

#### Knowledge check

- Why does the curriculum forbid `--squash` even though
  GitHub's UI defaults to it for many repos?
- A teammate says "the PR is too big to review." Three
  causes — what are they, and which one does `git rebase -i`
  fix vs which one does Module 5 fix?
- What does `git reflog` show that `git log` doesn't?

---

### Module 5 — Commit granularity (H6)

> Named gap from the Matt-v1 review. The single most
> teachable habit in the curriculum.

#### Concept

A **good commit** has three properties:

1. **One reason** — answers one question. "Why does this
   change exist?" has one sentence answer.
2. **Independently revertable** — `git revert <sha>` undoes
   *this* change without touching anything unrelated.
3. **Independently understandable** — a reviewer reading
   only the diff + message understands what changed and
   why.

Most "WIP" / "fix" / "stuff" commits violate all three.
They're not commits, they're saves.

The session-level pattern the curriculum teaches:

| Inside a session | What lands in `main` |
|---|---|
| 8–20 small commits as you work | Same 8–20 commits, FF-merged via `--rebase` |
| Some are "checkpoint" commits — fine while branching | None are checkpoints by the time you merge — you `rebase -i` to clean them up |
| Some are out-of-order discoveries — fine | Reordered into a coherent narrative before opening the PR |

The session-close ritual includes a `git rebase -i
origin/main` cleanup step *before* the PR opens. That step
takes 10 minutes and saves the reviewer an hour.

#### Example

**Matt-v1 commit history (the named gap):** the session
landed as one giant commit per session — "session 3 work,"
"session 4 work." Reviewing it required reading every
changed file from scratch. Reverting any single decision
meant unpicking the whole session by hand. `git bisect`
across the run pointed to "session 3" — useless precision.

**The corrective pattern, on the same work:**

```
$ git log --oneline feat/movies-readyz
8a3f1c2 readyz: document the 503 contract in /version response
7b2e9a1 readyz: tighten DB ping timeout to 200ms
6d4c1f3 readyz: test 200 / 503 paths
5e9b2a4 readyz: return 503 when db unavailable
4f1a8c5 readyz: extract checker interface
3c2d9b6 deployment: wire readinessProbe to /readyz
2b8e1f7 docs: spec §8.1 readyz contract note
```

Seven commits, each independently revertable, each with one
reason. `git bisect` across this history finds bugs to the
line. Reviewers can read one commit at a time. The PR
description is the same length but actually means something.

The mechanics:

```sh
# As you work — commit anything that compiles
$ git commit -am "checkpoint: readyz returns 503"

# Before opening PR — interactive rebase to clean up
$ git rebase -i origin/main
# In $EDITOR: reorder, squash trivial checkpoints into the
# logical commits they belong to, reword messages.
# Result: 7 clean commits where there were 14 messy ones.

$ git push --force-with-lease    # safe force-push, your branch only
$ gh pr create --fill
```

`--force-with-lease` (not `--force`) is the safe version: it
refuses if anyone else pushed to your branch in the
meantime. Use it whenever rebasing a published feature
branch.

#### Lab

1. Open the movies-bartr commit history for one session.
   Count commits in the session and rate each by the
   three-properties test (one reason / revertable /
   understandable).
2. Take a real WIP branch you have. Run `git rebase -i
   origin/main` and reorganize it into clean commits.
   Notice what's hard — that's the gap to close.
3. Write a one-paragraph PR description from the *commits*
   (read the commit messages, summarize). If you can't, the
   commits aren't clean enough.

#### Knowledge check

- Why is "one reason per commit" stronger than "small
  commits"?
- A teammate's PR has 40 commits, all titled "fix." What's
  the conversation? (Not "squash it" — that loses the
  signal. What does cleaning it up actually look like?)
- The methodology's close ritual produces 8–20 commits per
  session. What goes wrong at 1 commit per session? What
  goes wrong at 50?

---

## Per-release review

Per the curriculum-wide template in
[study-guide-observability.md](study-guide-observability.md):
every release runs the cold-cluster read (K8s-core M12), the
security capstone (security M7), the image-size budget check
(containers per-release), and **this guide's residual: the
inner-loop signal check** — every §12 step's "loop closed"
signal fires for this release, including `inspect`. The
inspect step is the silent-failure mode the per-release ritual
exists to catch.

## What this guide is

- The integration module for §12 inner loop, plus the four
  residual gaps (Makefile discipline, shell fluency, git
  fluency, commit granularity).
- Anchored in movies-bartr's Makefile + commit history as the
  worked example.
- The capstone for the dev-loop skills the rest of the
  curriculum assumes you already have.

## What this guide is not

- Not a re-teach of versioning (containers M6) or dev
  containers (local-platform M5) — those are covered, this
  guide cross-links.
- Not a complete `git` reference — six skills the
  methodology requires, no more.
- Not a comprehensive shell tutorial — four tools, the
  patterns that pay back in this curriculum.

## Open questions

1. Does Module 1 want a worked example of the loop *failing*
   at every step, so the operator sees what each signal
   feels like in the breach? Currently the lab does this
   piecemeal.
2. Module 3's `jq` patterns — is five the right count, or
   does a sixth (recursive descent, `..`) earn a slot for
   reading deeply-nested operator CRDs?
3. Module 5 is the most "soft skill"-shaped module in the
   curriculum. Is the lab strong enough? Should it require
   the operator to *pair* — clean up someone else's branch,
   not their own?

## Status

DRAFT. Not promoted to `methodology/` until at least one
operator runs this end-to-end on a real session arc and
confirms the labs produce the claimed behavior change
(commit-granularity especially — this is the hardest habit
to install).
