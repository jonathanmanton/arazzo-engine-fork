# Gas City in your sandbox — handoff

**Audience:** the next Claude Code agent who picks this up. You are starting cold; this
file is the only context you get. Read it end-to-end before touching anything.

**Author:** a previous Claude Code agent, 2026-05-24.
**User:** the human you're talking to — they'll introduce themselves.

**Important — this doc is self-contained.** It was originally drafted in a different
sandbox/repo entirely; that origin is irrelevant and intentionally not referenced
anywhere. By the time you are reading this, the file lives in the user's actual
**deliverable repo** (a brand-new repo the user created specifically for this work),
and that is the only repo you need to think about. There is no upstream relationship,
no "parent" repo, no other staging area. Treat the deliverable repo as the universe.

---

## 0. Progress log (most recent first)

Single source of truth for what's been decided and what's still open. **Update this
every time you make progress** so a future you (after compaction) or your successor
doesn't have to reverse-engineer state from git history.

All entries below are decisions made by the user during a prior agent session. They
are settled unless the user reopens them.

- **Deliverable structure: a docker container, ubuntu base.** User's words: *"run
  everything in a docker container inside the sandbox ... I just take the Dockerfile
  that is produced as part of this effort, change some parameters to point to real
  github repos instead of local ones, and off we go."* The Dockerfile + pack + README
  IS the deliverable. Docker daemon was confirmed working in the prior sandbox (root,
  Ubuntu 24.04, `docker info` healthy); re-verify in yours.
- **Sandbox vs laptop topology** (clarified by user):
  - **On the sandbox:** the agent creates local git repos for the pack, for dolt
    data, and for each demo rig. The agent commits frequently and pushes to
    GitHub remotes so artifacts survive container reclaim.
  - **On the laptop:** all of those are GitHub repos. The user clones them, builds
    the docker image, runs the container. If the container dies, the user spins up
    a fresh one and re-clones — the GitHub repos are the source of truth.
- **Beads provider: dolt (`bd`), not `file` — but with a clarification you must
  resolve.** User wants dolt so the "container dies → re-clone → resume" model works
  via dolt's git-like ops. **Important reality check:** dolt is NOT git and dolt's
  remote protocol is NOT compatible with GitHub. Dolt remotes live on DoltHub,
  DoltLab, or a self-hosted dolt server. So either: (a) use Beads `file` provider
  (plain JSON files in `.beads/` — trivially git-trackable, pushable to GitHub) and
  drop the "dolt" framing, or (b) keep Beads `bd`/dolt and add a dolt remote (not
  GitHub) for the re-clone story. **See §1b for the full explanation; flag to the
  user before building.**
- **Agent's role.** User will NOT operate the city. They will not open a shell,
  attach tmux, or run `bd create`. The agent does all of it. The user is the product
  manager; their only interface is chat with the agent. See §1.
- **Docker auth wiring (open question — needs user input).** Inside the container,
  `claude` needs credentials. In the prior sandbox, the parent agent's claude proc
  was auth'd via an `ANTHROPIC_BASE_URL` proxy set on the host (no API key visible).
  On the user's laptop, no such proxy exists — claude needs `ANTHROPIC_API_KEY`.
  Per the user's question ("how are you switching between proxy URL and API key —
  are those interchangeable?"): they are **complementary, not interchangeable.**
  `BASE_URL` says *where* to send requests; `API_KEY` is the *credential*.
  Recommendation surfaced to user: standardize the Dockerfile on `ANTHROPIC_API_KEY`
  as the primary input, accept `ANTHROPIC_BASE_URL` as an optional override for
  proxied/sandboxed environments. **Confirm with user before building.**
- **Compose vs plain docker (open question — recommended compose).** Volume mounts
  get verbose with `docker run`. docker-compose is cleaner for README. User said
  *"Compose is fine if that makes things easier to see."* — interpret as approved.
- **Pack location at runtime (open question — recommended baked-in, mounted as dev
  option).** Baked into the image for the demo path, optionally mounted from the host
  for live-edit development. Awaiting user confirmation.
