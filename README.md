# claude-cursor-bridge

**Claude Code orchestrates. Cursor executes.**

A Claude Code plugin that turns **Cursor's models** (Composer & friends) into your
**junior engineer**: Claude (e.g. Opus) stays the lead and proactively delegates both
*codebase investigation* (read-only legwork) and *mechanical implementation* (edits)
to Cursor through the headless [`cursor-agent` CLI](https://cursor.com/docs/cli) — the
only legitimate way to use Cursor's models outside Cursor itself. No API keys, no
router proxies: your Claude subscription powers the lead, your Cursor subscription
powers the worker, and for edits **the git diff is the contract** between them.

```
Claude Code session
  Opus — the lead engineer
    │  delegates proactively (cursor-worker description + your ~/.claude/CLAUDE.md)
    ▼
  cursor-worker subagent (Sonnet)            ← this plugin
    │  Investigate: ask a question, relay the findings
    │  Implement:   brief · pick model · review diff · 1 retry
    ▼
  bin/cursor-bridge-run (bash wrapper)       ← this plugin
    │  git-repo guard · per-repo lock · timeout watchdog · result block · log
    ▼
  cursor-agent -p --trust --model <m>   [--force only in edit mode]
    ▼
  edit mode → your working tree + git diff   ·   ask mode → the worker's answer
```

## Why

Cursor's models (Composer etc.) have no public API — they are only reachable
through Cursor's own products. But `cursor-agent` runs headless and is built for
scripting. So instead of routing Claude Code's *requests* to other providers
(gateway-style), this plugin treats Cursor as an **external junior engineer**.

It works in two modes:

- **Investigate** (read-only) — offload legwork like "where is X", "summarize this
  module", "what does this repo do", so the lead doesn't spend its own context. The
  worker reads and reports; it changes nothing, and no git repo is required.
- **Implement** (edits) — hand over a self-contained brief; Cursor edits the files
  autonomously inside a git repo and the lead reviews the diff like a code reviewer,
  retrying once with a fix-up brief if needed.

Good delegation targets: codebase investigation/summaries, boilerplate/CRUD, scoped
rename/extract refactors, test scaffolding, repetitive multi-file edits. Kept with
the lead by design: architecture decisions, security-sensitive code
(auth/crypto/secrets/payments), trivial one-liners, anything needing the
conversation's context.

## Requirements

- macOS or Linux, `git`
- [Claude Code](https://code.claude.com) with an active plan
- [Cursor](https://cursor.com) subscription + the `cursor-agent` CLI

## Install

> **New here?** [**SETUP.md**](SETUP.md) is the full ordered walkthrough
> (install → auth → plugin → permissions → proactive delegation → verify).
> The condensed version follows.

### 1. The cursor-agent CLI

```bash
curl https://cursor.com/install -fsS | bash
cursor-agent login        # opens a browser
```

### 2. The plugin

From GitHub:

```
/plugin marketplace add HeyAryaHere/claude-cursor-bridge
/plugin install claude-cursor-bridge@cursor-bridge
```

From a local clone (development):

```bash
claude --plugin-dir /path/to/claude-cursor-bridge   # session-only
# or persistently:
# /plugin marketplace add /path/to/claude-cursor-bridge
# /plugin install claude-cursor-bridge@cursor-bridge
```

### 3. Health check

```bash
cursor-bridge-doctor
```

Checks the CLI, authentication, available models, git, and the log directory —
with a remediation hint for anything missing.

### 4. Permissions (recommended)

So delegations run without a prompt for every call, add to
`~/.claude/settings.local.json` → `permissions.allow`:

```json
"Bash(cursor-bridge-run *)",
"Bash(cursor-bridge-doctor*)",
"Bash(*/bin/cursor-bridge-run *)",
"Bash(*/bin/cursor-bridge-doctor*)"
```

Deliberately **no** `Bash(cursor-agent *)` rule — the wrapper (git-repo guard,
lock, timeout, audit log) is the only sanctioned entry point.

## Usage

**Proactive** — just work normally. When a task fits, Claude delegates on its own:

> "Add a `subtract` function to calc.py plus a pytest test"
> → Opus hands it to `cursor-worker` → Composer edits the files → the diff is
> reviewed, tests run, and you get a verdict.

> "Where is rate-limiting handled in this service?"
> → Opus asks `cursor-worker` in read-only mode → gets a summary + file paths
> back, without spending its own context crawling the repo.

**Explicit** — mention it:

> "Have the cursor worker scaffold tests for src/parser.ts"
> "Ask cursor where the auth middleware lives"
> or invoke directly: `@agent-claude-cursor-bridge:cursor-worker`

**Manual** — the wrapper is a normal CLI you can use yourself:

```bash
cursor-bridge-run --list-models

# investigate (read-only — no git repo needed, makes no changes)
cursor-bridge-run --ask --model gemini-3.5-flash \
  "What does this repo do? Summarize the entry points and key modules."

# implement (edits the working tree inside a git repo)
cursor-bridge-run --model composer-2.5 - <<'BRIEF'
GOAL: ...
FILES: ...
ACCEPTANCE CRITERIA: ...
BRIEF
```

## How it works

- **Two modes** — `--ask` (read-only: investigate / answer, no edits, no diff, no
  git repo required) and the default edit mode (worker edits the tree with
  `--force`; the diff is the contract).
- **Brief contract** — the worker knows nothing about your conversation; every
  delegation is a self-contained brief or question piped via stdin.
- **Git guard** — edit mode refuses to run outside a git repository (exit 3); an
  autonomous editing worker without version control is how you lose work. Ask mode
  has nothing to guard, so it runs anywhere.
- **Per-repo lock** — edit mode takes `.git/cursor-bridge.lock` so only one editing
  worker runs per repo at a time (exit 5), with stale-lock auto-recovery. Ask mode
  takes no lock. Runs are serial by design in v1.
- **Timeout watchdog** — default 600 s, then SIGTERM → SIGKILL (exit 6). Works on
  stock macOS bash 3.2, no GNU coreutils needed.
- **Result block** — `mode`, status, exit code, chat id (for `--resume`), duration,
  and the worker's output; edit mode also includes the changed-files delta and
  `git diff --stat`. One fenced block the agent parses (text or `--json`).
- **Audit log** — one line per run (including `mode=`) in
  `~/.cache/claude-cursor-bridge/runs.log`.
- **Review loop** — for edits, the subagent checks the diff against the acceptance
  criteria (not the worker's claims), runs the narrowest relevant test, and retries
  at most once via `--resume`. It never commits — the diff stays in your tree.

## Tuning how often Claude delegates

The agent's `description` is what tells the orchestrator (e.g. Opus) *when* to
delegate — but the orchestrator has a strong bias to just edit files itself, so
the description alone tends to under-trigger. The reliable lever is a standing
instruction in your **`~/.claude/CLAUDE.md`** (loaded every session, highest
priority). Copy the ready-made balanced directive — it frames Cursor as your
junior engineer and covers both investigation and implementation:

```bash
cat examples/delegation-directive.md >> ~/.claude/CLAUDE.md
```

See [examples/delegation-directive.md](examples/delegation-directive.md). Dial it
up (delegate almost everything) or down (only on explicit request) by editing those
bullets. Each delegation costs ~30–60s and real Cursor quota, so delegating trivial
lookups or one-line edits is usually net-negative.

## Configuration & model routing

Per-setting precedence: CLI flag > environment variable > config file > default.

| Setting | Env var / config key | Default |
|---|---|---|
| Model for **ask** (read-only) | `CURSOR_BRIDGE_ASK_MODEL` | `gemini-3.5-flash` |
| Model for **edit** | `CURSOR_BRIDGE_EDIT_MODEL` | `composer-2.5` |
| Force one model for **all** calls | `CURSOR_BRIDGE_MODEL` | (unset) |
| Timeout (s) | `CURSOR_BRIDGE_TIMEOUT` | `600` |
| Log dir | `CURSOR_BRIDGE_LOG_DIR` | `~/.cache/claude-cursor-bridge` |

**Model resolution:** `--model` flag → `CURSOR_BRIDGE_MODEL` → the per-mode
setting (`_ASK_MODEL` / `_EDIT_MODEL`) → per-mode default. So out of the box,
read-only work goes to **Gemini Flash** and edits go to **Composer**, while the
agent routes harder tasks to other models (e.g. **`gpt-5.5-high`**) explicitly —
work spreads across models instead of piling onto one. Set `CURSOR_BRIDGE_MODEL`
only if you want to pin everything to a single model.

Optional config file `~/.config/claude-cursor-bridge/config` (plain `KEY=VALUE`,
shell-sourced; keys match the env var names):

```bash
CURSOR_BRIDGE_ASK_MODEL=gemini-3.5-flash
CURSOR_BRIDGE_EDIT_MODEL=composer-2.5
CURSOR_BRIDGE_TIMEOUT=900
```

## Wrapper exit codes

| Code | Meaning |
|---|---|
| 0 | Worker finished (edit: diff ready to review · ask: answer returned) |
| 2 | Usage error |
| 3 | Not a git repository (edit mode only — `--ask` runs anywhere) |
| 4 | `cursor-agent` not installed |
| 5 | Another worker holds this repo's lock (edit mode) |
| 6 | Timeout (worker killed) |
| 7 | Worker exited non-zero |

## Troubleshooting

| Symptom | Fix |
|---|---|
| `cursor-agent CLI not found` | `curl https://cursor.com/install -fsS \| bash` |
| `not authenticated` | `cursor-agent login` |
| Exit 5 but nothing is running | Stale lock is auto-recovered when the owner pid is dead; if it persists: `rm -rf .git/cursor-bridge.lock` |
| Worker times out on first run in a big repo | Cursor indexes the repo on first contact — raise `--timeout` / `CURSOR_BRIDGE_TIMEOUT` |
| Claude never delegates | Check the plugin is enabled (`/plugin`); add the standing directive to `~/.claude/CLAUDE.md` (see Tuning) — it's the main lever — or ask explicitly |
| Claude delegates too much | Soften the `~/.claude/CLAUDE.md` directive (see Tuning), or tell it "don't delegate X" in the moment |

## Limitations (v1, by design)

- **Serial, not parallel** — one worker per repo (lockfile). Parallel workers via
  git worktrees are on the roadmap.
- **No conversation sharing** — Cursor never sees your Claude context; only the
  brief. That is what makes it safe *and* what makes briefs matter.
- **Two subscriptions** — worker runs consume your Cursor plan's usage.
- Roadmap: explicit `/delegate` command · worktree-parallel workers · optional
  cost/usage reporting.

## License

[MIT](LICENSE)
