---
rfc: "0000-extra-surface-gating-rollout"
title: "Extra-surface gating rollout: an enrollment contract for parity:api:extra"
status: draft
created: 2026-08-24
updated: 2026-08-24
owner: "@deanmarano"
packages:
  - "arel"
  - "globalid"
  - "abstractcontroller"
  - "i18n"
  - "activerecord"
clusters:
  - "api-compare"
related-rfcs:
  - "0080-api-compare-jsdoc-metadata"
  - "0117-arel-extra-surface-burndown"
  - "0084-wide-call-set-burndown"
---

## Summary

`parity:api:extra` gates exactly one package. `scripts/api-compare/extra-surface-mark.json`
reads `{ "arel": { "novel": 0, "total": 63 } }`, and
`scripts/api-compare/extra-surface-mark.ts:41` pins `GATED_PACKAGES = ["arel"]`.
The ratchet itself (`lint-extra-surface-ratchet.ts`, CI at
`.github/workflows/ci.yml:1481`) is finished and correct. What is missing is an
**enrollment contract**: which package next, at what threshold, and what has to
be true before its door closes.

RFC 0117 proved the burndown is tractable — arel went from 89 novel on
2026-08-22 to 0 on 2026-08-24. This RFC turns that one-off into a rollout, and
closes the two holes that would otherwise let enrollment become a relabelling
exercise.

## The enrollment contract

The three slices `parity:api:extra` reports are not the same kind of thing and
must not be gated the same way.

| slice           | what it is                                                             | gate                                    |
| --------------- | ---------------------------------------------------------------------- | --------------------------------------- |
| `novel`         | name found nowhere in Rails — genuinely invented API                   | **held at 0** to enroll                 |
| `moved`         | name exists in Rails, wrong `.rb` — usually _our file layout_ diverges | **ratcheted only**, never required at 0 |
| `noCounterpart` | extras in TS files no Rails file maps onto at all                      | **files held at 0** to enroll           |

`moved` is never an enrollment condition. RFC 0117's triage model §2 is explicit
that most moved names are fixed wholesale in `PATH_SEGMENT_ALIASES` /
`RUBY_FILE_TS_OVERRIDES` — a `scripts/parity/conventions.ts` change with zero
trails source change. Requiring `moved: 0` would make every package's door wait
on a mapping campaign that is not the porting agent's work, and the gate's value
is almost entirely in `novel`. arel's existing pin already encodes this
correctly: `total` = novel + moved, so pinning `total` ratchets `moved` without
demanding it.

`noCounterpart` is the one place this RFC extends arel's contract.
`noCounterpartExtras` is already counted inside `novel`/`total`
(`extra-surface.ts:1734-1735`, scored with an empty allowed set per
`extra-surface.ts:1439-1440`), so it needs no separate burndown — but it is the
leading indicator. One new unmapped TS file dumps its entire public surface into
`novel` in a single commit, and a gate that reds for that reason is reding for
"we did not wire the file map", not "we invented API". Agents learn to read that
red as noise, which is how a ratchet dies.

> **Contract.** A package enrolls when `novel = 0` and `noCounterpartFiles = 0`.
> Its mark is `{ novel: 0, total: <measured>, noCounterpartFiles: 0,
allowed: <measured> }`, every dimension only-shrink. `moved` is ratcheted
> through `total` and is never an enrollment condition.

This matters for exactly the packages that look cheapest:

| package            | novel | noCntrp | the real work              |
| ------------------ | ----- | ------- | -------------------------- |
| abstractcontroller | 4     | 29      | file mapping, not deletion |
| globalid           | 5     | 13      | file mapping               |
| i18n               | 7     | 49      | file mapping               |

All three are cheap in `novel` and expensive in `noCntrp`. Enrolling them on
`novel` alone pins them on a foundation that re-opens the first time someone
maps one of those files.

## `@noRailsEquivalent` — what stops enrollment becoming a tagging exercise

Allowed extras are **subtracted** from the reported totals, so a package can
reach `novel: 0` by tagging rather than by deleting, and the gate would applaud.
The population is already moving: `grep -c '@noRailsEquivalent' packages/*/src`
returns **255** today against the **209** the last full run reported (not
identical measures — the report counts only tags on names that still flag — but
the direction is the point). Nothing anywhere stops that number rising.

**1. Ratchet `allowed` as a mark dimension. This is the load-bearing fix.** With
`allowed` only-shrink, tagging your way to `novel: 0` becomes arithmetically
impossible _after_ enrollment: a new tag raises `allowed` and reds the gate in
the same PR that added it. Before enrollment the constraint is a hard tag budget
written into that package's burndown story — exactly what RFC 0117 did ("arel
may finish with at most 8 tags; a story that would need a 9th is blocked, not
tagged").

This is already chartered. RFC 0080 closed with "the signal is advisory — a
ratchet or gate is a follow-up once it is classified", and classification is now
done (the run reports 0 unclassified). This RFC is that follow-up.

**2. `CONVERGEABLE` must carry a filed story id.** Grammar
`@noRailsEquivalent CONVERGEABLE <story-slug> — <reason>`, slug shape checked in
the extractor. The live failure this closes: the Tempfile receipts at
`packages/activerecord/src/encryption/encrypted-file.ts:144` and
`packages/activerecord/src/tasks/postgresql-database-tasks.ts:213` were both
classed PERMANENT when the truth was "nobody wrote the shared helper yet" — a
CONVERGEABLE with no story to point at, so PERMANENT was the only label that did
not lie about _tracking_. Story `port-ruby-tempfile-block-form` exists now.

