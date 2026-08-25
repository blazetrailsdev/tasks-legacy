---
title: "builder/association.js TDZ-crashes when imported as an entry module"
status: draft
updated: 2026-08-14
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

# `associations/builder/association.js` TDZ-crashes as an entry module

## Context

Importing the built module directly throws before any trails code runs:

````bash
node -e "import('./packages/activerecord/dist/associations/builder/association.js')"
# Cannot access 'Association' before initialization
```text

`packages/activerecord/dist/index.js` and `dist/base.js` both import cleanly, so
the graph only breaks when `builder/association.js` is the entry. Verified
present on `origin/main` (checked by stashing PR #6512's changes and rebuilding),
so it predates that PR — it was found while checking whether a new
`autosave-association.ts` → `builder/association.ts` import introduced a cycle.
It did not; the import was moved to `base.ts` anyway, beside
`include(Base, AutosaveAssociation)`, which is where Ruby's `included do` block
runs (`autosave_association.rb:153-155`).

The cycle runs `builder/association.ts` → `reflection.ts` → the association
classes → back to `builder/association.ts`, and `class Sub extends Association`
evaluates with `Association` still in TDZ. This is the shape CLAUDE.md's
"Call-time constant resolution" section describes: Ruby resolves the constant
named in a method body when the method RUNS (Zeitwerk autoloads it then), so
`association.rb` takes no load-order dependency on the subclasses at all.

## Why it matters

Any consumer that reaches the builder first — a deep import, a bundler entry, a
test importing the module under test directly — gets a hard crash rather than a
working module. A vitest run enters through the funnel module and masks it, so
the suite proves nothing here.

## Converged shape

Either break the `reflection.ts` edge out of `builder/association.ts`, or apply
the sanctioned zero-import slot (`associations/collection-proxy-slot.ts` and
`encryption/configurable-slot.ts` are the two existing instances) for whichever
constant closes the cycle. Verify BOTH directions with a plain-node import of the
built `dist/**.js` modules as entry modules, per CLAUDE.md — deferring the
subclass edges instead does not work, because then nothing loads the subclass
modules and their self-registration never runs.

## Acceptance criteria

- [ ] `node -e "import('./packages/activerecord/dist/associations/builder/association.js')"`
      resolves.
- [ ] `dist/index.js` and `dist/base.js` still resolve.
- [ ] No new slot module unless a plain import genuinely closes a cycle.
````
