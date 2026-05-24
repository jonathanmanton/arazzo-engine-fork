# Gas City in this sandbox — handoff

**Audience:** the next Claude Code agent who picks this up. You are starting cold; this
file is the only context you get. Read it end-to-end before touching anything.

**Author of this doc:** previous agent in session, 2026-05-24.
**User:** jonathan@manton.com.
**Staging repo (where this doc lives temporarily):** `jonathanmanton/arazzo-engine-fork`,
branch `claude/sandbox-independent-sessions-gScht`. You are MCP-scoped to this repo
only for the current session.
**Deliverable repo:** *to be created by the user* as a brand-new repo with nothing to
do with arazzo-engine. This handoff doc is what bridges the two — the user will read
it, create the new repo, and start a fresh agent session there. Treat
arazzo-engine-fork as a scratchpad for this doc only; do not assume the pack, city,
or any deliverable artifacts will live here long-term.

---

## 0. Progress log (most recent first)

Single source of truth for what's actually been done. **Update this every time you make
progress** so the next agent (or a future you, after compaction) doesn't have to
reverse-engineer state from git history.

- **2026-05-24** — **Deliverable repo is a brand-new user-created repo, NOT
  arazzo-engine-fork.** PR #1 review: *"Don't talk about the repo. I will create a
  brand new repo for all of this. ... The new repo will have nothing to do with
  arazzo-engine."* This handoff doc is the bridge — the user will read it, create
  the new repo, and start a fresh agent session there. Updated masthead, §1
  deliverable, and added a new §1b two-repos section to clarify staging
  (arazzo-engine-fork, scratchpad) vs deliverable (new repo, the real artifact).
- **2026-05-24** — **Docker-first install decided (user committed).** PR #1 review:
  *"run everything in a docker container inside the sandbox ... use an ubuntu base
  ... I just take the Dockerfile that is produced as part of this effort, change
  some parameters to point to real github repos instead of local ones, and off we
  go."* Docker daemon confirmed working in the sandbox (root access, dockerd
  available, `docker info` healthy). The Dockerfile is now part of the deliverable.
  Inside the container: dolt local-only, rigs as local git repos, structured so
  flipping to GitHub remotes is a config change not a code change. **§3 inventory
  and §5 setup plan still need a substantial rewrite for docker-first** — flagged.
- **2026-05-24** — **Beads provider flipped from `file` to `bd` (dolt-backed).** PR #1
  review: user wants dolt on their laptop. Dolt runs purely local via `managed_city`
  mode — no remote required. Updated §1a rule #4 and added a new §1b explaining the
  local-only persistence model: dolt for beads (SQL server + prefix scoping, not
  worktrees), plain `git init` for rigs.
- **2026-05-24** — Fixed an ambiguity in §1 "operating model" diagram (PR #1 review
  comment): the previous diagram made it look like the main agent and gascity were on
  separate hosts. They are not — main agent, gascity controller, and all role agents
  are co-located inside the same sandbox VM, sharing one filesystem and one process
  table. Also clarified in §4.5 that multiple role agents running concurrently is the
  *point* of gascity and is not what cost discipline restricts.
- **2026-05-24** — **Intent clarified by the user (major change).** The user will NOT
  operate the city. They will not open a shell, attach a tmux session, or run
  `bd create`. The agent runs everything. The actual deliverable is a *good city
  configuration* designed for learning and demonstration, which the user will later
  drop onto their laptop and expect to "just work" (modulo OS-level differences like
  Homebrew vs apt). See completely rewritten §1.
- **2026-05-24** — Added lago-morph "Gas City deep dive" to §10 references. It's the
  best architectural map of gascity currently available and you should skim §0–§2 and
  §4 of it before designing the city config.
- **2026-05-24** — Spawned-claude-via-tmux auth verified working (see §4.3). The
  `ANTHROPIC_BASE_URL` proxy authenticates subprocesses transparently — no API key
  needed. This had been flagged as the single most likely blocker; it isn't.
- **2026-05-24** — Sandbox inventory captured (§3). Go 1.25 missing, `gc` not built,
  everything else needed for file-backed Beads is present.
- **2026-05-24** — Handoff doc created and pushed as PR #1 (draft). No CI configured
  for this repo so nothing to babysit there.

**Not yet done** (i.e. Step 1 onward in §5 is all still pending):
- Read the lago-morph deep dive end-to-end (§10) before designing the pack.
- Outbound network confirmation (`curl` go.dev and github.com).
- Go 1.25 install to `$HOME`.
- `gc` build from source.
- City init, controller start, rig add.
- **Design the pack** — the laptop-portable deliverable. This is the actual goal now,
  not "set up a playground." See §1 and §1a.

---

## 1. What the user actually wants

**This section was substantially rewritten 2026-05-24 after the user clarified intent.
The earlier framing ("set up a playground the user observes") was wrong.** Direct
quotes:

> "I do not intend to ever run anything against the city. I will ask the agent to do
> it. I do not want shell access, I do not want to connect to sessions, none of that.
> I want the main agent I am talking to do that."

> "The agent running this city is figuring out a good configuration for the city for
> learning and demonstration use so that later when I set it up on my laptop using that
> configuration it 'just works' (within reason — but I want it to be OS differences and
> things like that which has to be debugged, not the gascity configuration itself)."

Earlier in the conversation, before this clarification:

> "Tell me if you think it is possible to set this thing up on your sandbox. Not for
> production, but so you can directly observe and debug it while we work together
> configuring a city and some rigs."

(That "we work together" framing has been narrowed: the user is the product manager,
the agent is the operator.)

### The operating model

Everything below the user's chat box lives inside **one Linux VM** — the sandbox.
That includes the main agent (you), the gascity controller, and every role agent the
controller spawns. They share a filesystem and a process table. The user is outside
the VM and only ever talks to you via chat.

