---
rfc: "0114-collection-proxy-decomposition"
title: "collection-proxy.ts decomposition: converge the proxy back onto its association"
status: closed
created: 2026-08-19
updated: 2026-08-22
owner: "@deanmarano"
packages:
  - "activerecord"
clusters:
  - "api-compare"
related-rfcs:
  - "0107-relation-ts-decomposition"
  - "0075-collection-association-target-fidelity"
  - "0112-one-rails-thing-n-trails-things"
priority: 1
---

## Summary

`packages/activerecord/src/associations/collection-proxy.ts` is **1,556 code
lines** (2,938 raw) against
`vendor/rails/activerecord/lib/active_record/associations/collection_proxy.rb`'s
**148 code lines** (1,163 raw — the file is 87% documentation). Per matched
method that is **10.5x** lines-per-method inflation, the worst in the repo once
assembly-point files are excluded. Post-cleanup `relation.ts` measures 2.7x and
the repo-wide average is 1.85x; nothing else with >=15 matched methods and >=100
Ruby lines is above 7x.

This RFC burns that down, modelled on RFC 0107 (relation.ts decomposition).

The ratio is not explained by the port of `collection_proxy.rb` being verbose.
Rails' `CollectionProxy` is thin **by construction**: of its 31 methods, 24 are
one- or two-line delegations to `@association`, and everything else is
`delegate(*delegate_methods, to: :scope)`. Every one of those bodies is
identical in shape:

```ruby
def delete_all(dependent = nil)   # :474
  @association.delete_all(dependent).tap { reset_scope }
end
def size;      @association.size;      end      # :782
def empty?;    @association.empty?;    end      # :831
def include?(record); !!@association.include?(record); end   # :927
def replace(other_array); @association.replace(other_array); end  # :391
def scope;     @scope ||= @association.scope;  end   # :949
def records;   load_target;            end      # :1024
```

trails' proxy instead **re-implements those bodies in place** — and
`collection-association.ts` already carries the very methods they duplicate.

## The measurement (2026-08-19)

Fresh `pnpm build` + `pnpm parity:api --calls` + `pnpm parity:api:extra
--package activerecord --json`. Every member of `collection-proxy.ts`
classified by line span, blank/comment lines stripped:

| bucket                                                                | code lines | %   |
| --------------------------------------------------------------------- | ---------- | --- |
| E — mutation terminals duplicated from `CollectionAssociation`        | **408**    | 26% |
| B — `:through` machinery                                              | **285**    | 18% |
| H — finder / `delegate ... to: :scope` overrides Rails does not write | **210**    | 13% |
| G — the delegate table + module tail                                  | 139        | 9%  |
| C — construction and eager scope seeding                              | 114        | 7%  |
| PREAMBLE — types, thenable interface, slot setter, helpers            | 112        | 7%  |
| D — load / target-merge machinery                                     | 95         | 6%  |
| J — fields and `target`/`owner`/`reflection` accessors                | 74         | 5%  |
| A — Enumerable / array-likeness block                                 | 62         | 4%  |
| I — `null_scope?` / `find_from_target?` / `inspect` / `pretty_print`  | 31         | 2%  |
| F — strict-loading helpers                                            | 26         | 2%  |
| **total**                                                             | **1,556**  |     |

Buckets J, I and the honest half of PREAMBLE (~180 lines together) are the part
that measures ~1:1 against Rails and is **not in scope**. The ~1,400 lines above
them are.

`parity:api:extra` scores the file **6 novel / 32 moved / 4 allowlisted**
(38 extras). The 6 novel names — `every`, `flatMap`, `new`, `reduce`, `setIds`,
`some` — are 4 Enumerable spellings (bucket A) plus `new` (a `build` alias) and
`setIds` (`ids_writer`). The 32 moved names name their own destination, and
they cluster exactly on buckets A/H:

- **`relation/delegation.rb`** (`delegate ... to: :records`): `length`,
  `slice`, `transaction`
- **`relation/finder_methods.rb`**: `firstBang`, `lastBang`, `takeBang`,
  `exists`
- **`relation.rb` / `querying.rb`**: `many`, `one`, `toArray`, `insert`,
  `insertBang`, `insertAll`, `insertAllBang`, `upsert`, `upsertAll`, `select`,
  `firstOrInitialize`, `firstOrCreate`, `firstOrCreateBang`, `load`
