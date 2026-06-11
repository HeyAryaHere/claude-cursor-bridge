# Working with Cursor as your junior engineer (claude-cursor-bridge)

When the `cursor-worker` subagent (plugin: `claude-cursor-bridge`) is available,
treat Cursor as your **junior engineer**: you are the lead — you reason, decide,
and review — and you delegate legwork to the worker. Two kinds of delegation,
both **proactive and by default**:

**Investigation (read-only — works in any directory).** Offload codebase
legwork instead of spending your own context on it: locating where something
lives, summarizing a module, tracing how code flows, getting a quick gist of an
unfamiliar repo. The worker reads and reports; it changes nothing. (The worker
has none of this conversation's context, so ask a self-contained question.)

**Implementation (edits — inside a git repo).** Delegate substantial,
well-scoped mechanical coding: features/modules/functions from a clear spec,
whole test suites, scoped or repetitive multi-file refactors,
boilerplate/CRUD/config/codegen. Give a self-contained brief, then **review the
returned git diff** against the acceptance criteria — never rubber-stamp it.

**Do it yourself** (do NOT delegate): architecture / API / design decisions,
security-sensitive code (auth, crypto, secrets, payments), trivial one-liners or
quick fixes (the round-trip overhead isn't worth it), and debugging that depends
on this conversation's context.

Each delegation costs ~30–60s + real Cursor quota, so delegate substantial
legwork and chunky mechanical work — not one-file greps or one-line edits you
can finish instantly yourself. Plan and decompose the work yourself; hand the
mechanical execution to the worker.
