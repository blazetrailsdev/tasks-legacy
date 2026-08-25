---
title: "drop-sync-belongs-to-reader-and-destroy-preload-shim"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`Base#_preloadBelongsToForDestroyCallbacks` (`packages/activerecord/src/base.ts:3849`,
called from `destroy` at `base.ts:3923`) is a trails-only shim with no Rails
counterpart. It exists solely so a **synchronous** `before_destroy` /
`around_destroy` callback can read an unloaded `belongs_to` and get a record
rather than trails' async-reader Promise.

Added by #4107 (`53215bf5f`, story `belongs-to-sync-read-direct-destroy-callback`,
+53 LOC in `base.ts`).

### How it works, and why it is unsound

Before the destroy callback chain runs, it:

1. `Function.prototype.toString()`s every before/around destroy callback body
   (`beforeOrAroundCallbackSources`).
2. Stringifies **every instance method** on the model's prototype chain below
   `Base`, plus instance-own arrow-function fields
   (`expandCallbackSourcesWithHelpers`, `base.ts:637`).
3. Regex-scans that source text for each `belongs_to` reflection name,
   transitively expanding through helper methods
   (`referencesAssociationName`, `base.ts:620`).
4. Speculatively `loadTarget()`s each matched association — each wrapped in its
   own savepoint when a transaction is open.
5. Swallows any resulting error, resolving the association to `null`.

Source-code introspection driving query emission fails silently in ways no test
catches:

- **Minification breaks it.** Mangled method/property names stop matching, the
  preload no-ops, and the sync reader returns a Promise instead of a record.
- **False positives:** `belongs_to :firm` plus the word `firm` in a comment or
  string literal issues a real query.
- **False negatives:** `this[assocName]` or any computed access is invisible.
- **The `opaque` fallback loads every `belongs_to` on the model**, unconditionally.
- **It queries where Rails does not.** Rails loads lazily at dereference, so a
  callback that branches and never reads the association issues zero queries;
  trails issues one regardless.
- **The error swallow suppresses `StrictLoadingViolationError`**, so destroying a
  `strict_loading` record whose destroy callback reads an unloaded `belongs_to`
  silently proceeds where Rails raises. Rails' `load_target`
  (`vendor/rails/activerecord/lib/active_record/associations/association.rb:189-196`)
  rescues `RecordNotFound` **alone**; everything else propagates.

PR #4984 fixed only that last point by rethrowing `StrictLoadingViolationError`.
It was **closed unmerged** deliberately: patching the speculative-preload path
makes it look principled and extends its life, when the correct answer is that
nothing should be speculatively preloading. The failure modes above are silent,
so a partial fix is worse than removal.

### The convergent shape

Support **async only** for singular association reads. With
`await record.developer` (or the existing `record.loadBelongsTo("developer")`,
`associations.ts:1387`) there is nothing to preload: the strict-loading
violation surfaces from `loadTarget()` at the actual dereference — exactly where
Rails raises it — and the bug #4984 addressed stops existing rather than being
fixed.

Deleting the shim also deletes `expandCallbackSourcesWithHelpers`,
`referencesAssociationName`, the per-association savepoints, and the broad
error swallow (~100 LOC in `base.ts`).

### Scoping note

The sync singular reader is a public API surface, not only a destroy-callback
concern — `packages/activerecord/src/strict-loading-sync-reader.test.ts` is an
entire file about it, and `test-fixtures.ts` declares `loadBelongsTo` on many
canonical models. Audit the sync-reader dispatch sites and dependent tests
before deleting; if the removal exceeds the LOC ceiling, ship the
`_preloadBelongsToForDestroyCallbacks` deletion first and register the reader
removal separately. Do **not** fan out sibling PRs.

## Acceptance criteria

- `_preloadBelongsToForDestroyCallbacks` and its call site in `destroy` are
  deleted, along with `expandCallbackSourcesWithHelpers` and
  `referencesAssociationName` if they have no other callers.
- Singular association reads in destroy callbacks are async; no callback-source
  stringification or speculative preloading remains on the destroy path.
- A `strict_loading` record whose destroy callback reads an unloaded
  `belongs_to` raises `StrictLoadingViolationError`, arising from `loadTarget()`
  at dereference rather than from a special case in the destroy path.
- Unregistered-target-class and missing-FK-row cases behave as Rails does:
  `load_target` rescues `RecordNotFound` only.
- No regression to the CPK destroy tests the original error swallow protected
  (`primary-keys`, `persistence`, `belongs-to-inverse-seed-composite-pk`).
- `parity:api` and `parity:test` deltas non-negative.