- **`associations/collection_association.rb`** / `through_association.rb`:
  `transaction`
- Enumerable spellings with a homonym elsewhere in the tree (`any`, `at`,
  `map`, `filter`, `forEach`, `keys`, `entries`, `indexOf`, `clone`, `then`,
  `appendBang`)

### The call-parity baselines are empty, and must stay empty

There is **no** `call-mismatches-exclude/activerecord/associations/collection-proxy.json`
shard, and a forced regeneration (`API_COMPARE_FORCE=1 pnpm parity:api --calls`)
puts **0 rows** for this `tsFile` in both `output/call-mismatches.json` (747
rows repo-wide) and `output/call-arg-mismatches.json` (488 rows). All 37 matched
members are green on both gates today.

So there is no baseline debt to fold in — but that is a **constraint, not a
freebie**: the gates score calls made inside Rails-named bodies, and every story
here rewrites Rails-named bodies. No story may close by adding a row. If a
converged body would trip either gate, the body is wrong, not the baseline.

This is also _why_ the bloat never surfaced on a gate: 1,400 lines of it live in
private `_`-prefixed helpers (`_buildThroughScope`, `_throughOwnerCols`,
`_execLoad`, `_removeFromTarget`, …) that `parity:api:extra` cannot see, called
from bodies whose Rails counterpart makes no such call — an omission the call-set
gate scores as green because it only ratchets _missing_ Rails calls, never extra
trails ones.

## The re-measurement (2026-08-21)

Same method, same commands (`pnpm build`, `API_COMPARE_FORCE=1 pnpm parity:api
--calls`, `pnpm parity:api:extra --package activerecord`), against `origin/main`
at `fc521e1b3`:

```console
$ wc -l packages/activerecord/src/associations/collection-proxy.ts
1520
$ grep -vcE '^\s*(//|/\*|\*|$)' packages/activerecord/src/associations/collection-proxy.ts
675
```

**1,556 → 675 code lines: a 57% burndown.** Per matched method the ratio against
`collection_proxy.rb`'s 148 code lines is now **~4.6x**, down from 10.5x — inside
the band `relation.ts` sat in after RFC 0107 finished with it.