- **Operational diagram corrected.** The main agent (you), the gascity controller,
  and all role agents are co-located inside one Linux box. Multiple concurrent role
  agents is the *point* of gascity — cost discipline caps pool sizes per role and
  in-flight order queue depth, not number of distinct roles. See §1 and §4.5.
- **Spawned-claude-via-tmux auth verified** in the prior sandbox (`tmux new-session
  -d -s test 'claude -p "say pong"'` returned `pong`). The `ANTHROPIC_BASE_URL` proxy
  authenticated subprocesses transparently. Re-verify in your sandbox.

**Open work for you:**
- Resolve the dolt-vs-github confusion (see §1b) with the user before building.
- Resolve the auth-wiring decision (`ANTHROPIC_API_KEY` as primary) with the user.
- Confirm pack-runtime-location preference.
- Read the lago-morph Gas City deep dive end-to-end (local copy at
  `./reference-only/lago-morph-13-gas-city-deep-dive.md` — see §10).
- Outbound network confirmation from the sandbox (`curl` go.dev and github.com).
- Verify docker daemon works in your sandbox; start it if not running.
- Write the Dockerfile, build the image, bring up gascity inside it.
- Inside the container: `gc` build, city init, controller start, rig add.
- **Design the pack** — the actual goal. See §1 and §1a.

---

## 1. What the user actually wants

Direct quotes from the prior agent session (kept verbatim so you can hear the user's
voice):

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
- The user does not need to know the city is in `/workspace/...` or that `gc start`
  is running in background tmux session `foo` inside container `bar`. Those are your
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

**The "easy to switch to github remotes later" requirement** (user stated): inside
the container, rigs use local `git init` repos and (provisionally) dolt uses its
local managed-city mode. The pack and city.toml should be structured so that
*flipping to GitHub remotes is a config change*, not a code change — e.g. rig URLs
in city.toml expressed as variables that default to local paths but accept
`git@github.com:...` overrides. **Caveat:** dolt's own remotes are not GitHub —
see §1b for the "what does 'github for dolt' actually mean?" discussion you need to
have with the user.

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

## 1b. Persistence: local repos on sandbox, GitHub on laptop, and the dolt question

### The user's mental model (must satisfy this)

The user articulated this clearly:

> "For development on the sandbox the pack will be in a git repo that you create in
> the sandbox. When I run it on my laptop they will all be github repositories. Same
> with dolt. I want this set up so that when I'm running it on my laptop, if the
> container is killed, I can just spin up another container, re-clone the github
> repos, and start again. I thought that was the whole intent behind using dolt with
> a git back-end. That's why I'm asking you to use the git back end for dolt, so that
> it transfers easily to my laptop."

Translated:

- **Sandbox loop:** agent creates local git repos for the pack, rigs, and bead store.
  Agent commits frequently and pushes to GitHub remotes (the deliverable repo + one
  GitHub repo per artifact, or all-in-one — see below).
- **Laptop loop:** user clones those GitHub repos, runs `docker build && docker run`,
  Gas City comes up using cloned state. If the container dies, user `docker rm`s it,
  spins up a fresh one, re-clones if needed, and resumes.
- **GitHub is the source of truth** for everything that must survive a container
  death.

### What works as-the-user-imagines vs. what doesn't

**Works straightforwardly:**

| Artifact | Sandbox storage | Laptop storage | Survives `docker rm`? |
|---|---|---|---|
| Pack (`pack.toml`, prompts, formulas) | Local `git init` in sandbox | GitHub repo | Yes — re-clone from GitHub |
| Rigs (each a git repo) | Local `git init` in sandbox | One GitHub repo per rig | Yes — re-clone from GitHub |
| `city.toml` template | Lives next to the Dockerfile | Same | Yes — re-clone from GitHub |
| Dockerfile, README | Lives in deliverable repo | Same | Yes — re-clone from GitHub |

**Does NOT work the way the user described — needs decision:**

