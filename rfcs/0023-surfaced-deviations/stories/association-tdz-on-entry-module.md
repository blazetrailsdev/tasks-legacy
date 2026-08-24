---
title: "associations/association.js TDZ-crashes when imported as an entry module"
status: draft
updated: 2026-08-24
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
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

`packages/activerecord/dist/associations/association.js` throws when imported as
an **entry module**:

```console
$ node -e "import('./packages/activerecord/dist/associations/association.js')"
Cannot access 'Association' before initialization
```

Measured on `origin/main` (verified pre-existing during PR #7005 by rebuilding an
unmodified checkout of the file, so it is not introduced by that PR). Sibling
modules are clean at the same commit: `nested-attributes.js`,
`autosave-association.js` and `base.js` all import fine as entry modules.

This is the same class as the existing `0023-surfaced-deviations` stories
`builder-association-tdz-on-entry-module`, `relation-tdz-on-entry-module` and
`arel-nodes-index-tdz-on-entry-module`, but a different module —
`associations/association.js`, not `associations/builder/association.js`.

Per CLAUDE.md ("Call-time constant resolution"), an import cycle whose
participants include a `class Sub extends Super` evaluates `Sub` with `Super` in
TDZ when the graph is entered at the wrong module. A vitest run enters the funnel
module first and masks it, which is why the whole suite is green while this
throws — so the repro must be a plain `node` import of the **built** `dist/**.js`
module, never a test.

## Acceptance criteria

- `node -e "import('./packages/activerecord/dist/associations/association.js')"`
  resolves without throwing, and the same holds for every other module in the
  cycle it participates in.
- The cycle is broken by the sanctioned shape — a zero-import slot module
  (`configurable-slot.ts` / `collection-proxy-slot.ts` are the two existing
  instances) — or by removing the offending edge outright. Deferring the
  subclass edges with a slot per `extends` site is the alternative that does not
  work (nothing then loads the subclass modules, so self-registration never
  runs).
- No new `parity:api:calls` / `parity:api:calls:args` baseline rows and no new
  extra surface.