**3. Open `CONVERGEABLE` tags do NOT block enrollment**, but their count is
ratcheted downward-only for gated packages. This is a deliberate inversion. If
an open CONVERGEABLE closes the door, PERMANENT becomes the cheap label and
CONVERGEABLE the expensive one, and the Tempfile misclassification gets
manufactured on purpose. We want CONVERGEABLE cheap and honest and PERMANENT
expensive. RFC 0080's re-audit cadence — every two quarters, or whenever
`tagged.total` grows by 10 — is what keeps PERMANENT honest; it has apparently
never fired, and this RFC re-arms it via the `allowed` ratchet.

**4. No per-package tag cap.** A standing cap is a number with no relationship
to how much genuine TS shortcoming a package hits, and the first time one binds,
the escape is to relabel surface `@internal` — which _removes it from the
compared population entirely_ (`extra-surface.ts:29-30`, `:952`), strictly worse
than a tag that is at least reported in the `Allowed` column. The `allowed`
ratchet applies the same pressure with no magic number and no incentive to hide.
Hard budgets stay where RFC 0117 put them: inside one campaign, scoped to it.

**5. `@internal` counts must become visible.** RFC 0080 drew the
`@internal`/`@noRailsEquivalent` line deliberately, and `@internal` is the one
escape the gate cannot see. Without reporting and ratcheting it for gated
packages, every other control here is a funnel into a blind spot.

## JSON side-channel → JSDoc

RFC 0080 executed this direction once, converting 27 method-level
justifications out of `extra-surface-allow.json` into `@noRailsEquivalent`. What
it did not convert, and why:

**Aggregates — stay JSON.** `extra-surface-mark.json`,
`test-compare/assertion-mismatch-mark.json`, and the
`call-mismatches-unreviewed/` high-water shards are counts over a package. A
count has no declaration to hang on. Do not propose converting these.

**Not TS declarations — stay JSON.** `schema-compare/invented-baseline.json` is
a list of _table names_; `scripts/parity/unported-files/baseline.json` is
patterns over _Ruby_ file paths. Neither has a TS declaration to tag at all.
This is a third bucket the per-site/aggregate split does not cover, and naming
it prevents a future RFC re-litigating it.

**Already drained — delete, do not migrate.** `arity-exclude.json`,
`inheritance-exclude.json` and `body-pins.json` are all `[]` (3 bytes each).
There is nothing to convert; the work is removing the file, the loader and the
validation.

**ESLint trees — out of scope, named so they are not forgotten.**
`eslint/no-standalone-associations-exclude.json`,
`rails-error-parity-exclude.json`, `no-explicit-any-{src,test}-exclude.json` are
per-site facts, but ESLint's native inline form is `eslint-disable-next-line`
with a description, not a JSDoc tag. That is a lint-convention decision with its
own tradeoffs and no shared mechanism with this family.

### `call-mismatches-exclude/**` — 558 rows, 201 shards, 832K

Per-site facts by construction: a row is `package + tsFile + rubyName + call +
reason` (`call-mismatch-baseline.ts:63-76`), a fact about one method, and its
inline twin already exists and is already load-bearing — `@missingRailsCall`
and `@missingRailsArgs` suppress the flag through the extractor
(`missing-rails-call-tags.ts:8-12`). 417 rows are `kind: "calls"`, 141 are
`kind: "args"`.

**These rows migrate. They do not migrate mechanically, and the reason is
policy, not taste.** `missing-rails-call-tags.ts:20-46` defines both seed
strings and states that a tag carrying one is deliberately _not_ a
justification: `justifies()` rejects it, "so if it suppressed, the whole
baseline would be blessed by prose nobody wrote". RFC 0083 already stopped
`parity:api:build` from minting such a tag — it "only writes a tag it has
curated prose to migrate". Nearly every row in the tree today carries a seed
string verbatim.

So the migration contract is **review-then-migrate, per cluster**:

- a seeded row is either **converged** (make the TS body call what Rails calls;
  delete the row) or **reviewed into real per-entry prose and migrated to a tag**
  at the declaration;
- never bulk-moved. A conversion pass that carried seed prose inline would
  produce 558 non-suppressing tags across 201 files and a red gate;
- `parity:api:build` is the tool, not a new one;
- the debt count survives the migration in the `call-mismatches-unreviewed/`
  marks, which stay JSON as aggregates.

This makes the migration cost per row identical to burning the row down, which
is the correct price: converting a _reviewed_ row to a tag is strictly better
than baselining it, and the ledger shrinks either way.

## Package order

- **Wave 0 — free:** `did-you-mean` (0 novel / 0 total, enroll as-is),
  `actionpackversion` (1 novel, 1 file).
- **Wave 1 — arel-shaped, days each:** `globalid` (5/13),
  `abstractcontroller` (4/29), `i18n` (7/49). Dominated by `noCounterpart`.
- **Wave 2 — chartered later against this contract:** `rack` (44),
  `actionview` (44), `activemodel` (68, had RFC 0115 momentum).
- **Wave 3 — own RFC each:** `actioncontroller` (70), `trailties` (141),
  `actiondispatch` (190), `activesupport` (295), `activerecord` (398).

`activerecord-test-support` (30 files, 89 novel, 6 moved, 82 noCntrp, **0
allowed**) is deliberately unscheduled. Almost everything in it is
invented-by-measurement and it mirrors Rails' _test helpers_, which are largely
not Rails public API. Whether it belongs in the compared population at all is
the first question, and burning it down before answering could waste 89 names of
work.

## Non-goals

- No change to the ratchet's only-shrink contract or its no-reseed rule.
- No widening of `GATED_PACKAGES` beyond waves 0 and 1 in this RFC.
- No change to the `@internal` semantics — only to its _visibility_.
- No ESLint exclude-tree conversion.