| Artifact | The misconception | The reality |
|---|---|---|
| **Dolt bead store** | "dolt has a git back-end, pushes to github, re-clone like other repos" | Dolt is **not** git. Dolt has its **own** git-LIKE operations (`dolt init`, `dolt commit`, `dolt push`) and its **own** remote protocol. Dolt remotes are DoltHub, DoltLab, or self-hosted dolt servers — **NOT GitHub.** You cannot `git clone` a dolt store; you can `dolt clone` from a dolt remote. |

This is the **single most important thing to resolve with the user before building**.
They asked for "dolt with a git back-end" — that's not a real thing. There are two
genuine paths that satisfy the "container dies → re-clone → resume" goal:

**Path A (recommended): use Beads `file` provider, not `bd`/dolt.**
- Beads has two providers: `bd` (uses dolt under the hood) and `file` (plain JSON
  files in `.beads/`).
- `file` mode produces ordinary files that ARE plain-git-trackable and can live in
  a GitHub repo, satisfying the user's "re-clone from github" model exactly.
- Trade-off: less performant, no SQL queries against bead history, no dolt-style
  branching of bead state. For a learning-and-demonstration pack, these probably
  don't matter.
- Per lago-morph deep dive §6: `file` provider is recommended for "tutorials,
  lightweight" use — which is exactly our case.

**Path B: keep `bd`/dolt, and add a dolt remote (NOT github).**
- Run a dolt remote alongside the city (e.g., on DoltHub free tier, or a
  self-hosted `dolt sql-server --remotesapi-port`).
- Container death → re-clone via `dolt clone <dolt-remote>` instead of
  `git clone <github>`. Two source-of-truth systems instead of one.
- Doesn't match the user's stated "all github" model — more moving parts on the
  laptop side.

**Recommendation to surface to the user:** Path A. Frame it as "we get exactly what
you asked for — everything in github, re-clone on container death — by using the
`file` Beads provider instead of dolt." Reserve dolt for a later "advanced" demo if
they want to show off branching workflows. The user previously rejected `file` Beads,
but that was before they articulated the GitHub-as-source-of-truth model; given that
goal, `file` is the natural fit.

### Rigs (plain git repos)
- Each rig is a regular git repo added to the city via `gc rig add <path>`.
- **On sandbox:** `git init` each rig locally; `git remote add origin
  git@github.com:<user>/<rig-name>.git`; push frequently.
- **On laptop:** user `git clone`s each rig repo into a directory the container will
  mount or copy in.
- **Git worktrees are NOT a gascity SDK feature** (lago-morph deep dive §15). Don't
  introduce them in v1 of the pack; if a later demo wants two agents on different
  branches of the same rig simultaneously, *pack scripts* can call `git worktree add`
  from a `pre_start` hook. Defer until needed.

### Layout sketch (one possible structure for the deliverable repo)

```
<deliverable-repo>/                   ← user-created github repo, ubuntu-base Dockerfile
├── Dockerfile                        ← image build
├── docker-compose.yml                ← brings the container up (recommended; user OK'd)
├── README.md                         ← single source of laptop bring-up instructions
├── pack/                             ← the portable pack (agents, formulas, prompts)
│   ├── pack.toml
│   ├── prompts/
│   └── formulas/
├── city.toml.example                 ← deployment-specific template
├── reference-only/                   ← supporting docs (NOT consumed by gascity)
│   └── lago-morph-13-gas-city-deep-dive.md
├── gascity-sandbox.md                ← this handoff doc (delete after first read if desired)
└── rigs/                             ← rig templates OR pointers to separate rig repos
    └── ...
```

Whether rigs are subdirs of this repo or separate sibling repos is a structural call
worth surfacing to the user. Separate repos match their "all github repositories"
phrasing better; subdirs of one repo is simpler to demo. Recommend separate repos —
it forces you to design the city.toml around rig URLs (the GitHub-flip portability
property in §1).

### Sandbox-side scratch dir vs deliverable repo

