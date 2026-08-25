---
rfc: "0113-branch-and-guard-parity"
title: "Branch and guard parity — the axis the call gates cannot see"
status: active
created: 2026-08-18
updated: 2026-08-25
owner: "@deanmarano"
packages:
  - "activerecord"
  - "activemodel"
  - "activesupport"
  - "actionpack"
  - "actionview"
  - "arel"
  - "date"
  - "trailties"
clusters:
  - "arm-parity-tooling"
  - "missing-arm"
  - "invented-arm"
  - "arm-order"
  - "guard-parity"
related-rfcs:
  - "0111-error-class-message-parity"
  - "0023-surfaced-deviations"
  - "0084-wide-call-set-burndown"
  - "0095-call-argument-parity"
  - "0108-call-gate-false-positives"
priority: 3
---

# RFC 0113 — Branch and guard parity

## Summary

A ported body can call everything Rails calls, with exactly the right arguments,
and still drop an `elsif`. Both existing call gates stay green. **Control flow is
the axis nothing measures**, and it is where behaviour actually lives.

59 stories in this RFC describe a missing arm, an invented
arm, a wrong arm order, or a guard Rails does not have — roughly 7,500 estimated
LOC across ten packages. This RFC proposes a report-only arm-signature
comparison, then a burndown.

## Motivation

### What the current gates measure

- **RFC 0084** gates the _call set_ — which methods a body calls.
- **RFC 0095** gates the _call arguments_ — count, order, literal values, kwarg
  keys.

Both read `scripts/api-compare/call-mismatches-exclude/`. Neither observes how
many arms a body has, in what order, or with which guards, **because neither
measures anything but calls**. That is a property of what they were built to do,
not a defect in them.

### The backlog says so in its own words

`sqlite3-translate-exception-branch-set` records that PR #6375 _"converged the
argument lists in this function but deliberately did not touch its branch
structure"_. What it left behind: an invented `String or BLOB exceeded size
limit → ValueTooLong` arm Rails does not have, a missing
`SQLite3::BusyException → StatementTimeout` arm
(`sqlite3_adapter.rb:706`), and a different arm order — in a six-arm method,
with both gates green throughout.

`find-nth-from-last-index-base-and-order-guard` says its two divergences were
_"left alone there because they are behavioral rather than call-set rows"_.
Eight stories in the register carry explicit "the tooling cannot see this"
language.

### What a dropped guard costs

`Relation#compute_cache_version`'s loaded branch takes a `max` over timestamps.
Ruby's `Array#max` raises on a `nil`. The trails port:

```ts
.reduce((max: unknown, value: unknown) => {
  if (max == null) return value;
  if (value == null) return max;   // <- Ruby raises here
  …
})
```

Every call Rails makes is made. The guard that turns bad data into an exception
is not, so a nil timestamp silently yields a cache version instead of failing.
No gate in the repo can see that, and no reviewer catches it reliably either —
which is the argument for tooling rather than vigilance.

## Design

### The extraction already exists — this RFC promotes it

**Correction (2026-08-19).** This RFC originally proposed building an arm
extractor on both sides. That was wrong: RFC 0084 already shipped it, and an
earlier draft of this section sent readers to build something that is in the
tree today.

| Piece                                  | Where                                                                                                                                                               | Delivered by                                                 |
| -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| Ruby ordered control+call skeleton     | `extract-ruby-api.rb:2119` `collect_method_skeleton`, attached at `:684`, `:742`                                                                                    | `ordered-call-skeleton-stream-in-extractors`, PR #6161       |
| TS equivalent                          | `extract-ts-api.ts:2662` `extractSkeleton(node.body)` — `if` / `loop` / `try` / `throw` plus normalized call and property reaches, over the TypeScript compiler API | same story, PR #6161                                         |
| Ruby↔JS construct folding              | `compare.ts:303` `foldSkeletonTokens` — folds Ruby block-iteration onto `loop`                                                                                      | `fold-ruby-block-iteration-onto-loop-in-skeletons`, PR #6163 |
| Paired artifact, every compared method | `compare.ts:3905` → `output/call-skeletons{mode}.json`                                                                                                              | `call-sequence-parity-in-wide-ratchet`, PR #6152             |

`compare.ts:894-904` records the deliberate stopping point, and names what is
missing:

