# claude-cursor-bridge

**Claude Code orchestrates. Cursor executes.**

A Claude Code plugin that lets Claude (e.g. Opus) proactively delegate well-scoped,
mechanical coding tasks to **Cursor's models** (Composer & friends) through the
headless [`cursor-agent` CLI](https://cursor.com/docs/cli) — the only legitimate way
to use Cursor's models outside Cursor itself. No API keys, no router proxies: your
Claude subscription powers the orchestrator, your Cursor subscription powers the
workers, and **the git diff is the contract** between them.

```
Claude Code session
  Opus (orchestrator)
    │  task matches cursor-worker description (proactive delegation)
    ▼
  cursor-worker subagent (Sonnet)            ← this plugin
    │  writes self-contained brief · picks model · reviews diff · 1 retry
    ▼
  bin/cursor-bridge-run (bash wrapper)       ← this plugin
    │  git-repo guard · per-repo lock · timeout watchdog · result block · log
    ▼
  cursor-agent -p --force --trust --model <m>
    ▼
  shared state: your working tree + git diff
```

## Why

Cursor's models (Composer etc.) have no public API — they are only reachable
through Cursor's own products. But `cursor-agent` runs headless and is built for
scripting. So instead of routing Claude Code's *requests* to other providers
(gateway-style), this plugin treats Cursor as an **external worker**: Claude writes
a self-contained brief, Cursor edits the files autonomously, Claude reviews the
diff like a code reviewer and retries once with a fix-up brief if needed.

Good delegation targets: boilerplate/CRUD, scoped rename/extract refactors, test
scaffolding, repetitive multi-file edits. Kept away from delegation by design:
architecture decisions, security-sensitive code (auth/crypto/secrets/payments),
anything needing the conversation's context.

## Requirements

- macOS or Linux, `git`
- [Claude Code](https://code.claude.com) with an active plan
- [Cursor](https://cursor.com) subscription + the `cursor-agent` CLI

## Install

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

**Proactive** — just work normally. When a task matches the worker's criteria,
Claude delegates on its own:

> "Add a `subtract` function to calc.py plus a pytest test"
> → Opus hands it to `cursor-worker` → Composer edits the files → the diff is
> reviewed, tests run, and you get a verdict.

**Explicit** — mention it:

> "Have the cursor worker scaffold tests for src/parser.ts"
> or invoke directly: `@agent-claude-cursor-bridge:cursor-worker`

**Manual** — the wrapper is a normal CLI you can use yourself:

```bash
cursor-bridge-run --list-models
cursor-bridge-run --model composer-2 - <<'BRIEF'
GOAL: ...
FILES: ...
ACCEPTANCE CRITERIA: ...
BRIEF
```

## How it works

- **Brief contract** — the worker knows nothing about your conversation; every
  delegation is a self-contained brief (GOAL / CONTEXT / FILES / TASKS /
  ACCEPTANCE CRITERIA / OUT OF SCOPE) piped via stdin.
- **Git guard** — refuses to run outside a git repository (exit 3). An autonomous
  worker without version control is how you lose work.
- **Per-repo lock** — `.git/cursor-bridge.lock` enforces one worker per repo at a
  time (exit 5), with stale-lock auto-recovery. Runs are serial by design in v1.
- **Timeout watchdog** — default 600 s, then SIGTERM → SIGKILL (exit 6). Works on
  stock macOS bash 3.2, no GNU coreutils needed.
- **Result block** — status, exit code, chat id (for `--resume` retries),
  duration, changed-files delta, `git diff --stat`, and the worker's last 80
  output lines, in one fenced block the agent parses.
- **Audit log** — one line per run in `~/.cache/claude-cursor-bridge/runs.log`.
- **Review loop** — the subagent checks the diff against the acceptance criteria
  (not the worker's claims), runs the narrowest relevant test, and retries at
  most once via `--resume`. It never commits — the diff stays in your tree.

## Tuning how often Claude delegates

The agent's `description` is what tells the orchestrator (e.g. Opus) *when* to
delegate — but the orchestrator has a strong bias to just edit files itself, so
the description alone tends to under-trigger. The reliable lever is a standing
instruction in your **`~/.claude/CLAUDE.md`** (loaded every session, highest
priority). A balanced directive — delegate substantial mechanical work, keep
trivial one-liners — looks like:

```markdown
# Delegating coding work to Cursor (claude-cursor-bridge)

When the `cursor-worker` subagent is available and you're in a git repo, prefer
delegating substantial mechanical coding to it instead of writing it yourself:
implementing features/modules from a clear spec, whole test suites, scoped or
repetitive multi-file refactors, boilerplate/CRUD/codegen.

Do it yourself for: trivial one-liners/quick fixes (overhead isn't worth it),
architecture/API/design decisions, security-sensitive code (auth, crypto,
secrets, payments), context-dependent debugging, or non-git directories.

When delegating, give a self-contained brief and review the returned diff.
```

Dial it up (delegate almost everything mechanical) or down (only on explicit
request) by editing that directive. Each delegation costs ~30–60s and real
Cursor quota, so delegating trivial edits is usually net-negative.

## Configuration

Precedence: CLI flag > environment > config file > default.

| Setting | Env var | Config key | Default |
|---|---|---|---|
| Model | `CURSOR_BRIDGE_MODEL` | same | `auto` |
| Timeout (s) | `CURSOR_BRIDGE_TIMEOUT` | same | `600` |
| Log dir | `CURSOR_BRIDGE_LOG_DIR` | same | `~/.cache/claude-cursor-bridge` |

Optional config file `~/.config/claude-cursor-bridge/config` (plain `KEY=VALUE`,
shell-sourced):

```bash
CURSOR_BRIDGE_MODEL=composer-2
CURSOR_BRIDGE_TIMEOUT=900
```

## Wrapper exit codes

| Code | Meaning |
|---|---|
| 0 | Worker finished, diff ready for review |
| 2 | Usage error |
| 3 | Not a git repository |
| 4 | `cursor-agent` not installed |
| 5 | Another worker holds this repo's lock |
| 6 | Timeout (worker killed) |
| 7 | Worker exited non-zero |

## Troubleshooting

| Symptom | Fix |
|---|---|
| `cursor-agent CLI not found` | `curl https://cursor.com/install -fsS \| bash` |
| `not authenticated` | `cursor-agent login` |
| Exit 5 but nothing is running | Stale lock is auto-recovered when the owner pid is dead; if it persists: `rm -rf .git/cursor-bridge.lock` |
| Worker times out on first run in a big repo | Cursor indexes the repo on first contact — raise `--timeout` / `CURSOR_BRIDGE_TIMEOUT` |
| Claude never delegates | Check the plugin is enabled (`/plugin`) and the task actually matches the criteria (mechanical + self-contained); or ask explicitly |
| Claude delegates too much | Tell it "don't delegate X"; durable tuning lives in the agent's `description` |

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