```
┌────────────┐         ┌─────────────────────────────────────────────────────────────┐
│   USER     │         │                  SANDBOX VM (one Linux box)                 │
│ jonathan@  │◀──chat──┤                                                             │
│ Only       │ (the    │   ┌──────────────────┐                                      │
│ interface  │  only   │   │  MAIN AGENT      │── runs `gc`, `bd`, tmux, git ──┐    │
│ is chat    │  link)  │   │  (you, Claude    │                                │    │
│            │         │   │   Code session,  │                                │    │
│            │         │   │   itself a       │                                │    │
│            │         │   │   `claude` proc) │                                │    │
│            │         │   └─────────┬────────┘                                │    │
│            │         │             │ launches & observes                    │    │
│            │         │             ▼                                        │    │
│            │         │   ┌──────────────────┐                               │    │
│            │         │   │  GAS CITY        │  controller / beads / events  │    │
│            │         │   │  (gc start)      │                               │    │
│            │         │   └─────────┬────────┘                               │    │
│            │         │             │ spawns tmux-runtime panes              │    │
│            │         │             ▼                                        │    │
│            │         │   ┌──────────────────────────────────────────────┐  │    │
│            │         │   │  ROLE AGENTS — multiple concurrent `claude`  │  │    │
│            │         │   │  processes, one per role per pool slot.      │  │    │
│            │         │   │  e.g. mayor (supervisor) + whatever roles    │  │    │
│            │         │   │  your pack defines (deacon, polecat, etc.).  │  │    │
│            │         │   │  All sharing the same FS as you.             │  │    │
│            │         │   └──────────────────────────────────────────────┘  │    │
│            │         │             ▲                                        │    │
│            │         │             └────────────────────────────────────────┘    │
│            │         │             (you can `ps`, `tmux ls`, read .gc/, etc.)    │
└────────────┘         └─────────────────────────────────────────────────────────────┘
```

Key things this diagram is asserting (which the previous version got wrong):

- **Main agent and role agents are co-located on one VM.** You can `ps -ef` and see
  yourself, the gascity controller, and every role agent in the same process table.
  You can read every file they touch. You are not in a different host from them.
- **Multiple role agents run concurrently — that is the entire point of gascity.**
  A pack typically defines several roles (mayor + others). Each role runs as its own
  `claude` subprocess, often in its own tmux pane, in parallel. Don't try to serialize
  them; that defeats the purpose. (See lago-morph §2 and §5 for the runtime model.)
- **You are also a `claude` process.** So when you launch role agents via `gc`, the
  resulting `ps` will show your own claude process *plus* one per role. Pool sizes
  multiply that.
- **The user only sees chat.** They do not see `ps`, they do not see tmux, they do
  not see file paths unless you put one in a chat message. Their only sensor on the
  whole VM is your text replies.

What the user does and doesn't do:
- The user **never** runs `gc`, `bd`, `tmux`, or any sandbox shell command.
- The user does not attach to the mayor or any other session.
- The user does not need to know the city is in `/home/user/arazzo-engine-fork/...` or
  that `gc start` is running in background tmux session `foo`. Those are your
  implementation details to manage and summarize.
- All operational state, all debug info, all "is it working" judgments come from you,
  in chat, in plain language.

### The deliverable (this is the actual goal)

The artifact you are producing is **a Dockerfile + Gas City configuration that the
user can `docker run` on their laptop** and have Gas City just work. Concretely, by
the time this work is "done," the user's brand-new deliverable repo should contain:

- A **Dockerfile** (ubuntu base — user's preference) that installs Go, gascity, dolt,
  bd, tmux, claude CLI, and any other tooling, then copies in the pack and brings up
  the controller. This is the "no OS-specific hurdles" path — sidesteps Homebrew vs
  apt, Mac vs Linux, etc.
- A **pack** (the portable part) sitting alongside the Dockerfile in the new repo —
  see §1a for what this means and §2 for the three-file separation pattern lago-morph
  documents.
- A `city.toml` that demonstrates the configuration meaningfully, with comments that
  explain *why* each section is there.
- One or more demonstration **rigs** (or rig templates) that exercise interesting
  multi-agent behavior.
- A short **README in the new repo root** telling the user the single
  `docker build` / `docker run` command (or `docker-compose up`) to bring the city
  up locally, plus where to set `ANTHROPIC_API_KEY` for the container. The
  Dockerfile + README is the only surface the user touches.
- Whatever the user runs on their laptop after following that README should work
  without them having to debug the *gascity configuration* itself. Docker-level
  issues (host docker setup, volume mounts, API key wiring) are acceptable; config
  bugs are not.

**The "easy to switch to github remotes later" requirement** (per PR #1 review):
inside the container, dolt uses a local repo and rigs use local `git init` repos with
no remotes. The pack and city.toml should be structured so that *flipping to GitHub
remotes is a config change*, not a code change — e.g. rig URLs in city.toml expressed
as variables that default to local paths but accept `git@github.com:...` overrides.

### Interpreted objectives (in priority order)

1. **Design a learning-and-demonstration city configuration.** This is the work. The
   sandbox is a lab to develop and validate that configuration; it is not the
   deliverable. The configuration *is* the deliverable.

2. **Operate the city fully on the user's behalf** to validate that configuration. The
   agent does the installs, runs `gc`, issues orders via `bd create`, reads mayor pane
   output, etc. The user only ever sees natural-language summaries from the agent.

3. **Optimize for "drop it on a laptop and it just works."** Concretely:
   - Use the three-file separation lago-morph recommends: `pack.toml` (portable),
     `city.toml` (deployment), `.gc/` (machine-local, gitignored).
   - Avoid hardcoding container-specific paths in the pack. Use relative paths or
     well-documented placeholders the laptop user can swap.
   - Don't bake in `ANTHROPIC_BASE_URL` proxy assumptions — the laptop will use a
     different auth path (Claude Code OAuth or `ANTHROPIC_API_KEY`).
   - Document every assumption the config makes about its host (tools, versions,
     filesystem layout, network).

4. **Demonstrate progressive activation** (lago-morph §4) — start with the minimum
   useful city, then add features (pools, mail, formulas, orders, health) in a way
   that's pedagogically obvious. A learner reading the pack should see "ah, this is
   the level 0 setup, this is the level 2 add-on."

5. **Don't blow money or break things.** Since the user is not in the loop on every
   command, the agent owns cost discipline. Set explicit concurrency caps, time
   bounds on convergence loops, and a "stop if cost crosses $X" mental budget. Pause
   and ask if you're about to spin up something that might fan out.

### What the user is explicitly **not** asking for

- Not asking for shell access, tmux attach, or any direct sandbox interaction.
- Not asking to "watch the mayor" themselves — the agent watches and reports.
- Not asking to productionize gascity or contribute upstream.
- Not asking for a writeup of gascity's architecture for its own sake (but you must
  understand it well enough to produce a good config — see §10 references).
- Not asking for a CI setup on the deliverable.
- Not asking for the configuration to anticipate hosting models beyond
  "personal laptop." (Multi-user, cloud hosting, etc. are explicitly out of scope.)

---

## 1a. The "portable pack" — what makes a config laptop-portable

This is the core technical problem this work has to solve. Read lago-morph §4 (cited
in §10) for the canonical version; this is the operational summary for our context.

Gas City separates configuration into three layers:

| File / dir | What lives here | Portable? | Commit to repo? |
|---|---|---|---|
| `pack.toml` (plus prompt templates, formula files) | Agents, formulas, identity, prompts. The "what this city does." | **Yes — fully portable.** | **Yes.** This is the deliverable. |
| `city.toml` | Deployment-specific: rigs, capacity, providers, paths. | Partially. Templated/commented for laptop use. | Yes, but with explicit notes on what to edit. |
| `.gc/` | Runtime state, sockets, event log, machine-local bindings. | **No.** Machine-local. | **No.** Gitignore it. |

**Portability rules that the configuration must follow:**

1. **Pack must not assume any absolute path.** No `/home/user/...`, no `/opt/...`.
2. **Pack must not assume `ANTHROPIC_BASE_URL` is set.** The laptop will auth
   differently. If the pack needs a specific auth mode, document it in the pack README,
   don't bake env-specific values in.
3. **Pack must not assume Linux.** Avoid Linux-only shell idioms in any `exec` blocks.
   `bash` is fine (Mac has it); GNU-only flags (`sed -i`, `date --iso-8601`) are not.
4. **Beads provider: use `bd` (dolt-backed), not `file`.** The user wants dolt on the
   laptop, so the sandbox should match. Dolt works purely locally — `managed_city`
   mode runs a per-city dolt SQL server with no remote required. Install `dolt` + `bd`
   in the sandbox (no apt root needed; both ship as static binaries from GitHub
   releases). The dolt data lives under `.beads/` (gitignored runtime state). Reserve
   the `file` provider for an explicitly-labeled "lightweight tutorial" variant if you
   build one. See §1b for the local-only persistence strategy and the worktree
   clarification.
5. **city.toml is allowed to have laptop-specific values**, but every such value must
   be commented with what to change. Example:
   ```toml
   [workspace]
   # Path to the workspace root. Change to wherever you cloned the pack on your laptop.
   root = "./workspace"
   ```
6. **Document tool prerequisites in the pack README** with exact install commands for
   macOS (Homebrew) and Linux (apt). The user's laptop is unknown; cover both.

**Test of portability:** the agent should be able to mentally simulate "user runs
these N commands on a fresh macOS laptop with Homebrew" and predict success. If you
can't, the pack isn't done yet.

---

## 1b. Local-only persistence — dolt, rigs, and the worktree question

The user raised this in PR #1 review:

> "I DO want to use dolt by the time it reaches my laptop. Is it possible for you to
> create a local git repo (no remote) on the sandbox, and have all agents access it
> using worktrees? [...] As a matter of fact, you should probably do this with the
> sample rigs as well. But make sure to frequently save in the main repo their state
> if it is important for configuration. Since gascity relies so much on git repos the
> local ones should work — just be aware that everything is transient on the sandbox,
> so frequent commits to the main repo and pushes to the remote is a good idea."

The user's instinct is correct (local-only stores work, frequent fork-repo commits
preserve config), but the mechanism is slightly different from what the comment
sketches. Here's the actual model.

### Dolt (the Beads backend)
- **Dolt is NOT git.** It's a SQL database with git-*like* operations (init, commit,
  branch, push). A dolt store lives in `.dolt/` (or under `.beads/` for our use).
- **It runs purely local with no remote configured.** Gascity's default `managed_city`
  mode launches a per-city dolt SQL server, port recorded in `.beads/dolt-server.port`,
  no DoltHub or external server needed. (Source: lago-morph deep dive §6, citing
  `internal/beads/contract/files.go`.)
- **Multi-agent access is via SQL connections to that one server**, not worktrees.
  All rigs share the same physical dolt server; isolation is by bead-ID prefix per
  rig (e.g. `riga-h2t`, `rigb-gne`). The rig's `.beads/config.yaml` sets
  `gc.endpoint_origin: inherited_city`.
- **Therefore, no worktrees are involved on the beads/dolt side.** The "local repo,
  no remote, multiple readers/writers" pattern the user wants is achieved by dolt's
  SQL server + prefix scoping. Just install dolt + bd and let the default behavior
  do this for us.
- **Laptop transition is clean:** the user can keep using local-only dolt on their
  laptop, OR add a remote (DoltHub, DoltLab, or self-hosted dolt remote) without
  changing the pack — that's a city.toml / `.beads/config.yaml` concern.

### Rigs (which ARE git repos)
- **Rigs are plain git repos** added to the city via `gc rig add <path>`. They are
  separate registered directories, not gascity-managed worktrees.
- **Local-only (no remote) is fine for sandbox demonstration rigs.** Just
  `git init` them, never `git remote add`. On the laptop the user can add a GitHub
  remote without affecting pack design.