> Recorded for every compared pair, not just the flagged ones: this is the
> population call-sequence-parity seeds from, not a mismatch list. **Signal only
> (RFC 0084)** … and **nothing gates on it yet**. […] Those are set operations by
> construction; **a sequence needs its own merge rule**, and until
> call-sequence-parity decides what it is, carrying the body's own stream and
> nothing else is the honest shape.

So the tooling work here is **one decision and its consequences**: pick the
sequence merge rule, surface it as a report, measure the noise, then gate or
don't. It is carried by three stories in the `arm-parity-tooling` cluster
(`promote-call-skeletons-to-an-arms-report` →
`measure-arm-mismatch-noise-floor` → `seed-arm-parity-ratchet-or-record-ungated`),
and it runs over ~1,462 (package, file, method) rows across 12 packages — not
the 10-file population the retired prism-codegen scorer had.

Two consequences worth stating, because they cut the risk this RFC was written
against:

- **The population is already wide.** No sampling decision is needed to get
  coverage; the artifact covers every name-matched pair the compare walks.
- **One artefact class is already handled.** `foldSkeletonTokens` folds Ruby
  block-iteration onto `loop`, so that spelling difference should not appear in
  the report at all. If it does, that is an extraction bug in RFC 0084's
  tooling, not a false positive to baseline.

### What the gate deliberately does not do

**Do not compare predicate semantics.** That is where this drowns. Ruby's
`unless x` against an inverted TS `if (!x)` is the same arm; a `case` lowered to
a lookup table is the same dispatch; a guard clause hoisted above a block is the
same guard. Any of those, treated as a mismatch, produces noise faster than
signal.

RFC 0108 exists specifically to absorb call-gate false positives. This RFC
should learn from that rather than reproduce it: **ship report-only first**,
exactly as RFC 0095 did with its `naming` rows, and promote to gating only once
the noise floor is measured.

### Representative stories

| est-loc | story                                                 |
| ------: | ----------------------------------------------------- |
|     220 | `type-registry-missing-klass-arm-and-varargs-lookup`  |
|     150 | `type-decimal-has-no-numeric-rational-cast-arm`       |
|     150 | `scheme-key-provider-adds-an-encryptor-early-return`  |
|     120 | `sqlite3-translate-exception-branch-set`              |
|     120 | `time-change-local-and-utc-offset-arms-conflated`     |
|     120 | `date-datetime-cast-value-three-arm-dispatch`         |
|     120 | `compute-cache-version-max-swallows-nil`              |
|     120 | `discriminate-local-level-symbol-arm-from-string-arm` |

Note `sqlite3-translate-exception-branch-set` is itself a merge of three
separate stories that had all independently found the same method — evidence
that this class is currently being rediscovered rather than tracked.

## Non-goals

- **Predicate-semantics equivalence.** Explicitly out; see Design.
- **Gating in the first phase.** Report-only until the noise floor is known.
- **Ruby idioms that change arm _count_ legitimately.** A Ruby `||=` is one arm;
  its faithful TS spelling may be two. Those belong in
  `0082-ruby-ts-idiom-conversion-classes`, not here — and if they prove common,
  they are a reason to narrow this gate, not to baseline them.
- **Arms missing because the whole method is unported.** That is the 143-story
  unported-surface class in 0023; a method with no TS body has no signature to
  compare.

## Alternatives considered

- **Leave it to review.** This is the status quo and the reason the class has 71
  stories. Three separate agents independently filed the same sqlite3
  `translate_exception` finding; review is not catching it.
- **Extend RFC 0095 rather than a new RFC.** 0095 is about call _arguments_ and
  has a settled artifact, contract and vocabulary. Arm structure is a different
  measurement over a different extraction, and folding it in would blur a gate
  that is currently precise about what it claims.
- **Compare full ASTs.** Maximally sensitive, unusably noisy across two
  languages with different lowering. Rejected without prototyping.
- **Skip the tooling; just fix the 71 stories.** The honest fallback if the
  noise floor is bad — but attempted second, not first. Without a gate the class
  regrows silently, which is exactly how it reached 71.

## Rollout

Story IDs below are the `arm-parity-tooling` cluster; the 59 burndown stories
re-homed from `0023-surfaced-deviations` on 2026-08-18 and carry the
`missing-arm` / `invented-arm` / `arm-order` / `guard-parity` clusters.

