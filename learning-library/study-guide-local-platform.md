# Study Guide — Local Platform (domain B)

> **DRAFT — NOT FOR PUBLICATION.** Seventh instance of the
> study-guide format. Scoped from domain B of
> [skills-inventory.md](skills-inventory.md). Anchored in
> [github.com/bartr/wsl](https://github.com/bartr/wsl) (the
> opinionated baseline used for WSL and for DigitalOcean droplets
> via SSH) and the `.devcontainer/` patterns used across the
> bartr-owned MIT repos.

## Why this exists

The local development platform is the layer everything else sits
on. Get it wrong and every other skill in the curriculum is taxed
— slow rebuilds, flaky networking, "works on my machine," lost
weekends reinstalling. Get it right and the operator forgets the
platform exists, which is the goal.

The agent's defaults are wrong on this one in two directions:

1. **It will install whatever it sees first.** Docker Desktop on
   Mac, native Windows builds of every tool on Windows, system
   Python on Linux. None of those are wrong; none of them are the
   baseline a team can share.
2. **It treats the dev machine as a one-off.** No reproducibility,
   no snapshot, no "spin a fresh environment in 10 minutes."
   That's exactly the opposite of what spec-shaped work requires:
   the dev environment should be **infrastructure-as-code**, same
   as the deploy.

This guide names the baseline and the alternatives, and gives the
operator a way to reproduce them.

**What this guide is anchored in:**

