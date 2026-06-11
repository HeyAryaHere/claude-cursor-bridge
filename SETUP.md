# Setup

End-to-end setup in order. Steps 1–2 are one-time prerequisites; 3–6 install
and tune the plugin. This mirrors a known-good install.

## Prerequisites

- macOS or Linux with `git`
- [Claude Code](https://code.claude.com) on an active plan
- A [Cursor](https://cursor.com) subscription (the worker models run on your
  Cursor usage)

## 1. Install the cursor-agent CLI

```bash
curl https://cursor.com/install -fsS | bash
```

This drops `cursor-agent` into `~/.local/bin` (make sure that's on your `PATH`).

## 2. Authenticate with Cursor

```bash
cursor-agent login      # opens a browser
cursor-agent status     # should print your account
cursor-agent models     # lists the models you can delegate to
```

## 3. Install the plugin

From GitHub:

```
/plugin marketplace add HeyAryaHere/claude-cursor-bridge
/plugin install claude-cursor-bridge@cursor-bridge
```

Or from a local clone:

```
/plugin marketplace add /path/to/claude-cursor-bridge
/plugin install claude-cursor-bridge@cursor-bridge
```

Confirm the health check passes (lists CLI, auth, models, git):

```bash
cursor-bridge-doctor
```

## 4. Allow the wrapper to run without prompts

Add to `~/.claude/settings.local.json` → `permissions.allow`:

```json
"Bash(cursor-bridge-run *)",
"Bash(cursor-bridge-doctor*)",
"Bash(*/bin/cursor-bridge-run *)",
"Bash(*/bin/cursor-bridge-doctor*)"
```

(Or just pick "don't ask again" the first time a delegation prompts.) There is
deliberately **no** `Bash(cursor-agent *)` rule — the wrapper is the only
sanctioned entry point.

## 5. Turn on proactive delegation (recommended)

The agent description alone under-triggers, because the orchestrator prefers to
edit files itself. The reliable lever is a standing instruction in your
**`~/.claude/CLAUDE.md`** (loaded every session). Copy the balanced directive:

```bash
cat examples/delegation-directive.md >> ~/.claude/CLAUDE.md
```

See [examples/delegation-directive.md](examples/delegation-directive.md). Dial
it up (delegate almost everything mechanical) or down (only on explicit request)
by editing those bullets. Each delegation costs ~30–60s + real Cursor quota, so
the balanced default skips trivial one-liners on purpose.

## 6. Restart Claude Code

`~/.claude/CLAUDE.md` and a freshly installed plugin only load on launch:

```
/exit        then:    claude -c     # -c resumes your last conversation
```

Verify:

```
/agents            # should list cursor-worker (claude-cursor-bridge:cursor-worker)
/plugin            # claude-cursor-bridge@cursor-bridge shown as enabled
```

## 7. Verify end to end

```bash
mkdir -p /tmp/bridge-smoke && cd /tmp/bridge-smoke && git init -q
printf 'def add(a, b):\n    return a + b\n' > calc.py && git add -A && git commit -qm seed
```

Then, in a Claude Code session in that directory, try both modes:

> "Add a subtract function to calc.py plus a pytest test."  → *edit*: delegates, reviews the diff
> "What does this repo do?"                                 → *ask*: read-only summary, no changes

Expect Claude to delegate to `cursor-worker`. Confirm runs were logged (the
`mode=` field shows which mode each used):

```bash
tail ~/.cache/claude-cursor-bridge/runs.log
```

## Updating the plugin

```bash
cd /path/to/claude-cursor-bridge && git pull
```

A local-dir marketplace pins the installed commit, so after pulling (or editing
the source) refresh the install:

```
/plugin marketplace remove cursor-bridge
/plugin marketplace add /path/to/claude-cursor-bridge
/plugin install claude-cursor-bridge@cursor-bridge
```

Then restart Claude Code. Confirm the new version with `/plugin` (or
`claude plugin list`).
