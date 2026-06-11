# Changelog

## 0.2.0 — 2026-06-11

Cursor as a junior engineer — adds a read-only investigation mode alongside
editing.

- `cursor-bridge-run --ask`: read-only mode. Runs the worker **without**
  `--force` (no edits), skips the git-repo requirement, lock, and diff, and
  returns the worker's answer. For codebase investigation — "where is X",
  "summarize this module", "what does this repo do".
- Result block and audit log now carry a `mode` field (`ask`/`edit`); `--json`
  gains `mode`.
- `cursor-worker` agent reframed as a junior engineer with two modes
  (investigate / implement); description updated for proactive use of both.
- Docs: README two-mode coverage, `examples/delegation-directive.md` and the
  `~/.claude/CLAUDE.md` guidance updated to the junior-engineer framing.

## 0.1.0 — 2026-06-11

Initial release.

- `cursor-worker` subagent: proactive delegation of mechanical coding tasks to
  Cursor models, with brief contract, diff review, and single-retry via
  `--resume`.
- `cursor-bridge-run`: hardened wrapper around headless `cursor-agent` — git-repo
  guard, per-repo lock with stale recovery, timeout watchdog (stock macOS bash
  3.2 compatible), fenced result block (text/JSON), audit log.
- `cursor-bridge-doctor`: environment checks with remediation hints.
- Ships as a self-contained single-plugin marketplace (`source: "./"`).