- **Git worktrees are not a gascity SDK feature** — per the lago-morph deep dive §15,
  worktrees are a pattern that *pack scripts* can use from `pre_start` hooks if a
  pack wants to demonstrate parallel agent work on the same codebase. Don't introduce
  worktrees in v1 of the pack; if a later demo calls for "two role agents working
  different branches of the same rig in parallel," reach for `git worktree add` from
  a pack script then.
- **Concurrent multi-agent access to a single rig directory** (without worktrees) is
  fine for sequential or coordinated work; problematic if two agents simultaneously
  do `git checkout` to different branches. Pack design should match this: prefer
  patterns where one agent owns the rig's working tree at a time, or use worktrees
  when truly parallel.

### Two repos, two purposes — staging vs deliverable

This was clarified by the user in PR #1 review ("don't talk about the repo. I will
create a brand new repo for all of this. ... The new repo will have nothing to do
with arazzo-engine"). There are two distinct repos in play:

- **Staging repo (`jonathanmanton/arazzo-engine-fork`, this one):** holds this handoff
  doc and any working notes during this Claude session. Treat it as a scratchpad.
  Do NOT design the pack to live here. When this session ends, the relevant artifact
  from this repo is *just this doc*.
- **Deliverable repo (to be created by the user, name TBD):** brand-new repo with no
  arazzo-engine relationship. Will contain the Dockerfile, the pack, city.toml
  template, rig templates, and a top-level README. The next agent (or a future you,
  in a session pointing at the new repo) will build all of this there.

### What lives where (the durability strategy)

Inside the docker container (which itself runs inside the sandbox), there's a third
location: runtime state. Putting it all together:

| Location | Contents | Lifetime |
|---|---|---|
| **Deliverable repo** (new, user-created) | Dockerfile, pack/, city.toml template, rig templates, README | Permanent — this is the deliverable |
| **Staging repo** (arazzo-engine-fork) | This handoff doc; any in-session scratch notes | Until the deliverable repo exists, then archive/abandon |
| **Docker container FS** (inside sandbox) | Running gascity install, dolt data, rig working trees, `.gc/` runtime state, `.beads/dolt-server.port` | Container lifetime — durable across `docker stop`/`docker start` if you mount a volume; gone on `docker rm` |
| **Sandbox host FS** (outside container) | The cloned deliverable repo (for development), docker daemon state, scratch | Sandbox session lifetime — gone on container reclaim |

**Frequent-commit rule** (per PR #1 review: *"frequent commits to the main repo and
pushes to the remote is a good idea"*): any time a meaningful pack/config change
lands and is validated, commit and push to the deliverable repo. Treat it as a
journal of configuration state worth preserving — runtime state and re-derivable
data stay out (gitignored or volume-mounted only).

**Re-derivable vs not:** dolt bead data is *technically* re-derivable from orders in
the pack + a fresh run, but in practice if you're mid-debug and the container dies,
you lose investigation context. The user's point that "we don't even have to worry
about the docker container failing, as we can restart everything by just bringing up
a new one" depends on the bead store being durable — so mount the dolt data
directory as a docker volume that survives `docker rm`. If snapshots-to-git become
needed later (periodic `dolt dump`), add then; don't pre-build.

---

## 2. What Gas City is (so you don't have to re-research it)

Source: https://github.com/gastownhall/gascity (README fetched 2026-05-24).

Gas City is a Go-based "orchestration-builder SDK for multi-agent coding workflows."
Mental model:

- A **city** is a declarative orchestration space defined by `city.toml`. You create one
  with `gc init <path>`.
- A **rig** is a git-backed project directory registered with `gc rig add .` from inside
  that git repo. Rigs are where work actually executes.
- **Beads** (`bd`) is the work-tracking / formula store. Default backend is Dolt-based
  (requires `dolt` + the `bd` binary), but you can switch to a file backend with
  `GC_BEADS=file` env var or `[beads] provider = "file"` in `city.toml`. **For our
  sandbox, file backend is the right choice** — avoids two extra binary installs.
- The **mayor** is the supervisor/controller session that reconciles desired state
  (orders in Beads) to running state (agents on rigs). You "watch" it with
  `gc session attach mayor`, which is a `tmux attach`-style command.
- Runtime providers include `tmux`, `subprocess`, `exec`, `ACP`, and `Kubernetes`.
  For this sandbox **tmux is the right provider** (it's what's installed and what the
  user already mentioned).
- Agent providers documented: `claude`, `codex`, `gemini`. We have `claude` only.

Quickstart per the README (verbatim):

```bash
gc init ~/bright-lights
cd ~/bright-lights
gc start

mkdir hello-world
cd hello-world
git init
gc rig add .

bd create "Create a script that prints hello world"
gc session attach mayor
```

Prereqs per the README (status in *our* sandbox in the next section):

| Tool | Required | Min ver |
|---|---|---|
| tmux | always | — |
| git | always | — |
| jq | always | — |
| pgrep | always | — |
| lsof | always | — |
| dolt | for `bd` Beads | 1.86.2+ |
| bd | for `bd` Beads | 1.0.0 |
| flock | for `bd` Beads | — |
| gh | optional (GitHub gates) | — |
| claude / codex / gemini | per provider | — |

---

## 3. Sandbox inventory (verified 2026-05-24)

> **STALE NOTE:** This inventory was captured for a direct-install model. Under the
> now-decided docker-first approach (see §0 and §1b), most of these tools only need
> to exist *inside the container* — the sandbox itself only needs docker + git +
> claude. **Sandbox-side requirements:** docker (✅ available, dockerd started this
> session — Ubuntu 24.04, root access), git, claude. **Container-side requirements:**
> everything else (Go 1.25, tmux, jq, pgrep, lsof, flock, dolt, bd, gascity build).
> The next agent should rewrite this section for the docker-first plan.

Run these yourself before trusting any of this — the container is ephemeral, so by the
time you read this it may have been rebuilt.

### Already present
```
tmux 3.4                /usr/bin/tmux
git 2.43.0              /usr/bin/git
jq 1.7.1                /usr/bin/jq
pgrep (procps)          /usr/bin/pgrep
lsof 4.95.0             /usr/bin/lsof
flock (util-linux)      /usr/bin/flock
make                    /usr/bin/make
go 1.24.7               /usr/local/go/bin/go          ← TOO OLD, gascity wants 1.25+
claude CLI v2.1.150     /opt/node22/bin/claude   ← auth verified for subprocesses, see §4.3
node, npm, pnpm, npx    /opt/node22/bin/
```

### Missing
- **Go 1.25+** (we have 1.24.7). Likely fix: download
  `https://go.dev/dl/go1.25.linux-amd64.tar.gz` to `/home/user/go-1.25/`, prepend its
  `bin/` to `PATH` for the build. *Do not* try to overwrite `/usr/local/go` — no root
  needed if you install to `$HOME`.
- **`gc` binary** — must build from source via `make install` (the Homebrew tap in the
  README is macOS-only). Build artifact will land in `$GOBIN` or `$HOME/go/bin`
  depending on your env; verify before running.
- **`dolt`, `bd`** — skip by using `GC_BEADS=file`. Only revisit if file backend turns
  out to be insufficient for what the user wants to demo.
- **`gh`** — skip. We don't need GitHub gates for a sandbox playground, and you're
  MCP-scoped to one repo anyway.

### Disk / network
- ~30 GB free on `/`. Plenty for Go, gascity, and several rigs.
- Outbound network policy is unknown until tested. **Before announcing anything works,
  try `curl -I https://go.dev/dl/go1.25.linux-amd64.tar.gz` and
  `git clone https://github.com/gastownhall/gascity` to confirm** — if either fails,
  the rest of the plan is moot until network is sorted.

---

## 4. Hard environmental constraints — read these or you'll waste time

These are not optional things to "design around later." They are sandbox properties that
shape every decision.

### 4.1 The sandbox is ephemeral
Container is reclaimed on idle or session end. **Anything that needs to survive must be
committed and pushed to `jonathanmanton/arazzo-engine-fork` on branch
`claude/sandbox-independent-sessions-gScht`.** That includes:
- The `city.toml` and any city-local state.
- Any rig directories the user wants persisted.
- Notes, scratch files, this very document.

Things that *don't* need to survive (and shouldn't be committed):
- The Go 1.25 install — re-downloadable.
- The cloned `gascity` repo and its `gc` binary — re-buildable.
- The tmux sockets and `~/.cache` artifacts.