1. **Phase 1 — decide the merge rule and ship the report.**
   `promote-call-skeletons-to-an-arms-report`. Reads the existing
   `output/call-skeletons.json`; no extractor work. Report-only.
2. **Phase 2 — measure the noise floor.**
   `measure-arm-mismatch-noise-floor`. A reproducible hand-audited sample
   classified real / lowering artefact / extraction bug, against the one-third
   tripwire below.
3. **Phase 3 — gate, or decide not to.**
   `seed-arm-parity-ratchet-or-record-ungated`. Both outcomes close the story;
   only one of them adds a baseline.
4. **Phase 4 — the specified methods.** Burndown stories that already name the
   exact arm list and order — `sqlite3-translate-exception-branch-set` is the
   model. No investigation needed, only the edit. **Not blocked on Phases 1-3**:
   these are correct fixes whether or not a gate ever lands.
5. **Phase 5 — the rest**, prioritised by whether the missing arm is a _raise_.
   A dropped raise is a silent wrong answer; a dropped fast path is a
   performance note. They are not the same severity and should not be worked in
   filing order.

## Verification

- `pnpm parity:api:arms:report` exists and runs in CI, reading the existing
  `output/call-skeletons.json`.
- **The Phase 3 decision is recorded with its number** — the measured
  artefact rate on the audited sample, whichever way it goes. A "we tried and it
  was too noisy" outcome with a figure attached is a successful RFC; one without
  a figure is not.
- If gated: the arms baseline is only-shrink and reaches 0 rows for the 59
  in-scope stories' methods.
- If ungated: all 59 stories reach `done`, and the Phase 4 subset is verified by
  arm-for-arm diff against the cited Rails `file:line` in review.
- No story converges by adding a baseline row.

## Open questions

1. **What artefact rate is acceptable?** **Recommendation:** set the tripwire
   before measuring, not after — if more than roughly one third of reported
   mismatches are lowering artefacts rather than real divergences, take the
   ungated path. Writing the threshold down first is what stops the gate being
   tuned until it agrees with us. Carried by
   `measure-arm-mismatch-noise-floor`.
2. ~~**Does the TS side need a real parser or will the existing extraction
   do?**~~ **Resolved 2026-08-19.** `extract-ts-api.ts` imports the TypeScript
   compiler API directly (`import * as ts from "typescript"`) and already calls
   `extractSkeleton(node.body)` alongside `extractCalls` / `extractCallSeq` /
   `extractCallArgs` (`:703-706`). Statement bodies are fully reachable and the
   skeleton is already emitted. No parser work, and Phase 1 is smaller than this
   RFC originally estimated.
3. **What is the sequence merge rule?** This is the one genuinely open design
   question, and `compare.ts:894-904` flags it as the reason nothing gates yet.
   Options are strict sequence equality, multiset equality plus a separate
   `reordered` verdict (the retired prism-codegen scorer's `matched` /
   `reordered` split), or control-token subsequence ignoring interleaved call
   reaches. **Recommendation:** decide it in
   `promote-call-skeletons-to-an-arms-report` and document why the other two
   were rejected, rather than deferring it into the gate.
4. **Should `rescue` / `catch` arms count?** Rails' `translate_exception`
   family makes them load-bearing, but TS `try/catch` shapes differ more than
   `if/else` does. **Recommendation:** include them in the report, exclude them
   from any gate until Phase 3 has data.
5. **Severity split.** Phase 5 proposes ordering by whether the missing arm
   raises. **Recommendation:** confirm that split is derivable from the story
   bodies; if it needs a human read of all 59, it is not worth the ordering.

## Changelog

- 2026-08-18: initial RFC, carved out of `0023-surfaced-deviations` by the
  backlog triage pass; 59 burndown stories re-homed.
- 2026-08-19: **corrected a false premise.** The RFC proposed building an arm
  extractor; RFC 0084 had already shipped one (PRs #6152, #6161, #6163) and the
  paired skeleton artifact is written for every compared pair today, signal-only.
  Design and Rollout rewritten around promoting it; open question 2 resolved;
  the sequence merge rule promoted to an open question in its own right; three
  `arm-parity-tooling` stories filed for the work that actually remains.
