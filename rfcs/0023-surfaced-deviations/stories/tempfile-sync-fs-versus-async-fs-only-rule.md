---
title: "Decide: Tempfile uses sync fs primitives against the async-fs-only rule"
status: draft
updated: 2026-08-24
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Needs a maintainer decision, not a port — filed so it is visible rather than
buried in PR #6984's description.

`packages/activesupport/src/tempfile.ts` uses the fs adapter's **synchronous**
primitives throughout (`getFs()`, `openSync`, `closeSync`, `chmodSync`,
`writeFileSync`, `readFileSync`, `unlinkSync`), against the "async fs only" hard
rule that trails task prompts carry.

That was a deliberate, reviewed choice in #6984 and it is load-bearing twice
over:

- Ruby's `Tempfile.open`/`Tempfile.create` run the block inline and return its
  value directly (`tempfile.rb:373-379`, `:446-464`). An `async` wrapper cannot:
  it defers a synchronous block's value behind a Promise, past every synchronous
  reader of it — the failure mode CLAUDE.md warns about under "async body defers
  scalar writes". The port's block form serves sync and async blocks and returns
  `T` un-Promised for the former, which requires synchronous creation.
- `File.atomic_write` (`core_ext/file/atomic.rb:20-52`) is synchronous and
  yields a synchronous block. It only converged onto `Tempfile.open` — deleting
  its hand-rolled name generation, payload buffer and rename-failure unlink —
  because `Tempfile` is synchronous. An async `Tempfile` sends `atomic.ts` back
  to a stand-in, or makes `atomicWrite` async and changes every caller
  (`cache/file-store.ts:195` among them).

So the two rules are in genuine tension here and one has to give. Either:

1. The async-fs rule is narrowed — it exists to keep I/O off the hot sync path,
   and a temp file created inside an already-synchronous Rails method
   (`atomic_write`) is arguably outside its intent. Write that carve-out down.
2. The async-fs rule wins — `Tempfile` goes back to `getFsAsync`, its block form
   returns `Promise<T>` only, and `atomic.ts` reverts to its stand-in with the
   reason restated at the call site.

Related: `fs-adapter-node-names-hide-rails-file-calls` and
`credit-fs-adapter-spellings-in-call-ratchet` both touch how the fs adapter is
allowed to be spelled.

## Acceptance criteria

- [ ] A decision is recorded: which rule governs a synchronous Rails method that
      needs the filesystem.
- [ ] If (1), the carve-out is written into CLAUDE.md / the prompt's hard-rules
      block, and `tempfile.ts` is left as-is.
- [ ] If (2), `tempfile.ts` moves to `getFsAsync`, `Tempfile.open`/`create`
      return `Promise<T>`, and `atomic.ts` reverts with the reason at the call
      site — and this story records why the story-mandated `T | Promise<T>`
      shape is not achievable.