| bucket                                                               | 2026-08-19 | 2026-08-21 | retired by                                                                                                                                                           |
| -------------------------------------------------------------------- | ---------- | ---------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| E — mutation terminals duplicated from `CollectionAssociation`       | 408        | **158**    | `clear`/`include`/`replace`/bulk-insert/target-replay stories (F1)                                                                                                   |
| H — finder / `delegate ... to: :scope` overrides                     | 210        | **157**    | `retire-collection-proxy-bang-finder-and-first-or-overrides`, `delegate-select-bang-to-scope`                                                                        |
| PREAMBLE — types, thenable interface, slot setter, helpers           | 112        | 86         | partial                                                                                                                                                              |
| J — fields and `target`/`owner`/`reflection` accessors               | 74         | 85         | not in scope (grew as bodies moved onto the shared accessors)                                                                                                        |
| G — the delegate table + module tail                                 | 139        | 60         | `derive-collection-proxy-delegate-list-from-mixin-keys` (F7, #6756)                                                                                                  |
| C — construction and eager scope seeding                             | 114        | 51         | `collection-proxy-initialize-is-five-lines` (F5, #6745)                                                                                                              |
| I — `null_scope?` / `find_from_target?` / `inspect` / `pretty_print` | 31         | 31         | not in scope (~1:1 with Rails)                                                                                                                                       |
| D — load / target-merge machinery                                    | 95         | 29         | `retire-collection-proxy-load-and-merge-block` (F6, #6773)                                                                                                           |
| B — `:through` machinery                                             | 285        | **11**     | `move-through-owner-attribute-helpers-to-through-association`, `delete-dead-collection-proxy-build-through-scope`, `retire-collection-proxy-ensure-through-writable` |
| A — Enumerable / array-likeness block                                | 62         | 6          | `retire-collection-proxy-enumerable-block` (F4, #6759)                                                                                                               |
| **total**                                                            | **1,556**  | **675**    |                                                                                                                                                                      |

`parity:api:extra` now scores the file **1 novel / 7 moved** (was 6 novel /
32 moved / 4 allowlisted). The single novel name is `new` — Rails'
`alias_method :new, :build` (`collection_proxy.rb:321`), which is a real Rails
member the matcher cannot see through an alias, not invented surface.

### The standing constraint still holds — with one finding

`output/call-mismatches.json` carries **0 rows** for this `tsFile`, and there is
still no `call-mismatches-exclude/.../collection-proxy.json` shard.

`output/call-arg-mismatches.json` carried **one** row, and per this RFC's own
rule that is a finding rather than a baseline: `replace` was declared
`replace(records: T[])` against Rails' `def replace(other_array)`
(`collection_proxy.rb:391`). Ported parameters keep the Rails identifier, so the
parameter was renamed to `otherArray` in the re-measurement PR and the row is
gone. Both artifacts are now empty for this file.

### What is left, and is it worth a campaign?

Yes — but a short one. Buckets B, A, D, G and C have converged or are within a
story of it; F2 is down to `_pushThrough` (11 lines, already owned by
`0023-surfaced-deviations/retire-push-through-for-association-concat`). Two
buckets still hold real residue, and neither is large:

- **E, 158 lines** — but most of it is now genuine delegation plus TypeScript
  overload signatures on `build`/`new`/`create`/`create!`, which is irreducible.
  The re-implementations that survive are `_raiseOnTypeMismatch` (15 lines, a
  second copy of `Association#raise_on_type_mismatch!`, `association.rb:257` —
  Rails' `<<` does not type-check at the proxy at all), `appendBang` (17),
  `_wireInverseTarget` (6), and `push`'s `:through` arm (the F2 remnant).
- **H, 157 lines** — `transaction` (10; Rails has no `CollectionProxy#transaction`
  — it is `CollectionAssociation#transaction`), `clone` (12), `equals` (11),
  `load`/`reload`/`reset`/`toArray` (37), and the `calculate`/`pluck`
  `disable_joins` arms (already owned by
  `0106-wide-call-set-direct-burndown/djar-eager-chain-ids-drop-disable-joins-arms`).

Three stories are filed for the residue
(`retire-collection-proxy-raise-on-type-mismatch`,
`retire-collection-proxy-append-bang-and-wire-inverse-target`,
`move-collection-proxy-transaction-and-clone-to-their-rails-seats`). When those
land the file is ~600 lines at ~4x, dominated by PREAMBLE + J + I — the part
this RFC declared out of scope on day one — and **0114 should close** rather
than chase the remainder.

The buckets below are the 2026-08-19 findings as originally written, kept for
provenance.

## Findings

### F1 — 408 lines re-implement `CollectionAssociation` bodies that already exist

`collection-association.ts` already has `deleteAll` (`:564`),
`deleteOrNullifyAllRecords` (`:608`), `destroyAll` (`:623`), `delete` (`:634`),
`destroy` (`:647`), `size` (`:669`), `isEmpty` (`:692`), `replace` (`:717`),
`isInclude` (`:811`), `deleteRecords` (`:1217`),
`computeNullifiedOwnerAttributes` (`:1228`), `nullifyAllRecords` (`:1240`),
`mergeTargetLists` (`:1333`), `difference`/`intersection` (`:458`/`:463`) —
the full Rails set. `size`, `isEmpty`, `delete` and `create` on the proxy
already delegate correctly and prove the seam works.

The rest do not:

- `clear()` (`collection-proxy.ts:1627`, 38 lines) reimplements
  `delete_or_nullify_all_records` — the `null_scope?` short-circuit, the
  `:dependent` collapse, the through arm's `deleteRecords`, and the target
  reset — where Rails is `delete_all; self` (`collection_proxy.rb:1066-1069`).
  Its private half is `_buildNullifyUpdates` (`:1578`, 22 lines — that is
  `computeNullifiedOwnerAttributes`), `_decrementCounterCache` (`:1554`, 18) and
  `_removeFromTarget` (`:1530`, 18).
- `isInclude()` (`:1711`, 46 lines) reimplements the type-mismatch guard,
  the in-memory scan and the `exists?` fallback that
  `CollectionAssociation#include?` (`collection_association.rb:258-270`) owns.
  Rails' proxy body is `!!@association.include?(record)`.
- `replace()` (`:1971`, 15) plus `_replaceRecords` (17),
  `_replaceCommonRecordsInMemory` (5), `_difference` (3), `_intersection` (3),
  `_replaceTransaction` (8) reimplement `collection_association.rb:242-256` and
  `:418-437`. Rails' proxy body is `@association.replace(other_array)`.
- `destroyAll()` (`:2089`), `deleteAll()` (`:2511`), `push`/`appendBang`
  (`:1232`/`:2619`), `_raiseOnTypeMismatch` (`:1480`), `_invalidateAssociationIds`
  (`:1430`), `_wireInverseTarget` (`:1174`), `_assertBulkInsertable` (`:1063`)
  are the same pattern.

### F2 — 285 lines of `:through` machinery on a class Rails keeps through-agnostic

`collection_proxy.rb` mentions `:through` only in five documentation comments —
it contains no through-specific _code_ at all, because the proxy hands every
call to `@association` and `HasManyThroughAssociation` is what answers. trails
carries `_buildThroughScope` (`:2233`, 110 lines), `_throughOwnerPolymorphic`
(`:1317`, 43), `_throughOwnerCols` (`:1256`, 35), `_ensureThroughWritable`
(`:940`, 29), `_includeInMemoryThrough` (`:1452`, 23), `_throughOwnerAttrs`
(`:1378`, 11), `_pushThrough` (`:1418`, 11), `_resolveThroughModel` (`:2074`, 9),
`_throughAssociation` (`:1414`, 3), and the `isThrough`/`_isThrough` pair (6).

Every one belongs in `associations/has-many-through-association.ts` /
`through-association.ts` / `association-scope.ts`, which already exist. The
`_buildThroughScope` half is already filed as
`0023-surfaced-deviations/converge-collection-proxy-through-scope-builder-to-association-scope`
and `_pushThrough` as `0023/retire-push-through-for-association-concat`; this
RFC owns the remainder and depends on those.

### F3 — 210 lines of overrides for methods Rails delegates to `scope`

Rails' `delegate_methods` list (`collection_proxy.rb:1128-1137`) routes
`:scoping, :values, :insert, :insert_all, :insert!, :insert_all!, :upsert,
:upsert_all, :load_async` plus every `QueryMethods`/`SpawnMethods` public method
to `scope`; `first!`, `last!`, `take!`, `many?`, `one?`, `exists?`,
`first_or_create` and friends come from the inherited `Relation` unchanged,
because `records` is `load_target` and `loaded?` is `@association.loaded?` —
that is the _whole_ mechanism by which a Rails proxy answers them off the target.

trails overrides all of them anyway: `exists` (`:1900`, 24 lines), `many`
(`:1871`, 12), `firstBang`/`lastBang`/`takeBang` (7 each), `one` (3),
`firstOrInitialize`/`firstOrCreate`/`firstOrCreateBang` (6 each), the six
`insert*`/`upsert*` bodies that call `super` behind an invented
`_assertBulkInsertable` guard (38 total), `select` (`:2690`, 15), `clone`
(`:2721`, 13), `transaction` (`:2650`, 10), `toArray`/`load`/`length`/`records`
(27).

### F4 — the Enumerable block: 62 lines, 4 of the 6 novel names

`collection-proxy.ts:383-497` hand-writes `length`, `[Symbol.iterator]`, `at`,
`map`, `filter`, `forEach`, `some`, `every`, `any`, `slice`, `reduce`,
`indexOf`, `flatMap`, `keys`, `entries` over `this._target`. Rails gets all of
them from `include Enumerable` (`relation.rb:67`) plus `delegate ... to:
:records` (`relation/delegation.rb:99-102`).

RFC 0107 already built the mechanism and retired the same block from
`relation.ts`: `RECORD_DELEGATES` + `DelegationMethods` +
`delegateRecordMethodSync` in `packages/activerecord/src/relation/delegation.ts`
(stories `give-relation-enumerable-surface-one-mechanism`,
`resolve-relation-enumerable-delegation-surface`, both done). `delegateRecordMethodSync`
exists **specifically** to give a loaded CollectionProxy the synchronous path.
The proxy was never migrated onto it.

### F5 — 114 lines seeding Relation state the Rails proxy never has

Rails' `initialize` is five lines: store `@association`, `super klass,
klass.arel_table`, `extend(*extensions)`. The proxy carries no where-clause, no
order, no `@values` — every query method it answers goes through the memoized
`scope` (`:949`), which is `@association.scope`, built lazily and thrown away by
`reset_scope`.

trails' constructor (`:563`, 61 code lines) instead eagerly builds a relation
(`hasManyScope(...)` or `_buildThroughScope()`) and `initializeCopy`s it onto
the proxy's own inherited Relation state, with a `seedNone()` path that must
then hand-unset `_isNone` because the copy would otherwise poison
`update_all`'s `return 0 if @none`, and a `_deferredFkError` field to replay an
`ArgumentError` that Rails never raises early because it never builds the scope
early. `_targetModelFor` (14), `static create`/`_create` (15), the
`_setAssociationRelationCtor` preamble (24) and `_cachedNamedScopeRelation`
(`:2202`, 13) hang off the same decision.

This is the root cause of F3: once the proxy carries its own seeded relation
state, `super.insert(...)` and `firstBang()` and `clone()` all have something to
run against, so overriding them looks reasonable. Delegating to `scope` the way
Rails does removes the reason for every one of them.

### F6 — 95 lines of load/merge machinery beside the association's own

`_execLoad` (`:704`), `_findTargetViaAssociation` (`:729`), `toArray` (`:742`),
`load` (`:775`), `_mergeTargetLists` (`:825`, 16), `_refreshUnchangedAttributes`
(`:853`, 13), `_identityFor` (`:878`, 10), `_staleWrapper` (`:902`),
`_hydrateFromPreload` (`:498`). `CollectionAssociation#mergeTargetLists` already
exists at `collection-association.ts:1333` (Rails
`collection_association.rb:335`) — this is a second, divergent copy.

### F7 — 139 lines to express Rails' 12-line delegate table

`collection-proxy.ts:2755-2929` hand-transcribes
`QueryMethods.public_instance_methods(false)` as an 83-line string array and
`SpawnMethods`' as a 7-line one, then builds the property descriptors. Rails
computes both by reflection.

TypeScript cannot reflect over a Ruby module, but trails does not need it to.
The mixin objects `include()` mixes into `Relation` are exported and their keys
ARE the list: `QueryMethodBangs` (`relation/query-methods.ts:2651`) and
`SpawnMethods` (`relation/spawn-methods.ts:138`).
`Object.keys(QueryMethodBangs)` is the faithful spelling of that half of
`QueryMethods.public_instance_methods(false)`.

The non-bang half is not yet a module — those members still sit on `Relation`
itself, and RFC 0107's `fan-out-query-methods-*` stories are what move them into
`relation/query-methods.ts`. So this RFC converges what is derivable today and
pins the residual hand-list with a test asserting it holds only names no
exported mixin carries; the list shrinks to nothing as 0107's fan-out lands.
Until then the hand-list is a maintenance hazard as well as bloat: a query
method added to a mixin is silently not delegated.

## Nothing here is ratifiable as a TS shortcoming

Each finding above has a live, in-repo mechanism it converges onto — the
association object (F1, F2, F6), `scope` delegation (F3, F5), RFC 0107's
`RECORD_DELEGATES` (F4), the mixin objects' own keys (F7). Two shapes in the
file _are_ genuine language deviations and are explicitly **out of scope**,
already carrying `@noRailsEquivalent PERMANENT` receipts: the thenable
(`then`/`catch`/`finally`, `:78-95`) and `[Symbol.asyncIterator]` (`:2707`).
The `[Symbol.iterator]` tag at `:388` is _not_ one of them — it is F4's, and it
goes with the block.

No story in this RFC closes by writing a better justification. If a story cannot
converge, it is `pnpm tasks block`ed with the specific blocker.

## Out of scope / follow-on

Measured the same way, and deliberately left for their own campaigns so this
one stays reviewable:

- `associations/collection-association.ts` — 910 lines against
  `collection_association.rb`'s 364 (2.5x). It is the **destination** for F1,
  F2 and F6, so it will grow here; its own ratio is a later problem.
- `associations.ts` — 9.3x, with 11 novel + 38 moved names.
- `base.ts` — 2,836 lines against a 64-line `base.rb`.

## Sequencing

F4 and F7 are self-contained and land first (no behavioural surface). F5 is the
keystone — F3's overrides only become deletable once the proxy stops carrying
seeded relation state — so F3's stories depend on it. F1, F2 and F6 are
independent of the F5/F3 axis and of each other, and can run in parallel: each
moves a distinct body into `collection-association.ts`,
`has-many-through-association.ts`, or both.

Ordering: **F4 → F7 → (F1 ‖ F2 ‖ F6) → F5 → F3**.
