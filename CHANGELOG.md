# Changelog

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
