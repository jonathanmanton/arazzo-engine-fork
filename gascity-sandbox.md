# Gas City in this sandbox — handoff

**Audience:** the next Claude Code agent who picks this up. You are starting cold; this
file is the only context you get. Read it end-to-end before touching anything.

**Author of this doc:** previous agent in session, 2026-05-24.
**User:** jonathan@manton.com.
**Working repo:** `jonathanmanton/arazzo-engine-fork` (you are MCP-scoped to this one repo
only — see the system reminder you got at startup).
**Branch to develop on:** `claude/sandbox-independent-sessions-gScht`.

---

## 1. What the user actually wants

Direct quotes from the conversation so you have the tone right:

> "Are you allowed to start independent Claude code sessions in your sandbox? And
> control them via tmux?"

> "The reason I'm asking is because of this project, https://github.com/gastownhall/gascity ,
> that uses tmux to communicate with agents. It uses git for saving context. Tell me if
> you think it is possible to set this thing up on your sandbox. Not for production, but
> so you can directly observe and debug it while we work together configuring a city and
> some rigs."

### Interpreted objectives (in priority order)

1. **Get Gas City installed and running inside this remote Claude Code sandbox.**
   Not on the user's laptop — *here*, in your ephemeral container, because the user wants
   an agent (you) to be able to *directly observe* it while they configure it.

2. **Make the "mayor" session reachable for observation/debug.** The user is not trying
   to admin a fleet. They want a working playground where they can issue `bd create`
   orders and a city/rigs/mayor will respond, and where you can read the state and
   explain what's happening.

3. **Iteratively configure "a city and some rigs" together with the user.** This is
   collaborative, not a one-shot install. After the install lands, expect a back-and-forth
   like "ok now add a rig at X", "now create a formula that does Y", "why did the mayor
   do Z." Optimize the setup so you can answer those questions quickly.

4. **(Implicit) Don't blow money or break things.** "Not for production" is a green light
   to skip resilience, but it is *not* a green light to leave a free-running multi-agent
   loop unattended in a billed cloud sandbox. Default to conservative loop limits and
   make sure the user is in the driver's seat for anything that spawns child `claude`
   processes.

### What the user is **not** asking for

- They are not asking you to package or productionize gascity.
- They are not asking you to contribute upstream to gastownhall/gascity.
- They are not asking for a writeup of gascity's architecture for its own sake.
- They are not asking for a CI/automation setup. The whole point is interactive debug.

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
claude CLI v2.1.150     /opt/node22/bin/claude
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

### 4.5 Cost discipline
Multi-agent loops can fan out fast. Before issuing the first `bd create` that triggers
real agent work:
- Confirm with the user how many concurrent agents they want allowed.
- Check `city.toml` for any concurrency/budget knobs (`max_concurrent`, budget caps,
  etc. — names TBD, audit the schema after `gc init`).
- Default to **one** rig and **one** in-flight order until the user explicitly opts up.

---

## 5. Proposed setup plan — with rationale

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

### Step 8 — Hand control back to the user
**Why:** §1 priority 3 — this is collaborative from here. The user wants to drive
`bd create` themselves (or have you do it on their instruction) and watch the mayor's
reaction.

At this point your job is to summarize:
- Where the city lives, where the rig lives, what `gc start` is doing.
- How to read the mayor's pane (`tmux capture-pane -t mayor -p`).
- What budgets/limits are in place.
- The auth result from Step 2.
- What to commit to the branch before the container idles out.

Then **wait for instructions**. Do not start issuing orders on your own.

---

## 6. Open decisions the user has NOT made yet

Ask about these the moment you're ready to start — don't guess.

1. **Where should the city live?** Recommended: inside the fork at
   `gascity-sandbox/cities/bright-lights/`. Alternative: throwaway location outside
   the repo, accepting that it dies with the container.
2. **Which agent provider for the rigs?** Only `claude` is installed; if Step 2 auth
   check fails, this becomes a forced conversation.
3. **Concurrency cap.** Default to 1. Confirm before raising.
4. **What real work do they want to try first?** "hello world" is the README example;
   the user may have something more concrete in mind given they mentioned "configuring a
   city and some rigs."
5. **Should the gascity source clone be gitignored inside the fork or kept in /tmp?**
   Recommend gitignored-in-fork.

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

- Project: https://github.com/gastownhall/gascity
- README (raw): https://raw.githubusercontent.com/gastownhall/gascity/main/README.md
- README mentions `docs/` in the repo — read those once cloned for installation,
  tutorials, and architecture deep-dives.
- Predecessor name: "Gas Town" — search the repo's issues/discussions for context if
  something behaves unexpectedly.

---

## 11. Final note on tone

The user is technical, decisive, and prefers terse exchanges with a recommendation and
the main tradeoff up front. They asked the previous agent a yes/no question and got a
clean yes/no + caveat table back; they liked that enough to ask for this handoff. Mirror
that style: don't write essays at them, don't ask permission for trivial things, do ask
before anything that spawns child agents or commits something they haven't seen.