**Strong recommendation:** put the city *inside* the fork repo, e.g.
`/home/user/arazzo-engine-fork/gascity-sandbox/city/`, so `git add` survives container
churn. Don't put it at `~/bright-lights` (the README's example path) because that's
outside the repo and will vanish.

### 4.2 You are MCP-scoped to ONE repo
Your GitHub MCP tools only work against `jonathanmanton/arazzo-engine-fork`. You cannot
create a separate "city" repo. Either:
- Host the city as a subdir of this fork (recommended; simplest), or
- Initialize a local-only git repo for the city and just accept it dies with the
  container (only OK for throwaway demos).

### 4.3 Auth for spawned `claude` subprocesses — RESOLVED, it works
**Originally flagged as the single most likely blocker. Verified working 2026-05-24
and the previous agent updated this section after the test.**

Test that was run:
```bash
tmux new-session -d -s authtest 'claude -p "reply with the single word: pong" > /tmp/authtest.out 2>&1; echo EXIT=$? >> /tmp/authtest.out'
# wait for /tmp/authtest.out to contain EXIT
cat /tmp/authtest.out
# → pong
# → EXIT=0
```

Why it works (best inference): `ANTHROPIC_BASE_URL` is set to a host-managed proxy and
is inherited by subprocesses through normal env-var inheritance, and the proxy
authenticates based on container identity rather than per-process credentials. The
OAuth file descriptor (`CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR`) is *not* needed by
spawned processes — they go through the proxy.

Practical implication: Gas City can spawn `claude` inside rigs and tmux runtimes
without us needing to provision an `ANTHROPIC_API_KEY`. Treat this as resolved unless
you see authentication errors come back from agent runs, in which case re-test with the
snippet above and reopen the question.

Note for situational awareness:
- There is no `ANTHROPIC_API_KEY` in env.
- There is no `~/.claude/` directory.
- `ANTHROPIC_BASE_URL` is set (do not unset it — that's what's making this work).

### 4.4 You can't truly "attach" a tmux session interactively
`gc session attach mayor` is meant for a human at a TTY. Neither you nor the user has a
live TTY into this sandbox. Workaround:
- Use `tmux capture-pane -t mayor -p` to read the pane's current contents.
- Use `tmux send-keys -t mayor '...' Enter` to drive it if needed.
- Tail any log files gascity writes (look under the city dir and `~/.local/state` or
  similar — verify with `find <city-dir> -type f -name '*.log' -mmin -10`).
- When the user asks "what is mayor doing?", capture the pane and relay the contents
  with file references. Do *not* claim to be "watching" it — you're sampling it.

### 4.5 Cost discipline — the agent owns this, not the user
Per the updated §1 intent, the user is not in the loop on individual commands. That
means **you** own preventing runaway agent fan-out. There is no "ask the user before
each `bd create`" safety net.

**Important distinction:** "multiple agents at once" is the *point* of gascity. A pack
typically defines several distinct roles (mayor + others) that run concurrently — do
not interpret cost discipline as "only one agent allowed." What we cap is duplicates
within a role pool and queue depth of in-flight work, not the number of distinct
roles.

Concrete rules:
- **Distinct roles:** as many as the pack design calls for. Adding a role is a design
  decision, not a cost decision.
- **Pool size per role:** default to **1 worker per role** during demo/dev. The
  controller will still spin up one process per role, so a pack with mayor + 3 other
  roles = 4 concurrent `claude` processes plus you. That's expected.
- **In-flight orders:** default to **1 at a time** until you have a specific
  demonstration reason to allow more (e.g. showing a fanout pattern).
- **Convergence loop bounds:** set explicit `max_iterations` and cooldown periods —
  check lago-morph §7 for the actual config keys.
