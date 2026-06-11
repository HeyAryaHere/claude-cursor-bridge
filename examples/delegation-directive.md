# Delegating coding work to Cursor (claude-cursor-bridge)

When the `cursor-worker` subagent (plugin: `claude-cursor-bridge`) is available
**and** the working directory is a git repository, **prefer delegating
substantial mechanical coding to it instead of writing the code yourself.**
Plan and decompose the work yourself; hand the mechanical execution to the
worker.

**Delegate by default** when the task is mechanical and self-contained:
- Implementing a feature, module, or set of functions from a clear, settled spec
- Writing or scaffolding whole test suites for existing code
- Scoped or repetitive refactors and multi-file edits (renames, extractions,
  applying an established pattern across files)
- Boilerplate, CRUD, config, codegen

**Do it yourself** (do NOT delegate) when it is:
- A trivial one-liner or quick fix — the ~40s + Cursor-quota overhead isn't worth it
- Architecture / API / design decisions, or anything needing real judgment
- Security-sensitive: auth, crypto, secrets, payments
- Debugging that depends on this conversation's context
- Not in a git repository

When you delegate, give the worker a self-contained brief (it has none of this
conversation's context) and **review the returned git diff against the
acceptance criteria** — never rubber-stamp it.