- [github.com/bartr/wsl](https://github.com/bartr/wsl) — opinionated
  WSL2 setup (also reused on DigitalOcean droplets and Microsoft
  Dev Box). `install.sh` + `scripts/base.sh` + `scripts/config.sh`.
- The WSL improvements announced at Build 2026 (June 2026) — treat
  the bartr/wsl baseline as a moving target that absorbs them as
  they ship.
- The `.devcontainer/` pattern (movies-bartr and other repos).
- The shell baseline (`zsh` + `.oh-my-zsh`) — now the macOS default
  (since Catalina, 2019) and the curriculum default everywhere
  else.

## How to use this guide

Same protocol as the other study guides. One curriculum-level rule
specific to this one: **the operator picks one primary platform
and one secondary.** Trying to be fluent on three at once means
fluent on none. The recommendations below name the trade-offs;
each operator picks their primary and treats the others as
"can-read, can-help-someone."

## Modules

### Module 1 — Linux is the floor; everything else is emulation

> **The mental model that makes the rest of the choices make
> sense.** Misunderstanding this is the source of most "but it
> works on Linux" headaches.

#### Concept

Every container runtime, every Kubernetes node, every production
target the curriculum touches is **Linux**. macOS and Windows are
*developer convenience layers* on top of a Linux VM. Always.

- **Docker Desktop for Mac** — a Linux VM (formerly HyperKit, now
  Apple Virtualization Framework) running `containerd`. The
  `docker` CLI on Mac talks to the daemon *inside* the VM.
- **Docker Desktop for Windows** — same idea, the VM is either
  WSL2's Linux kernel or Hyper-V depending on configuration.
- **WSL2** — a real Linux kernel running in a lightweight Hyper-V
  VM, with deep integration into Windows (shared clipboard, GPU
  passthrough, filesystem bridges).
- **Codespaces / Dev Box / cloud droplets** — actual Linux
  machines, no emulation.

The four things that follow from the model:

1. **Filesystem perf is asymmetric.** Crossing the Mac/Win ↔ Linux
   boundary is slow. Keep source code, build caches, and
   `node_modules` on the Linux side; cross only for editing and
   browser preview.
2. **Networking has three address spaces** — the host OS, the VM,
   and any container inside the VM. Knowing which `localhost` is
   which is half the battle (Module 7).
3. **Resource limits are real.** The VM has a configured CPU /
   memory / disk size. Big builds OOM inside the VM even though
   the host has 64GB free.
4. **Architecture matters.** Apple Silicon Macs run an arm64
   Linux VM; most production runs amd64 (or Graviton arm64). The
   "but it works on my Mac" failure mode is usually
   architecture-confusion.

The recommendation that follows:

| Host OS | Primary recommendation | Why |
|---|---|---|
| Windows | **WSL2 + the [bartr/wsl](https://github.com/bartr/wsl) baseline.** As of Build 2026, **WSL Containers** (public preview) is the native Linux-container runtime; Docker Engine inside the distro still works and is the safer default until WSL Containers reaches GA. | Real Linux kernel, fast, snapshot-able, integrates with VS Code via Remote-WSL. Build 2026 added a Microsoft-supported native container runtime that sidesteps the Docker Desktop licensing question. |
| macOS | **OrbStack** (or Colima) as the container runtime + zsh + the curriculum shell baseline; **UTM** or Multipass if you need a full Linux VM for kernel-level work | Docker Desktop works; the alternatives are faster, lighter, and avoid the licensing footgun (Module 3). |
| Linux | **Native Docker Engine + zsh + the bartr/wsl `scripts/base.sh` adapted** | The reference. What everything else is emulating. |
| Anything | **GitHub Codespaces** as a sidearm | A 10-second fallback when local is broken or you're on the wrong machine (Module 5). |

#### The non-negotiable for Windows developers

> **You have to learn Linux. You have to get comfortable in
> bash/zsh.** This is the part there is no path around, and the
> agent will never tell you so.

Every Kubernetes example you'll read \u2014 official docs, blog posts,
StackOverflow answers, the spec's inner loop in
[study-guide-observability.md](study-guide-observability.md),
every other guide in this curriculum \u2014 assumes a POSIX shell.
`kubectl get pods -o jsonpath='{.items[*].metadata.name}' | xargs
-I{} kubectl logs {}` is what the ecosystem speaks. PowerShell can
do some of it; the entire ecosystem does not write examples for
it. **Nobody uses it for K8s work.** That's not a value judgment;
it's an observed fact about where the docs live.

The failure mode shows up at every onboarding: a Windows developer
clinging to PowerShell, line-by-line translating bash one-liners
that should have taken 30 seconds into 20-minute PowerShell
puzzles. The same developer trying to read `for f in **/*.yaml;
do kubectl apply -f $f; done` from a doc and having to
*translate* it instead of *run* it. Multiply by every
debugging session, every lab in this curriculum, every Stack
Overflow answer.

The fix is unromantic and one-time-cost:

1. **Make WSL2 your default working environment** (Module 2).
   Open VS Code into a WSL distro, not into a Windows folder.
   Open your terminal into zsh inside WSL, not into PowerShell.
2. **Spend a week being uncomfortable.** `ls / ll / cd / cp / mv
   / cat / less / grep / find / xargs / awk / sed / jq / yq / tar
   / chmod / chown` \u2014 these are the verbs. The first week is
   slow; by the second week they're faster than the PowerShell
   equivalents you knew.
3. **Stop translating.** When you read a K8s doc, run the
   commands *as written* in your WSL zsh. The point of the WSL
   baseline (Module 2) is to make this friction-free.
4. **Keep PowerShell for Windows-side work** \u2014 managing WSL
   itself (`wsl --export`), driving Active Directory, automating
   the Windows OS. That's where it earns its keep. K8s, Docker,
   Go, Node, Python, every observability tool, every CI/CD
   example \u2014 those happen in a POSIX shell.

This isn't anti-Windows. The Windows operating system at the
desk \u2014 fine. The Windows *shell* for cluster work \u2014 a tax on every
hour. The curriculum's recommendation is structural, not cultural:
**K8s == bash. Get comfortable with the switch.**

#### Example

The "where does my code live" mistake on Windows + WSL2:

```
C:\Users\bart\projects\movies\           ← Windows filesystem, mounted into WSL as /mnt/c/...
\\wsl$\Ubuntu\home\bart\projects\movies\ ← Linux filesystem, native ext4
```

Editing `/mnt/c/...` from inside WSL works. `npm install` on
`/mnt/c/...` takes 10× longer than on `~/projects/movies/`. The
right pattern: **always `git clone` into the Linux home
directory**; let VS Code's Remote-WSL bridge handle the editor
view.

#### Lab

1. On your primary platform, find where Docker is actually
   running. On Mac: `docker context ls` and `docker info | grep
   -i operating`. On Windows: same. The "operating system" the
   daemon reports is the VM's Linux, not your host.
2. **Demonstrate the filesystem-perf trap.** On Mac or Windows,
   clone a medium repo (e.g., a Go service) into a bind-mounted
   directory. Run `time go build ./...`. Then move it inside the
   VM (or to a Docker named volume). Re-run. Compare.
3. On Mac: `docker run --rm alpine uname -m`. On Apple Silicon
   this prints `aarch64`; on Intel Mac it prints `x86_64`. Run
   `docker run --rm --platform=linux/amd64 alpine uname -m`.
   Note the QEMU emulation kick in (slower).
4. Open `top` (or Activity Monitor / Task Manager). Find the
   process that *is* the Linux VM. Note its memory footprint.
   That's not "Docker"; that's a whole Linux kernel running
   alongside your host OS.

#### Knowledge check

1. "Docker on Mac" — what's actually running where?
2. Two reasons keeping source code on the Linux side of WSL is
   the right default.
3. Apple Silicon + a production amd64 deploy target: what does
   the operator's workflow need to look like to avoid surprises?
4. The Linux VM has a memory cap. Why does this matter for
   builds that "should have plenty of RAM"?
5. Three address spaces for `localhost` on a Mac/Win developer
   machine — name them.
6. **Why "K8s == bash" is structural, not cultural.** What's the
   concrete cost paid by a Windows developer who keeps trying
   to do cluster work from PowerShell? Name three places in a
   normal day where it shows up.

---

### Module 2 — WSL2 on Windows: the bartr/wsl baseline + Build 2026

> **If you are on Windows doing this work, live in WSL2.** Don't
> live in Docker Desktop's Windows-host emulation; don't live in
> PowerShell with Windows-native binaries. The Build 2026 (June
> 2–3, San Francisco) announcements pushed WSL from "first-class
> developer platform" to "native container host for Windows,"
> which materially changes the runtime-choice conversation in
> Module 3.

#### Concept

WSL2 is **a real Linux kernel** running in a managed Hyper-V VM,
with deep integration into Windows:

- One command (`wsl --install -d ubuntu`) gets you Ubuntu.
- VS Code's Remote-WSL extension makes the editor run on the
  Windows side and the language servers / terminals / debuggers
  run on the Linux side. No SSH dance.
- WSL distributions are **snapshot-able**: `wsl --export` produces
  a tarball, `wsl --import` instantiates a new distro from one. A
  fully-configured dev environment is a 3-8 GB tarball you can
  back up, share, and restore.
- Multiple distros run side by side. Run `ubuntu-24.04` for one
  project, `bartr` (your custom snapshot) for another, `alpine`
  for a quick experiment.
- **WSL is open source** (Build 2025), with the project taking
  200+ PRs per month from external contributors. The roadmap is
  visible; the velocity is real.

**Build 2026 (June 2–3, 2026) — the changes worth knowing:**

1. **WSL Containers (public preview).** A new dedicated executable
   + CLI + API for building, running, and managing **Linux
   containers natively on Windows through WSL**, without Docker
   Desktop, OrbStack, Podman Desktop, or any other third-party
   runtime. For curriculum purposes this is the headline: on
   Windows, the WSL baseline + WSL Containers covers the
   container-runtime slot that Docker Desktop used to occupy. The
   licensing-and-org-policy conversation around Docker Desktop on
   Windows largely goes away.
2. **Enterprise management for WSL Containers.** IT admins can
   define policies for container image sources, monitor which
   Linux containers run on dev machines, and gate how containers
   interact with the Windows host. Native Windows apps can also
   launch Linux containers programmatically via a new API. The
   operational implication: at enterprise scale, this is the
   policy-compliant alternative to Docker Desktop.
3. **WSL AI App Catalog + Agent Sandboxes.** A curated catalog
   of agents validated against Microsoft's Agent Governance
   requirements; IT can constrain WSL Agent Sandboxes to
   catalog-approved agents only, with unapproved agents running
   in a stricter "untrusted" enclave (no network unless
   explicitly elevated). Out of scope for the spec floor; named
   here because it's the direction the curriculum's MCP-and-agent
   work intersects with enterprise-controlled environments.
4. **Coreutils for Windows (GA, companion announcement).**
   Microsoft's Rust-based reimplementation of GNU coreutils
   (built on `uutils`), now native on Windows. Closes the
   no-`grep` / no-`sed` / no-`cat` gap on Windows hosts without
   Git Bash, Cygwin, or even a WSL distro. *Does not replace
   WSL* for curriculum work — a real Linux kernel and a real
   filesystem still matter — but removes a class of friction
   when the operator needs to script *on the Windows side*.

Primary source: Pavan Davuluri's Windows Developer blog post
([blogs.windows.com/windowsdeveloper/2026/06/02/build-2026-furthering-windows-as-the-trusted-platform-for-development](https://blogs.windows.com/windowsdeveloper/2026/06/02/build-2026-furthering-windows-as-the-trusted-platform-for-development/)).
Treat the WSL Containers feature as **public preview** —
production adoption needs the operator to verify it against
current docs before betting workflows on it.

The **bartr/wsl** baseline (the curriculum reference) does all of
the following from a single `sudo ./install.sh`:

1. Adds the user to `sudo`, `admin`, and `docker` groups; sets
   passwordless sudo for the dev user.
2. Installs Docker Engine, Docker Compose, `kubectl`, `helm`,
   common CLI tools (`jq`, `yq`, `gh`, `ripgrep`, etc.).
3. Sets `zsh` as the default shell, installs `oh-my-zsh` (Module
   6).
4. Configures git (with the user's name + email).
5. Hands off to `scripts/config.sh` for any user-specific
   tweaks.

The reproducibility property: **the entire dev environment is a
git repo plus one `install.sh`**. Re-imaging a laptop is a 30-
minute exercise instead of a 3-day one.

The snapshot property: after `install.sh` finishes successfully,
**immediately export to a tarball**. From then on, "reset my dev
machine" is `wsl --unregister bartr && wsl --import bartr ...
bartr.tar`. Two commands, ~30 seconds.

> **And install Windows Terminal.** Not optional. The legacy
> Windows console host is the single worst part of working on
> Windows; Windows Terminal fixes it. Tabs, splits, GPU-accelerated
> rendering, proper Unicode, configurable per-profile (one tab for
> PowerShell, one for your WSL distro, one for Azure Cloud Shell),
> and \u2014 the one that pays for the install on its own \u2014
> **`Ctrl-Shift-C` / `Ctrl-Shift-V` copy-paste that just works**.
> No more "right-click-to-paste-but-only-sometimes" from the old
> conhost. Install from the Microsoft Store or `winget install
> Microsoft.WindowsTerminal`; set it as the default terminal app
> in Windows Settings \u2192 Privacy & Security \u2192 For Developers \u2192
> Terminal.

#### Example

The full bartr/wsl bring-up on a fresh Windows machine. Open
**Windows Terminal** \u2192 PowerShell tab:

```powershell
# Prereqs: VS Code installed, Windows Terminal installed.
winget install Microsoft.WindowsTerminal           # if not already there
code --install-extension ms-vscode-remote.vscode-remote-extensionpack

# WSL itself.
wsl --update
wsl --set-default-version 2
wsl --install -d ubuntu

# Inside the new Ubuntu shell: set git identity, then exit.
git config --global user.name "your-name"
git config --global user.email "you@example.com"
exit

# Clone the baseline and run it.
git clone https://github.com/bartr/wsl
cd wsl
wsl -- sudo ./install.sh

# Snapshot it.
wsl -t ubuntu
wsl --export ubuntu bartr-baseline.tar
wsl --unregister ubuntu
wsl --import bartr C:\WSL\bartr bartr-baseline.tar

# Open VS Code into a project, attached to the new distro.
wsl -- code wsl
```

After this, "spin a new Ubuntu identical to mine" is `wsl
--import some-name C:\WSL\some-name bartr-baseline.tar`. Repeat
per project if you want strong isolation.

#### Lab

1. On a Windows machine (or Microsoft Dev Box, which supports
   nested virtualization): follow the bartr/wsl README end-to-
   end. Time yourself.
2. After install completes and you've verified Docker works
   (`docker run --rm hello-world`), **export to a tarball
   immediately**. This is the "golden image" you'll reset to
   later.
3. **Break it on purpose.** Inside the distro: `sudo rm -rf
   /usr/local/bin/docker`. Confirm it's broken.
4. From PowerShell: `wsl --unregister bartr && wsl --import
   bartr C:\WSL\bartr bartr-baseline.tar`. Re-attach VS Code.
   Confirm Docker works again. *Total time: ~60 seconds.*
5. From VS Code's Remote-WSL: open a folder inside the distro
   (e.g., `~/projects/movies-bartr`). Open a terminal.
   Confirm: editor on Windows, terminal on Linux, files on
   Linux, language server on Linux.
6. Open Task Manager → Performance → CPU. Run a sizable build
   inside WSL (`cd repos/movies-bartr/src && go build ./...`).
   Watch the host CPU light up. That's a real Linux kernel
   scheduling on your real cores.

#### Knowledge check

1. Why does the bartr/wsl baseline live in a git repo + an
   `install.sh` instead of "here's a list of things to install"?
2. WSL `--export` / `--import` — what does each do, and what's
   the operational property the pair gives you?
3. Multiple WSL distros, side by side — why might you use
   per-project distros?
4. Filesystem perf: which side (Windows or Linux) should hold
   the source code, and why?
5. VS Code Remote-WSL puts the editor on Windows and the
   language servers on Linux. What concrete problem does that
   solve that the alternatives (full-VM RDP, X-forwarding, plain
   SSH) don't?
6. **Build 2026 — WSL Containers (public preview).** What slot
   does it occupy that Docker Desktop used to? What's the
   enterprise-management story that ships with it, and why does
   that matter for org-policy-constrained environments?
7. **Coreutils for Windows (GA, Build 2026).** What gap does it
   close on the Windows side, and why does it *not* replace WSL
   for curriculum work?

---

### Module 3 — macOS: the zsh-default era + container runtime choice

> **macOS Catalina (2019) made zsh the default login shell.**
> That's a happy alignment with the curriculum baseline — the
> shell is one fewer choice to argue about. The container runtime
> is the choice that still matters.

#### Concept

On macOS, the operator has two independent decisions:

**1. Shell.** zsh is the default since 2019. The curriculum
recommendation is to keep it and add `oh-my-zsh` (Module 6) for
the configuration baseline. No bash, no fish, no nushell for
team-shared work — uniformity matters more than personal
preference for the curriculum's tooling.

**2. Container runtime.** Three credible options:

| Runtime | License | Notes |
|---|---|---|
| **Docker Desktop** | Paid for orgs > 250 employees / $10M revenue (as of writing) | The default; works fine; pays a per-seat license fee at any real company. |
| **OrbStack** | Paid (commercial), free for personal | Faster than Docker Desktop, lighter on resources, also runs Linux VMs (overlapping with Module 4). The **current curriculum recommendation** on Mac for users who'd otherwise need a Docker Desktop license. |
| **Colima** | Free, OSS | Backs the standard `docker` CLI with Lima (Linux VMs). Slightly less polished; no license question. |
| Rancher Desktop, Podman Desktop | Free, OSS | Both real and used at scale; teams pick one. Worth knowing they exist. |

The licensing reality matters because it's the thing that drives
team-wide tool choices, and the operator should not be surprised
when their employer says "no Docker Desktop." (See B6 in the
inventory.)

> **Note (Build 2026):** On **Windows**, the licensing question
> just got a Microsoft-supported answer in **WSL Containers**
> (Module 2). On **macOS** there is no equivalent yet \u2014 you still
> pick from the table above. If you support a mixed-OS team, the
> macOS half of the team likely settles on OrbStack or Colima
> while the Windows half settles on WSL Containers (once GA) or
> WSL + Docker Engine.

**3. Optional: a full Linux VM on the Mac.** Sometimes you need
kernel access, real systemd, or a Linux environment that
matches production node-for-node. Three options:

- **UTM** — QEMU front-end, free. Apple Silicon native. Good for
  full-VM workflows.
- **Multipass** (Canonical) — Ubuntu-flavored, CLI-first.
  `multipass launch --name dev --cpus 4 --mem 8G --disk 40G`,
  then `multipass shell dev`. Closest analog to the WSL
  experience on a Mac.
- **Parallels** / VMware Fusion — paid, polished, good for
  GUI-heavy work.

The right pattern when you need it: **a Multipass VM running the
same bartr/wsl `scripts/base.sh`** (with the WSL-specific bits
skipped). Now your Mac dev environment matches your droplet dev
environment matches your colleague's WSL dev environment.

#### Example

A minimal `~/.zshrc` opinion (full version in Module 6):

```bash
# zsh + oh-my-zsh assumed installed.
export ZSH="$HOME/.oh-my-zsh"
ZSH_THEME="robbyrussell"

plugins=(git docker docker-compose kubectl gh fzf ripgrep)

source $ZSH/oh-my-zsh.sh

# Curriculum aliases — present on every platform.
alias k=kubectl
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgd='kubectl get deploy'
```

Bringing up a Multipass-based "Mac that matches WSL" environment:

```bash
brew install --cask multipass
multipass launch --name dev --cpus 4 --memory 8G --disk 40G 24.04
multipass shell dev

# inside the VM:
git clone https://github.com/bartr/wsl
cd wsl
sudo ./install.sh   # WSL-specific bits are no-ops on bare Ubuntu

# back on the Mac:
multipass mount ~/projects dev:/home/ubuntu/projects   # optional
code --remote ssh-remote+<vm-name-or-ip> /home/ubuntu/projects/movies-bartr
```

#### Lab

1. On macOS: confirm shell. `echo $SHELL` should print
   `/bin/zsh`. If not, `chsh -s /bin/zsh`.
2. Install OrbStack (`brew install --cask orbstack`) **or**
   Colima (`brew install colima docker docker-compose`). Verify
   `docker run --rm hello-world`.
3. **Demonstrate the resource footprint.** Note RAM use before
   starting the runtime; note it after. Compare to Docker
   Desktop if installed. Note the difference.
4. Install Multipass (`brew install --cask multipass`). Launch
   an Ubuntu 24.04 VM. SSH in. Run a subset of bartr/wsl's
   `scripts/base.sh` (skip the WSL-specific bits). Confirm
   Docker, `kubectl`, `jq`, `gh`, and `zsh` are all present.
5. **VS Code Remote-SSH to the Multipass VM.** Confirm: editor
   on Mac, language servers / terminal / files all on Linux.
   *Same UX as WSL on Windows.*
6. Re-run the filesystem-perf lab from Module 1: build inside
   the VM vs on a Mac-mounted bind. Confirm the asymmetry.

#### Knowledge check

1. macOS zsh default — what year did it ship and why does it
   matter for team-shared tooling?
2. The container-runtime choice on Mac — name three options and
   the trade-off for each.
3. When is "Mac + Multipass" the right pattern over "Mac + Docker
   Desktop" alone?
4. Apple Silicon vs production amd64 — what's the workflow
   change?
5. Why does the curriculum recommend keeping the shell choice
   uniform across the team even when individual operators have
   strong preferences?

---

### Module 4 — Cloud dev environments: Codespaces, droplets, dev tunnels

> **An Ubuntu VM in the cloud with a VS Code interface.** The
> sidearm to the local environment. Spin up in 30 seconds; pay
> only for the time you use it; never carry "the laptop got wet"
> as a risk.

#### Concept

Three credible patterns for "VS Code attached to a Linux machine
that isn't yours":

**1. GitHub Codespaces.** A managed Ubuntu VM (4-vCPU / 8GB
default; bigger sizes available) bound to a GitHub repo. VS Code
in the browser or attached locally via the Codespaces extension.
Configured by a `.devcontainer/` directory in the repo (Module
5). Bills per minute of running time; spin down when idle.

- **Strength**: zero local install; same environment for every
  contributor; works on a Chromebook.
- **Weakness**: requires network for everything; storage is
  ephemeral (the repo persists; uncommitted work is fragile);
  GPU options are limited.
- **Use it for**: code reviews, urgent fixes on a borrowed
  laptop, onboarding contributors, multi-machine workflows
  (laptop + tablet on the same project), demos.

**2. DigitalOcean droplets (or any cloud VM) + VS Code
Remote-SSH.** A persistent Ubuntu VM you own. Bring up with one
`doctl compute droplet create ...`; SSH in; install the
bartr/wsl baseline (it runs unmodified on a DO droplet); attach
VS Code via Remote-SSH.

- **Strength**: full control, persistent state, can run anything,
  can be much bigger / cheaper than Codespaces for sustained
  workloads, snapshots are free and instant.
- **Weakness**: you administer it. Patches, firewall, backups,
  the whole game.
- **Use it for**: long-running experiments (training runs,
  multi-day builds), heavy workloads, anything that needs a
  fixed IP or its own DNS entry.

**3. Microsoft Dev Box.** Managed Windows host with nested
virtualization, so WSL works inside it. Less "Linux VM in the
cloud" and more "Windows desktop in the cloud that can run WSL."
Use it where the team's policy mandates Windows but the work
wants WSL.

**The shared property**: the operator is editing in VS Code; the
work is happening on Linux; the local machine is just a thin
client. If you can do this fluently, "my laptop is broken" stops
being a workday-killer.

**VS Code dev tunnels** are the bridging primitive: even when
SSH is blocked or the IP isn't reachable, `code tunnel` exposes
the machine through GitHub auth as a browser-accessible VS Code.
Useful for laptops behind hostile NATs, demo machines, etc.

#### Example

Bring up a DigitalOcean droplet running the bartr/wsl baseline:

```bash
# Prereq: doctl auth init + SSH key uploaded to your DO account.
doctl compute droplet create dev-1 \
    --image ubuntu-24-04-x64 \
    --size s-4vcpu-8gb \
    --region nyc1 \
    --ssh-keys "$(doctl compute ssh-key list --format ID --no-header | head -1)" \
    --wait

ip=$(doctl compute droplet get dev-1 --format PublicIPv4 --no-header)

# Install baseline. Same script, different host.
ssh root@$ip "apt update && apt install -y git && \
    git clone https://github.com/bartr/wsl && cd wsl && \
    adduser --disabled-password --gecos '' bart && \
    cp -r . /home/bart/wsl && chown -R bart:bart /home/bart/wsl && \
    sudo -u bart bash -c 'cd /home/bart/wsl && sudo ./install.sh'"

# Attach VS Code.
code --remote ssh-remote+bart@$ip /home/bart/projects/movies-bartr

# Snapshot it when configured.
doctl compute droplet-action snapshot dev-1 --snapshot-name dev-1-baseline --wait
```

The droplet is now a re-imageable dev box. "Reset to baseline"
is `doctl compute droplet create dev-2 --image <snapshot-id>
...` and re-attach VS Code.

A minimal `.devcontainer/devcontainer.json` for Codespaces (full
treatment in Module 5):

```json
{
  "name": "movies-bartr",
  "image": "mcr.microsoft.com/devcontainers/go:1.26-bookworm",
  "features": {
    "ghcr.io/devcontainers/features/docker-in-docker:2": {},
    "ghcr.io/devcontainers/features/kubectl-helm-minikube:1": {}
  },
  "postCreateCommand": "go mod download && make tools"
}
```

Push that; click "Open in Codespace" on GitHub; 30 seconds later
you have a working environment.

#### Lab

1. **Codespaces lab.** In a personal scratch repo, add a minimal
   `.devcontainer/devcontainer.json`. Push. From the GitHub UI,
   open a Codespace. Time how long until you have a usable
   terminal.
2. Edit a file in the Codespace's browser VS Code. Then open
   the same Codespace from a local VS Code via the Codespaces
   extension. Confirm seamless handoff.
3. Stop the Codespace. Note that the *machine* is gone but the
   *state* (anything you'd committed) persists.
4. **Droplet lab.** Bring up a $12/month droplet running the
   bartr/wsl baseline as above. Snapshot it when ready.
5. **Demonstrate snapshot-as-reset.** Break something inside
   the droplet (`sudo systemctl stop docker`, then delete the
   binary). Destroy the droplet. Recreate from snapshot.
   Confirm it works again.
6. **VS Code tunnel lab.** On any Linux machine (even a
   Raspberry Pi at home), run `code tunnel`. Authenticate via
   GitHub. From a browser anywhere, open
   `https://vscode.dev/tunnel/<name>`. Confirm you can edit
   files on the remote machine through the browser.

#### Knowledge check

1. Three credible cloud dev-environment patterns — name them
   and one strength of each.
2. Codespaces vs your-own-droplet — what's the cost / control
   trade-off?
3. When is "VS Code dev tunnel" the right answer over SSH?
4. The bartr/wsl baseline runs unmodified on a DO droplet. Why
   is that a deliberate design property and not a coincidence?
5. The droplet snapshot is the cloud equivalent of `wsl
   --export`. What other reset-to-baseline patterns share this
   shape across the curriculum?
6. An operator's laptop dies mid-week. How long until they're
   productive again, and what determines that number?

---

### Module 5 — Dev containers: IaC for the dev machine

> **The piece that makes "spec the dev environment" tractable.**
> A `.devcontainer/` directory in the repo means *the
> development environment is a deployable artifact*, same as the
> service.

#### Concept

A **dev container** is a Docker container configured as a
development environment, declared in
`.devcontainer/devcontainer.json` (with an optional companion
`Dockerfile` or `docker-compose.yml`). VS Code's Dev Containers
extension reads the file, builds/pulls the image, mounts your
source, and attaches the editor.

The key insight: **the same `devcontainer.json` is the source of
truth for**:

- VS Code Dev Containers (local, your laptop builds the
  container).
- GitHub Codespaces (cloud, GitHub builds the container).
- VS Code Remote-SSH with Dev Containers (cloud machine builds
  the container).
- The bare `devcontainer up` CLI for headless / CI use.

One file, four execution targets. That's the IaC property.

The structure (the common shape):

```
.devcontainer/
  devcontainer.json     # the contract
  Dockerfile            # optional — for custom needs
  post-create.sh        # optional — runs once after build
  docker-compose.yml    # optional — for multi-service dev (Postgres etc.)
```

The four directives that matter most in `devcontainer.json`:

1. **`image` or `build`** — base image, or path to a `Dockerfile`.
   Prefer `mcr.microsoft.com/devcontainers/<lang>:<ver>` images
   when they fit; they're well-maintained and pre-include common
   tooling.
2. **`features`** — composable "install X" recipes. Maintained by
   the community at
   `ghcr.io/devcontainers/features/...`. Add Docker-in-Docker,
   `kubectl`, `gh`, `aws-cli`, etc. without writing Dockerfile
   lines.
3. **`customizations.vscode.extensions`** — VS Code extensions to
   install automatically. Now your team-shared linter / formatter
   / debugger config is in the repo.
4. **`postCreateCommand`** (or `postStartCommand`,
   `postAttachCommand`) — commands to run after the container is
   built. Typically: `go mod download`, `npm ci`, `make tools`.

The pattern that earns its keep: **`.devcontainer/` in every
service repo**, lined up with the production Dockerfile (Module
3 of [study-guide-containers.md](study-guide-containers.md))
where it makes sense. Dev container has the build tools; runtime
container is minimal. Same source of truth for "what Go version,
what kubectl version."

#### Example

A `.devcontainer/devcontainer.json` for a movies-bartr-style Go
service:

```json
{
  "name": "movies-bartr",
  "image": "mcr.microsoft.com/devcontainers/go:1.26-bookworm",
  "features": {
    "ghcr.io/devcontainers/features/docker-in-docker:2": {
      "moby": false,
      "version": "latest"
    },
    "ghcr.io/devcontainers/features/kubectl-helm-minikube:1": {
      "version": "latest",
      "helm": "none",
      "minikube": "none"
    },
    "ghcr.io/devcontainers/features/github-cli:1": {},
    "ghcr.io/devcontainers/features/common-utils:2": {
      "configureZshAsDefaultShell": true,
      "installOhMyZsh": true
    }
  },
  "customizations": {
    "vscode": {
      "extensions": [
        "golang.go",
        "ms-kubernetes-tools.vscode-kubernetes-tools",
        "redhat.vscode-yaml",
        "github.vscode-pull-request-github"
      ],
      "settings": {
        "go.useLanguageServer": true,
        "editor.formatOnSave": true,
        "[go]": { "editor.defaultFormatter": "golang.go" }
      }
    }
  },
  "remoteUser": "vscode",
  "postCreateCommand": "go mod download && make tools",
  "forwardPorts": [8080, 3000, 9090],
  "portsAttributes": {
    "8080": { "label": "movies-api" },
    "3000": { "label": "grafana" },
    "9090": { "label": "prometheus" }
  }
}
```

What this gives a new contributor: clone the repo, open in VS
Code, "Reopen in Container," wait ~90 seconds, and they have **Go
1.26, kubectl, docker, gh, zsh+oh-my-zsh, the right VS Code
extensions, dependencies downloaded, and the right ports
labeled**. Zero per-machine setup.

#### Lab

1. In a scratch repo (or a fork of movies-bartr): add the
   `.devcontainer/devcontainer.json` above. Commit. Push.
2. From VS Code: "Dev Containers: Reopen in Container."
   Time the build.
3. Inside the container: confirm `go version`, `kubectl
   version --client`, `docker version`, `zsh --version`,
   `omz version`. Confirm the extensions installed.
4. **Same file, two targets.** Open the same repo on a
   different machine (or in Codespaces). Confirm identical
   environment.
5. **Demonstrate the reproducibility property.** Bump the Go
   version in `devcontainer.json` from `1.26-bookworm` to
   `1.27-bookworm`. Rebuild the container. Confirm `go
   version` reports the new version. All contributors get the
   same change on their next rebuild.
6. **Compare to docs-only setup.** Time how long it would take
   to set up the same environment from a README's "install Go,
   then kubectl, then docker, then gh, then..." — even
   pessimistically, the dev container wins by 30+ minutes per
   contributor.

#### Knowledge check

1. The four execution targets a single `devcontainer.json`
   supports — name them.
2. `features` in dev containers — what problem do they solve
   that a custom Dockerfile would also solve?
3. `postCreateCommand` vs `postStartCommand` vs
   `postAttachCommand` — when does each fire?
4. The dev container and the production Dockerfile both
   describe environments. What's the right relationship between
   them?
5. A new contributor joins the team. With dev containers in
   the repo, what's their day-0 setup?
6. A subtle library version bump needs to propagate to every
   contributor's environment. Without dev containers? With?

---

### Module 6 — Shell baseline: zsh + oh-my-zsh

> **Tab completion is the headline reason; there are others.**
> macOS made zsh the default in 2019; the curriculum makes it
> the default everywhere.

> **Windows operators: this is not a "nice to have."** See
> Module 1's *non-negotiable for Windows developers* section. The
> entire K8s ecosystem writes its examples in POSIX shell. zsh
> (or bash) is the language of the curriculum; PowerShell is the
> language of Windows administration. Use each where it fits.

#### Concept

`zsh` vs `bash`, with the practical wins called out:

1. **Tab completion that knows your CLIs.** `git ch<TAB>`
   completes branch names; `kubectl get pods -n <TAB>` completes
   namespaces; `docker exec <TAB>` completes container names;
   `gh pr <TAB>` completes subcommands. Bash has completion via
   `bash-completion`, but the zsh ecosystem is broader and the
   experience is more uniform.
2. **Better globbing.** Recursive globs (`**/*.go`), qualifiers
   (`*.go(.)` = regular files only, `*.go(L0)` = empty files,
   `*(N)` = nothing if no match instead of error). Once you've
   used it, bash globbing feels primitive.
3. **History search.** `Ctrl-R` works in both; in zsh with the
   right config it's incremental and substring-aware out of the
   box.
4. **Themes + prompt info.** Show current git branch / dirty
   state / kubectl context / current namespace in the prompt
   without writing your own prompt code.
5. **Macros and parameter expansion** — substring slicing,
   case manipulation, expansion modifiers (`${var:l}`,
   `${var:u}`, `${var/from/to}`) that bash also has but zsh's
   are more capable.

`oh-my-zsh` is the configuration framework: theme + plugin
system + a sane default `~/.zshrc`. The alternative is `prezto`
(similar shape, more performance-tuned); the further alternative
is just `zsh` raw with hand-written config. For curriculum
purposes, oh-my-zsh is the recommendation because:

- It's the default the bartr/wsl baseline ships.
- It's well-known; new contributors recognize it.
- The plugin set covers 90% of what an SE-shaped operator needs.

The plugin set worth standardizing on:

```bash
plugins=(
  git              # branch in prompt, dozens of aliases (gst, gp, gco, ...)
  docker           # tab-completion for docker
  docker-compose   # tab-completion for compose
  kubectl          # tab-completion + 80+ aliases (kgp, kgs, kgd, klf, ...)
  gh               # tab-completion for GitHub CLI
  fzf              # ctrl-r / ctrl-t / cd ** for fuzzy finders (requires fzf installed)
  ripgrep          # rg completion
  command-not-found
)
```

Other tools that pair with the shell baseline and the curriculum
inner loop:

- **`fzf`** — fuzzy finder. `ctrl-r` for history, `ctrl-t` for
  files, `cd **<TAB>` for fuzzy directory jumping.
- **`ripgrep`** (`rg`) — fast recursive grep that respects
  `.gitignore`. Used by every editor's "find in files."
- **`bat`** — `cat` with syntax highlighting + paging.
- **`fd`** — `find` with sane defaults + `.gitignore` respect.
- **`zoxide`** — `cd` that learns. `z movies` jumps to the
  movies dir from anywhere.
- **`starship`** — alternative prompt engine (works under
  zsh/bash/fish). Many teams prefer it over oh-my-zsh themes.

The curriculum aliases that follow the operator across every
machine:

```bash
# kubernetes
alias k=kubectl
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgd='kubectl get deploy'
alias kgn='kubectl get ns'
alias kctx='kubectl config use-context'
alias kns='kubectl config set-context --current --namespace'

# git
alias gst='git status -sb'
alias gp='git pull'
alias gco='git checkout'
alias gl='git log --oneline --graph --decorate'

# docker
alias d=docker
alias dps='docker ps'
alias di='docker images'
```

If you're typing `kubectl get pods` in full, every workday is
taxed. The cost of *not* knowing the aliases is paid every hour.

#### Example

A working `~/.zshrc` baseline matching bartr/wsl:

```bash
# Path to oh-my-zsh.
export ZSH="$HOME/.oh-my-zsh"

# Theme — robbyrussell is the default; pick whatever you like
# but standardize across the team for screen-share readability.
ZSH_THEME="robbyrussell"

# Plugins (above).
plugins=(git docker docker-compose kubectl gh fzf ripgrep command-not-found)

source $ZSH/oh-my-zsh.sh

# Curriculum aliases (above).
alias k=kubectl
# ... etc.

# Tool config.
export EDITOR=vim
export PAGER=less
export LESS="-FRX"

# Go.
export GOPATH="$HOME/go"
export PATH="$PATH:$GOPATH/bin:/usr/local/go/bin"

# fzf integration (if installed via package manager).
[ -f ~/.fzf.zsh ] && source ~/.fzf.zsh

# zoxide (if installed).
command -v zoxide >/dev/null && eval "$(zoxide init zsh)"
```

#### Lab

1. On your primary platform: confirm `echo $SHELL` shows zsh.
   If not, install it (`sudo apt install zsh` /
   `brew install zsh`) and `chsh -s $(which zsh)`.
2. Install oh-my-zsh: `sh -c "$(curl -fsSL
   https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"`.
3. Edit `~/.zshrc` to include the plugin list above. `source
   ~/.zshrc`.
4. **Demonstrate tab completion.** `kubectl get pods -n
   <TAB>` (with a real kubeconfig pointed at any cluster).
   `git checkout <TAB>` in a repo with branches. Note that
   neither requires you to remember exact names.
5. Install `fzf`, `ripgrep`, `bat`, `fd`, `zoxide` (whatever
   your package manager calls them). Add the curriculum
   aliases.
6. **Compare to bash.** Open a fresh `bash` shell. Try the
   same `kubectl get pods -n <TAB>`. Note what works and what
   doesn't.
7. **Falsify the "uniformity tax" claim.** Open a screen
   share with a colleague who runs a wildly different prompt.
   Notice the cognitive load reading their terminal vs reading
   one that looks like yours.

#### Knowledge check

1. Three concrete tab-completion wins zsh gives you over bash.
2. zsh became the macOS default in what year and why?
3. oh-my-zsh vs prezto vs raw-zsh — what's the trade-off for
   curriculum use?
4. The `k=kubectl` alias is the headline; what's the actual
   cost of not having it?
5. `fzf` + `ripgrep` + `zoxide` — name what each replaces and
   what it adds.
6. The curriculum standardizes on zsh + oh-my-zsh even when
   individuals prefer other shells. What's the argument for
   uniformity?

---

### Module 7 — Networking pitfalls: which `localhost` is which

> **The trap that bites everyone.** Once internalized, it's
> trivial; before then, every other lab loses an hour to it.

#### Concept

On a Mac/Windows + Docker-Desktop-style setup, `localhost` means
different things from different vantage points:

1. **From your host shell** — your host OS's loopback.
2. **From inside a container** — the container's *own* loopback.
   Not the host.
3. **From inside a different container in the same network** —
   yet another loopback.

The bridging primitives:

- **`host.docker.internal`** — DNS name that resolves from
  inside a container to the host. Mac / Windows / WSL2 Docker
  Desktop populate it; native Linux does *not* by default (use
  `--add-host=host.docker.internal:host-gateway`).
- **`172.17.0.1`** — the default Docker bridge's gateway on
  Linux. Same job as `host.docker.internal`, lower-level.
- **`--network=host`** — the container shares the host's
  network stack. `localhost` inside = `localhost` outside. **On
  Mac/Windows, this still means the *Linux VM's* localhost, not
  the host OS's** — a notorious source of confusion.
- **`-p 8080:8080`** — publish container port 8080 on host
  port 8080. The bridge between the two worlds.

On WSL2 specifically, there's a *fourth* address space:

- **Windows host** — accessed from WSL as `$(ip route | awk
  '/default/ { print $3 }')` or via the WSL-injected
  `host.docker.internal`.
- **WSL distro** — accessed from Windows as `localhost`
  (Windows auto-forwards), or as `wsl.localhost` (Build 2024+),
  or as the distro's IP.
- **Container inside WSL** — its own loopback.
- **Container's published port** — visible on the WSL distro's
  localhost, which Windows auto-forwards to the Windows
  localhost.

The recommendation that follows: **for any service in your
local cluster, use `type: LoadBalancer` with klipper-lb** (covered
in [study-guide-kustomize.md](study-guide-kustomize.md) Module 7
and [skills-inventory.md](skills-inventory.md) C16). This binds a
real port on the cluster node (which on WSL2 = the WSL distro =
auto-forwarded to Windows localhost), and you stop thinking about
address-space gymnastics.

#### Example

The "I can curl it from inside the container but not from my host"
sequence:

```bash
# Inside a container — works.
docker exec mycontainer curl http://localhost:8080/healthz

# From the host — fails. localhost is the host, not the container.
curl http://localhost:8080/healthz   # connection refused

# Fix: publish the port.
docker run -d -p 8080:8080 myimage   # now host localhost:8080 → container 8080
curl http://localhost:8080/healthz   # works
```

The "container needs to call my host service" sequence:

```bash
# Service running on the host at :5432 (Postgres, say).
# From inside a container:

# Mac/Win/WSL2 Docker Desktop:
curl http://host.docker.internal:5432/

# Linux:
docker run --add-host=host.docker.internal:host-gateway ...
# then same call works.
```

#### Lab

1. `docker run -d --rm --name web -p 8080:80 nginx`. From
   your host: `curl http://localhost:8080`. Works.
2. `docker exec web curl http://localhost:80`. Works (inside
   the container).
3. `docker exec web curl http://host.docker.internal:8080`.
   This is the container reaching the host's published nginx.
   Should work on Mac/Win/WSL2; needs `--add-host` on native
   Linux.
4. **The WSL2-specific test.** On Windows: open
   `http://localhost:8080` in a Windows browser. Works (WSL
   auto-forwards).
5. **The address-space confusion lab.** Run a Go HTTP server
   on your host at `:9090` (`go run ./cmd/...` from a
   movies-style repo). From inside a container, try to call
   `:9090`. Document which addresses work from where.
6. **The LoadBalancer fix.** Deploy any service to a local k3d
   cluster with `type: LoadBalancer`. Note the EXTERNAL-IP is
   `127.0.0.1` (or your node's IP). Curl from host directly.
   No port-forward, no `host.docker.internal`, no confusion.
   *This is why the curriculum recommends LB-with-klipper for
   every dev service.*

#### Knowledge check

1. Three meanings of `localhost` in a Docker-on-Mac setup.
2. `host.docker.internal` — what does it resolve to? On which
   platforms does it work by default?
3. `--network=host` on Mac/Windows — what's the gotcha?
4. WSL2 adds a fourth address space. What is it and how does
   Windows reach it?
5. Why does the curriculum recommend `type: LoadBalancer` for
   local dev services over `kubectl port-forward`?

---

### Module 8 — Snapshot + reproducibility discipline

> **The capstone.** The point of every earlier module compounds
> here. A dev environment you can re-create in 5 minutes is a
> dev environment you can experiment fearlessly in.

#### Concept

Five primitives, each a "reset to known good" lever at a
different scope:

| Primitive | Scope | "Reset" time |
|---|---|---|
| **`wsl --export` / `--import`** | Whole WSL distro | ~30 seconds |
| **Cloud provider snapshot** (DO, EC2 AMI, GCE image) | Whole VM | 1-5 minutes |
| **Dev container rebuild** | Per-repo dev environment | 30-120 seconds |
| **Docker named volume reset** | Per-service runtime state | seconds |
| **`git` itself** | Source code | seconds |

The discipline:

1. **Always have at least one rung above the current level
   working.** If you're editing in a dev container, the
   surrounding WSL distro should be snapshot-able. If you're on
   a droplet, it should be a snapshot-able image. If you're on
   a Mac with no cloud option, your `.devcontainer/` is the
   reset lever.
2. **Snapshot *after* setup, *before* experimentation.** The
   moment the environment is configured and working, capture it.
   Before any "let me just install this one thing" that might
   pollute state.
3. **Treat the snapshot as the artifact.** Like the production
   container image: built once, verified, then re-used. Re-building
   from script is the audit trail; re-using the snapshot is the
   inner loop.
4. **Document the recovery path in the repo.** A `README.md`
   that says "if your dev environment breaks, here's the
   3-command reset" is curriculum-level. The agent will not
   write that for you.

The compounding effect: once every layer has a reset button, the
operator stops fearing experiments. "Try this risky thing" stops
being a 4-hour commitment. The methodology's session model
(sessions-not-stories) depends on this — sessions close cleanly
because environments reset cleanly.

#### Example

A repo `README.md` reset-recipe section:

```markdown
## Dev environment reset

| If broken | Run | Time |
|---|---|---|
| Source code | `git reset --hard origin/main && git clean -fdx` | seconds |
| Dependencies | `docker volume rm <project>_node_modules <project>_go_cache` | seconds |
| Dev container | VS Code: "Dev Containers: Rebuild Container" | ~90s |
| WSL distro | `wsl --unregister <distro> && wsl --import <distro> <dir> <baseline>.tar` | ~30s |
| Cloud droplet | `doctl compute droplet create dev-N --image <snapshot-id> ...` | ~2 min |

Snapshots / baselines live in `~/wsl-snapshots/` (local) and as
DO snapshot `dev-baseline-<date>` (cloud).
```

A short shell function for the WSL case:

```bash
wsl-reset() {
  local distro="${1:-bartr}"
  local snapshot="${2:-$HOME/wsl-snapshots/bartr-baseline.tar}"
  wsl --terminate "$distro" 2>/dev/null
  wsl --unregister "$distro" 2>/dev/null
  wsl --import "$distro" "C:\\WSL\\$distro" "$snapshot"
  echo "reset $distro from $snapshot"
}
```

#### Lab

1. **Snapshot every layer you have.** Whatever your platform:
   capture a snapshot/tarball/image of the dev environment
   right now, while it's working. Note the size and the time
   it took.
2. **Verify recovery works.** On a non-critical secondary
   environment: destroy and recreate from the snapshot. Time
   it. Confirm it works.
3. **Document the reset recipe** in a personal `~/dev-notes`
   directory (or in the relevant project repo if curriculum-
   shaped). Use the table format above.
4. **Set a snapshot cadence.** Calendar entry: weekly check
   that your baseline is still good (no drift between baseline
   and what you'd actually want to recover to).
5. **Practice the cross-layer recovery.** Pick any project repo.
   Trigger a "complete reset": reset source via git, rebuild
   dev container, confirm the inner-loop tests still pass.
   Time it. Beat 5 minutes.
6. **Falsify the claim.** Set the snapshot aside. Manually
   re-create the environment from scratch (install OS, install
   tools, configure). Time it. Compare. Demonstrate to yourself
   the cost of *not* having snapshots.

#### Knowledge check

1. Five "reset to known good" levers — name them and their
   approximate reset times.
2. Why does the curriculum recommend snapshot *after* setup,
   *before* experimentation, rather than the other way around?
3. The methodology's session model depends on fast environment
   resets. Why?
4. A teammate's environment breaks an hour before a demo. With
   the right discipline, how long until they're ready?
5. "Treat the snapshot as the artifact" — what's the analogous
   discipline on the production side (covered in another
   guide)?

---

## Per-release review

Same template as the other guides; see the
[Per-release review template section](study-guide-observability.md#per-release-review-template).

Local-platform-specific addition: **at every release, verify the
baseline reset works.** A baseline you haven't recovered from in
a month is a baseline you don't know works. The operator runs the
recovery on a non-critical environment (a second WSL distro, a
scratch droplet, a dev container rebuild) and confirms.

The smallest cadence: once per spec release, on the operator's
primary platform, document the result in the RETRO.

## What this guide is and is not

- **Is:** the slice of "local platform" that maps onto running
  the curriculum (WSL, macOS choices, Codespaces / droplets, dev
  containers, shell baseline, networking traps, snapshot
  discipline).
- **Is not:** a sysadmin guide. OS hardening, kernel tuning,
  driver troubleshooting, dual-boot setups — all real, all out
  of scope.
- **Is not:** a "best laptop" recommendation. The platform
  choice is what's named here; the hardware choice is the
  operator's.
- **Is not:** a Windows-specific guide or a Mac-specific guide.
  Both are covered because real teams have both.

## Open questions

- **GPU-accessible dev environments** — WSL2 GPU passthrough is
  real and improved at Build 2026; Codespaces GPU options are
  limited and expensive; droplets with attached GPUs are
  available but pricey. Worth its own module if/when the
  curriculum grows ML-shaped workloads (cllm-style).
- **Team-wide enforcement of the shell baseline.** Per-machine
  config is suggestion; per-repo dev container is enforcement.
  When does the curriculum cross that line?
- **Multi-machine state sync** (dotfiles, SSH keys, gh auth).
  Currently per-operator. A standardized
  `chezmoi` / `yadm` / dotfile-repo recommendation might be a
  9th module.
- **WSL Containers (Build 2026, public preview).** Module 2 names
  it; Module 3 still recommends OrbStack / Colima on Mac and the
  same on Linux. **Open question:** does WSL Containers replace
  Docker Desktop *and* Docker Engine on Linux for curriculum work
  once it ships GA, or does it stay Windows-only as a Docker
  Desktop alternative? Worth a revisit after GA.
- **Microsoft Dev Box as a curriculum primary.** Currently
  flagged in Module 1 but not a primary recommendation. Worth
  revisiting if/when more enterprises adopt it — and the Build
  2026 enterprise-management story for WSL Containers makes Dev
  Box + WSL Containers a credible managed-dev-environment pattern
  that didn't exist before.
- **WSL AI App Catalog + Agent Sandboxes.** Intersects directly
  with the curriculum's MCP work. Likely belongs in a future
  "agents in regulated environments" guide rather than here, but
  flagging it as a known direction.

## Status

- Not yet run end-to-end with any operator.
- Anchored in
  [github.com/bartr/wsl](https://github.com/bartr/wsl) (real,
  in-use, MIT-licensed) and the dev-container patterns in the
  bartr-owned MIT repos.
- Unlocks for promotion to `methodology/` once at least one full
  run has used Module 5 (dev containers) and Module 8
  (snapshot recovery) end-to-end and reported honestly on the
  time costs vs the friction it removed.