- **Pre-dispatch cost estimate:** before issuing any order that triggers an `exec`
  block calling `claude`, mentally estimate: "if this loops 10x across N roles, what's
  the cost?" If the answer is more than a few dollars, pause and surface a plan to
  the user before dispatching.
- **Disk hygiene:** watch the `.gc/events.jsonl` unbounded-growth gotcha (lago-morph
  "Critical Gotchas") — periodically check disk and rotate if needed.
- **Emergency stop:** if the controller is spawning more processes than expected,
  `gc stop` (or kill the controller process) first, diagnose second. Don't let it
  run while you investigate.

---

## 5. Proposed setup plan — with rationale

> **STALE NOTE:** This plan describes a direct sandbox install. Under the now-decided
> docker-first approach (see §0 and §1b), the plan should be rewritten as:
> (1) verify network + auth on the sandbox host, (2) clone the user's new deliverable
> repo, (3) write a Dockerfile that installs Go 1.25, gascity, dolt, bd, tmux, claude
> (with auth wiring TBD — `ANTHROPIC_API_KEY` on laptop, but inside the sandbox the
> proxy works via `ANTHROPIC_BASE_URL`; need to plumb that into the container build),
> (4) `docker build`, (5) `docker run` with volume mounts for dolt data and the pack,
> (6) `docker exec` to drive `gc init` / `gc start` / `gc rig add` from inside the
> container. The next agent should rewrite this section before executing.
>
> The direct-install steps below are kept as reference for what each gascity
> bring-up step looks like — the *operations* are the same, just done inside a
> container.

Do these in order. Each step has a *why* so you can deviate intelligently if reality
disagrees.

### Step 0 — Sanity check the environment (5 min)
**Why:** the container may have been rebuilt and the inventory in §3 may be stale.

```bash
which tmux git jq pgrep lsof flock make go claude
go version
tmux -V
claude --version
df -h /home/user
```
If anything from §3's "Already present" list is gone, stop and tell the user before
proceeding.

### Step 1 — Confirm outbound network (2 min)
**Why:** if you can't reach go.dev or github.com, the rest of the plan is dead. Fail
fast.

```bash
curl -fsI https://go.dev/dl/go1.25.linux-amd64.tar.gz | head -1
curl -fsI https://github.com/gastownhall/gascity | head -1
```

### Step 2 — Re-confirm spawned-claude auth (1 min — quick sanity, was previously verified)
**Why:** see §4.3. Originally flagged as the single most likely blocker; verified
working on 2026-05-24. Re-run this in case the container/proxy state changed; if it
still passes you can move on quickly.

```bash
tmux kill-session -t authtest 2>/dev/null
tmux new-session -d -s authtest 'claude -p "reply with the single word: pong" > /tmp/authtest.out 2>&1; echo EXIT=$? >> /tmp/authtest.out'
until [ -s /tmp/authtest.out ] && grep -q EXIT /tmp/authtest.out; do sleep 2; done
cat /tmp/authtest.out
tmux kill-session -t authtest 2>/dev/null
```
- Expected: `pong` followed by `EXIT=0`.
- If it now fails: stop, capture the error, surface to the user. The previous "what
  to do if it fails" options are in §4.3's git history.

### Step 3 — Install Go 1.25+ to $HOME (5 min)
**Why:** gascity requires it; we don't have root for `/usr/local/go`.

```bash
cd /home/user
curl -fsSL https://go.dev/dl/go1.25.linux-amd64.tar.gz | tar -xz
mv go go-1.25
export PATH="/home/user/go-1.25/bin:$PATH"
export GOPATH="/home/user/go"
export GOBIN="/home/user/go/bin"
mkdir -p "$GOBIN"
export PATH="$GOBIN:$PATH"
go version   # expect go1.25.x
```
**Important:** these `export`s only survive within a single Bash call (the harness does
not persist shell state). For every subsequent Bash call you make, either re-export
them or prefix the command. Consider writing a tiny `gascity-sandbox/env.sh` to
`source` from each call.

### Step 4 — Clone and build gascity (10 min)
**Why:** Homebrew path is macOS-only. Source build is the only option here.

```bash
mkdir -p /home/user/arazzo-engine-fork/gascity-sandbox
cd /home/user/arazzo-engine-fork/gascity-sandbox
git clone https://github.com/gastownhall/gascity.git
cd gascity
make install
which gc && gc version
```
If `make install` writes outside `$GOBIN`, find where it landed and add that to PATH.
Add `gascity-sandbox/gascity/` to a top-level `.gitignore` entry (or place the clone
outside the fork) — we do not want to commit a clone of someone else's repo into ours.

**Decision to surface to the user before doing this:** should the gascity clone live
inside the fork (and be gitignored) or in `/tmp` (and be re-cloned on every container
rebuild)? Recommend gitignored-inside-fork: re-clones are cheap but a stable path is
nicer to refer to.

### Step 5 — Initialize the city inside the fork (5 min)
**Why:** §4.1 — anything outside the fork dies with the container.

```bash
mkdir -p /home/user/arazzo-engine-fork/gascity-sandbox/cities
cd /home/user/arazzo-engine-fork/gascity-sandbox/cities
GC_BEADS=file gc init bright-lights
cd bright-lights
# inspect what got created — read city.toml end-to-end and summarize it for the user
cat city.toml
```
Things to look for in `city.toml` and report to the user:
- The configured agent provider (must be `claude` for us).
- Concurrency / budget settings.
- The Beads provider (must be `file`).
- Any references to absolute paths — those are container-fragile.

### Step 6 — Start the controller, but do NOT attach yet (5 min)
**Why:** we want to see if `gc start` is a daemon, a foreground process, or a tmux
spawner before we run an order through it.

```bash
GC_BEADS=file gc start &
GC_START_PID=$!
sleep 3
tmux ls 2>&1                # see if it created sessions
ps -ef | grep -E 'gc |claude' | grep -v grep
```
Report findings to the user before issuing any `bd create`.