While developing on the sandbox, the agent will inevitably create scratch files,
clone gascity upstream, build go binaries, etc. **These don't belong in the
deliverable repo.** Use a separate sandbox scratch path (e.g., `/workspace/scratch/`)
that's gitignored or simply outside the deliverable repo's tree. The only things
that get committed to the deliverable repo are the artifacts in the layout above.

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
Container is reclaimed on idle or session end. **Anything that needs to survive must
be pushed to a GitHub remote** — the deliverable repo for the pack/Dockerfile/etc.,
and (per §1b) separate GitHub repos for each rig if you go with the separate-rig-repos
layout. That includes:
- The Dockerfile, pack, `city.toml` template, README.
- Each rig's source tree (committed in its own GitHub repo).
- Bead store (if you go with Path A `file` Beads — committed in either the
  deliverable repo or its own repo).
- Notes, scratch files, this very document — committed in the deliverable repo.

Things that *don't* need to survive and shouldn't be committed:
- The Go install (re-downloadable; lives inside the container's image layers anyway).
- The `gascity` source clone (re-buildable; image build clones during `docker build`).
- `tmux` sockets, `~/.cache`, runtime `.gc/`, dolt server port files.

**Practical rhythm:** every time a pack/config/Dockerfile change reaches a working
state, commit and push. Treat git history as your durability log. The sandbox can
vanish at any time; nothing in the container or on the sandbox host survives.

### 4.2 GitHub MCP scope
Your GitHub MCP tools may be scoped to one repo at a time by the sandbox's session
configuration. Check the system reminders you got at startup to confirm which repo
you have write access to (it should be the user's deliverable repo). If you find
yourself blocked from a repo you need access to, surface that to the user before
trying workarounds.

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
mkdir -p /workspace/scratch
cd /workspace/scratch
git clone https://github.com/gastownhall/gascity.git
cd gascity
make install
which gc && gc version
```
If `make install` writes outside `$GOBIN`, find where it landed and add that to PATH.
The cloned `gascity` source does not belong in the deliverable repo — keep it in the
sandbox scratch area or do the clone inside the Dockerfile so it lives in image
layers only.

### Step 5 — Initialize the city (5 min)
**Why:** every step from here on assumes a working `gc` binary and a city directory.

```bash
mkdir -p /workspace/scratch/cities
cd /workspace/scratch/cities
GC_BEADS=file gc init bright-lights
cd bright-lights
# inspect what got created — read city.toml end-to-end and summarize it for the user
cat city.toml
```
Things to look for in `city.toml` and report to the user:
- The configured agent provider (must be `claude` for us).
- Concurrency / budget settings.
- The Beads provider (set per §1b decision — `file` if Path A, `bd` if Path B).
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
mkdir -p /workspace/scratch/rigs/hello
cd /workspace/scratch/rigs/hello
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

#### 8c. The pack lives in the deliverable repo
The pack (`pack.toml` + prompts + formulas) belongs at the top level of the
deliverable repo (see the layout sketch in §1b). The `city.toml.example` lives
alongside it. Runtime state (`.gc/`, `.beads/dolt-server.port`, tmux sockets)
stays inside the container and is gitignored if it leaks out.

---

## 6. Decisions — what to decide yourself vs what to ask

Under the updated §1 intent, the user is the product manager, not the operator. Most
implementation choices are now yours to make and report on, not yours to ask. **Default
to deciding and informing; only escalate when the choice is product-shaped.**

### Decide yourself (just report what you did)
- **Where gascity actually runs.** Inside a docker container (ubuntu base) — see §0
  and §1b. Inside the container, the install path is your choice (`/opt/gascity/` or
  `/workspace/` is fine). The deliverable repo (which the user clones on their laptop
  and builds the image from) holds the Dockerfile and pack.
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
mkdir -p /workspace/scratch
cd /workspace/scratch
git clone https://github.com/gastownhall/gascity.git
(cd gascity && make install)
echo "gascity-sandbox/gascity/" >> /workspace/scratch/.gitignore

# Step 5
mkdir -p /workspace/scratch/cities
cd /workspace/scratch/cities
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