### Step 7 — Add a throwaway rig (5 min)
**Why:** keep the first rig tiny and disposable so the user can see the mayor pick it
up without committing to anything real.

```bash
mkdir -p /home/user/arazzo-engine-fork/gascity-sandbox/rigs/hello
cd /home/user/arazzo-engine-fork/gascity-sandbox/rigs/hello
git init -q
git commit --allow-empty -m "init rig"
GC_BEADS=file gc rig add .
```

### Step 8 — Validate the smoke test, then start designing the pack
**Why:** §1 intent says the deliverable is a *portable pack*, not a running city. By
this step you have a working sandbox installation; now the real work begins.

#### 8a. Smoke test (agent runs it, reports findings)
Issue a trivial `bd create` ("print hello world") against the throwaway rig from Step 7,
let the mayor pick it up, and capture the result. Confirm end-to-end:
- Order shows up in the bead store.
- Mayor reconciles and dispatches.
- `claude` subprocess runs (auth proxy still works in this context).
- Output is captured somewhere readable (event log, pane capture, or rig commit).

Report to the user in chat:
- One-paragraph "the smoke test worked, here's what happened."
- Specific paths to anything useful (`.gc/events.jsonl`, mayor pane capture, etc.) so
  the user can ask follow-up questions like "show me what the mayor said when X."

#### 8b. Pack design (the actual deliverable)
Once smoke test passes, **stop using the throwaway rig** and start building the real
deliverable: a `pack.toml` + accompanying prompt templates + a documented `city.toml`
that demonstrates Gas City's capabilities for a learner.

Read lago-morph §4 first. Then propose a pack design to the user — probably as a short
chat message listing:
- What level of progressive activation the pack will target (start small; level 0–2 is
  probably enough for a demo).
- What agents/roles are in the pack and what they do.
- What formulas demonstrate interesting multi-agent behavior.
- What the pack's README will tell the laptop user to install and run.

Get user buy-in on that plan before writing 500 lines of TOML. Then build it iteratively,
validating each piece in the sandbox before moving on.

#### 8c. The pack lives in the fork repo
Specifically, recommend `/home/user/arazzo-engine-fork/gascity-sandbox/pack/` for the
pack itself, separate from `cities/bright-lights/` (which is the local
deployment-specific `city.toml`). The pack gets committed and pushed; the city.toml
gets committed *as a reference example* with comments showing what to change for a
laptop install; the `.gc/` runtime state is gitignored.

---

## 6. Decisions — what to decide yourself vs what to ask

Under the updated §1 intent, the user is the product manager, not the operator. Most
implementation choices are now yours to make and report on, not yours to ask. **Default
to deciding and informing; only escalate when the choice is product-shaped.**

### Decide yourself (just report what you did)
- **Where gascity actually runs.** Inside a docker container (ubuntu base) — the user
  committed to this in PR #1 review. Inside the container, the install path is your
  choice (`/opt/gascity-sandbox/` or `/workspace/` is fine). The deliverable repo
  (which the user will clone on their laptop and build the image from) holds the
  Dockerfile and pack.
- **Agent provider.** `claude`, because it's installed and auth works. Don't even bring
  up codex/gemini unless the user does.
- **Beads provider.** `bd` (dolt-backed) — user explicitly wants dolt. Inside the
  container, dolt uses a local repo with no remote. Structure city.toml so the rig
  URLs can be swapped to GitHub remotes by config change later.
- **Concurrency cap.** 1 worker per role pool, 1 in-flight order at a time, until you
  have a specific reason to raise it. (Multiple distinct *roles* is expected and good
  — see §4.5.)
- **Source clone location for `gascity`.** Inside the container image at build time;
  the upstream clone shouldn't end up in the deliverable repo.
- **Smoke-test rig content.** Throwaway "hello world" — done inside the container.
- **What gets committed to the deliverable repo.** Dockerfile, pack/, city.toml
  template, rig templates, README. Not: cloned upstreams, runtime `.gc/` or
  `.beads/dolt/` data, tmux sockets, log files.

### Ask the user (these are product-shaped)
- **What the pack should demonstrate.** Multi-agent coordination on what kind of task?
  Code review? Refactor pipelines? Issue triage? The user mentioned "learning and
  demonstration" — get one or two sentences from them on the audience and use case.
- **Pack identity / naming.** What is this pack called? Who is it "by"? The pack will
  end up in a public repo; the user may have opinions.
- **How ambitious to be on day one.** "Minimum viable demo" (one agent, one formula)
  vs. "show off several capabilities" (multiple roles, formulas, orders, health). Both
  are valid; user picks the bar.
- **When you hit a config choice that shapes the laptop UX.** E.g., "do you want the
  pack README to assume Homebrew or also document apt?" Ask.
- **Before any operation that might cost more than ~$1 in agent calls.** Surface the
  plan first.

---

## 7. Things the previous agent already told the user

So you don't contradict yourself:

- Confirmed `claude` and `tmux` are both available, demoed `tmux new-session` works.
- Listed the missing prereqs: Go 1.25+, `gc`, `dolt`+`bd`, `gh`.
- Recommended: Go 1.25 via tarball to `$HOME`, build `gc` from source, file-based Beads,
  skip `gh`.
- Flagged the three caveats: ephemerality, no interactive attach, child-claude billing.
- Recommended starting with "minimum viable setup ... one rig pointing at a throwaway
  git dir ... before adding complexity."
- Ended by asking the user whether to proceed or pick the host-repo strategy first.
  **The user's response to that question was to ask for this handoff doc**, so the
  host-repo question is still open. Treat it as Open Decision #1 above.

---

## 8. Failure modes to anticipate

In rough order of likelihood:

1. ~~**Spawned `claude` has no credentials** (§4.3). Most likely blocker.~~
   Resolved 2026-05-24 — works via `ANTHROPIC_BASE_URL` proxy. Re-test if errors
   reappear.
2. **`make install` writes `gc` somewhere not on PATH.** Find it with
   `find / -name gc -type f -executable 2>/dev/null` and adjust PATH.
3. **`city.toml` defaults to a non-tmux runtime provider** or to `bd` Beads despite
   `GC_BEADS=file`. Read the file and fix explicitly.
4. **`gc start` forks into the background and you can't tell what it's doing.** Look
   for log files under the city dir or `$XDG_STATE_HOME`. Add `--verbose` or `--debug`
   flags if `gc start --help` exposes them.
5. **Tmux pane capture comes back empty** because the agent process writes via a
   non-tty. Try `tmux capture-pane -t <session> -pS -2000` to grab scrollback, or
   redirect agent stdout to a file the city manages.
6. **Network proxy blocks `api.anthropic.com` from inside a subprocess** even though
   the parent claude works. If Step 2 fails with a network error rather than an auth
   error, this is why; surface to the user.
7. **Disk fills up from agent transcripts.** Watch `df -h /home/user` periodically once
   orders start flowing.

---

## 9. One-shot kickoff command block

If you want to just blast through the install once you've talked to the user, here's
the full thing (after Step 2's auth check passes):

```bash
# Step 3
cd /home/user
curl -fsSL https://go.dev/dl/go1.25.linux-amd64.tar.gz | tar -xz && mv go go-1.25

# All subsequent calls must include this:
export PATH="/home/user/go-1.25/bin:/home/user/go/bin:$PATH"
export GOPATH="/home/user/go"
export GOBIN="/home/user/go/bin"
mkdir -p "$GOBIN"

# Step 4
mkdir -p /home/user/arazzo-engine-fork/gascity-sandbox
cd /home/user/arazzo-engine-fork/gascity-sandbox
git clone https://github.com/gastownhall/gascity.git
(cd gascity && make install)
echo "gascity-sandbox/gascity/" >> /home/user/arazzo-engine-fork/.gitignore

# Step 5
mkdir -p /home/user/arazzo-engine-fork/gascity-sandbox/cities
cd /home/user/arazzo-engine-fork/gascity-sandbox/cities
GC_BEADS=file gc init bright-lights
```

Stop there and inspect `bright-lights/city.toml` before continuing.

---

## 10. Useful references

### Primary — read these before designing the pack

- **lago-morph "Gas City Deep Dive"** —
  https://github.com/lago-morph/software-factory/blob/main/research/followup/13-gas-city-deep-dive.md
  Independent architectural deep-dive (May 2026), with file-path citations into the
  gascity repo. This is the best single document for understanding *how to design a
  good pack*. Required reading. Sections most relevant to our work:
  - **§0 & §1** — positioning. "ZERO hardcoded roles" is the key insight: all role
    behavior comes from user config and Markdown prompts, never from Go code. The
    pack is everything.
  - **§2 (Nine Concepts)** — canonical mental model. Layer 0–1 primitives (Session,
    Bead Store, Event Bus, Config, Prompt Templates) and Layer 2–4 derived mechanisms
    (Messaging, Formulas/Molecules, Dispatch/Sling, Health Patrol).
  - **§4 (Configuration)** — `city.toml` structure, **the three-file separation
    (`pack.toml` / `city.toml` / `.gc/`)**, progressive activation levels 0–8, and
    the six-level override cascade. **This is the playbook for portability.**
  - **§5 & §6** — runtime providers (tmux is the right one for us) and beads
    topology. Beads is the universal persistence substrate.
  - **§7 (Workflow Primitives)** — formulas, molecules, wisps, orders, sling,
    convergence loops. The most complex section; the pack will live here.
  - **§8 (Controller Loop)** — the reconciler. Critical for debugging anything
    going wrong with mayor reconciliation.
  - **§19 (Quick Reference)** — cheat sheet once you've internalized §2.
  - **Critical Gotchas** to internalize: in-memory crash tracker (controller restart
    clears quarantine), no cascading restarts, sling shell-exec'd serial, no retry on
    order dispatch failure, `.gc/events.jsonl` unbounded growth, controller socket
    has no auth.
  - **Design principles** to absorb: ZFC (Zero Framework Cognition), GUPP ("Get Up
    and Pick Plums"), NDI (Nondeterministic Idempotence), Bitter Lesson alignment.

- **gascity README (raw)** —
  https://raw.githubusercontent.com/gastownhall/gascity/main/README.md
  Quickstart, prereq table, install paths.

- **gascity project root** — https://github.com/gastownhall/gascity
  After cloning, read `docs/` for installation, tutorials, architecture reference, and
  `AGENTS.md` (referenced by lago-morph as a canonical source).

### Secondary

- Predecessor name "Gas Town" — search the gascity repo's issues/discussions for
  context if something behaves unexpectedly.
- lago-morph deep-dive cites two companion docs (Gas Town deep-dive, Gas Systems
  Substrate) that place Gas City in the wider ecosystem; reach for them if the user
  asks ecosystem-level questions.

---

## 11. Final note on tone and working style

The user is technical, decisive, and prefers terse exchanges with a recommendation and
the main tradeoff up front. They asked the previous agent a yes/no question and got a
clean yes/no + caveat table back; they liked that enough to ask for this handoff.

Working-style implications under the updated §1 intent:
- **Decide and inform; don't ask and wait.** They have explicitly delegated operation
  of the city. Asking "should I install Go now?" wastes their time. Asking "should the
  pack include a code-review agent or a refactor agent?" is on-topic.
- **Surface state proactively, briefly.** Short status updates ("smoke test passed,
  mayor reconciled in 8s, transcript in `.gc/events.jsonl`") beat long ones. They will
  ask follow-ups if they want more.
- **Commit and push frequently.** The container is ephemeral; the user has explicitly
  asked for "commit and push everything asap" once already in this thread. When you
  reach a stable point, push without being asked.
- **When you do ask, ask product questions, not implementation questions.** "What
  should the demo show?" not "what should I name the agent role?"
- **Don't narrate internal deliberation** in chat. They get tool calls summarized; a
  text reply should be a result or a question, not a thinking log.
