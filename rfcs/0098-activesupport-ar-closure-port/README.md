---
rfc: "0098-activesupport-ar-closure-port"
title: "activesupport AR-closure porting"
status: active
created: 2026-08-10
updated: 2026-08-24
owner: "@your-handle"
packages:
  - "activesupport"
clusters: []
related-rfcs:
  - "0072-api-compare-parity-burndown"
  - "0092-parity-tools-consolidation"
priority: 2
---

# activesupport AR-closure porting

## Problem

activesupport sits at 46.9% API parity (1,075/2,292, main 5c54182f1) and is the only AR-dependency package below 100%. The audit `~/.btwhooks/data/github/blazetrailsdev/trails/audits/activesupport-ar-gaps-20260810T143915Z.md` (2026-08-10) cross-referenced every missing member against the `require "active_support/…"` closure of `vendor/rails/activerecord/lib` + `vendor/rails/activemodel/lib`: **518 missing members in 64 files are inside the closure** and are the AR-necessary work; the other 699 are out of scope (excluded via `0072/activesupport-out-of-closure-unported-entries`; non-portable in-closure members triaged into SKIP_GROUPS via `0072/activesupport-closure-skip-groups-triage`).

This RFC owns porting the in-closure remainder. The dominant cluster is the `DateAndTime::Calculations` mixin surface, counted once per receiver ≈ 249 members (~48% of the 518): the mixin file itself (63) plus its Time copy (attributed by the compare to `core_ext/object/blank.rb`, 67), Date copy (attributed to `core_ext/date/acts_like.rb`, 66), and the Time/DateTime-specific calc remainders (29 + 24). One mixin implementation wired onto Date/Time/DateTime credits all of them. Much of the underlying logic already exists in `@blazetrails/date` — reuse it; the parity credit requires the members at the activesupport paths.

## Approach

Nine slots, each a standalone ~220–280 LOC PR from main (no stacking). Rails sources under `vendor/rails/activesupport/lib/active_support/`. Every port follows CLAUDE.md fidelity rules: Rails names, control flow, decomposition; deviations only for genuine TS shortcomings, justified at the call site.

Direct AR/AM call-site evidence (grep of vendor AR/AM lib): `blank?/present?` 64+20 sites, `extract_options!` 20, `Array#from/to` 15, `deep_dup` 10, `try` 10, `stringify_keys` 10, `symbolize_keys` 9, `change` 11, `acts_like?` 7, `second_to_last` 4, `ago` 4, `in_time_zone` 2, `megabytes` 1. Test-axis: `assert_called` used by 30 AR test files, `travel_to` by 6.

## Stories

- date-and-time-calculations-predicates-and-day-arithmetic (Slot A)
- date-and-time-calculations-week-month-quarter-year (Slot B)
- time-and-date-time-specific-calculations (Slot C)
- core-ext-sweep-array-and-numeric (Slot D)
- core-ext-sweep-hash-module-string (Slot E)
- time-with-zone-and-duration-residue (Slot F)
- deprecation-and-logging-internals (Slot G)
- testing-helpers-for-ar-test-parity (Slot H)

(Slot I of the audit — SKIP_GROUPS triage — already filed under RFC 0072.)

## Changelog

### 2026-08-21 — the four in-closure XML conversions (option 1)

`resolve-the-in-closure-xml-conversions` asked which of three dispositions
resolves the contradiction between this RFC's measurement and RFC 0101's
"XmlMini is out-of-closure" framing. **Option 1 — port the minimum XmlMini
slice, inside 0098 — was chosen** (owner decision, backlog triage, recorded on
the story). **The require path, read off `ar-closure.ts`'s walk** (`output/ar-closure.json`,
`files.activesupport`), and it does not say quite what the story assumed:

- `core_ext/array/conversions.rb` **is** in the closure, reached in one hop from
  `activemodel/lib/active_model/errors.rb:3`,
  `require "active_support/core_ext/array/conversions"`. So `Array#to_xml` is
  genuinely in-closure.
- `core_ext/hash/conversions.rb` is **not** in the closure. Nothing under
  `activerecord/lib` or `activemodel/lib` requires it, directly or transitively
  — the only requirers anywhere are `core_ext/hash.rb` and
  `core_ext/object/conversions.rb`, neither of which AR/AM pulls. The in-closure
  `core_ext/hash/*` files are just `deep_merge`, `except`, `indifferent_access`,
  `keys`, `reverse_merge`, `slice`.
- `xml_mini.rb` itself is **not** in the closure either, consistent with RFC
  0101's framing for the rest of it.

The three `Hash` members nevertheless counted against the AR-closure gap because
`parity:api` maps every Hash core_ext member onto `hash-utils.ts`, whose Ruby
anchor is the in-closure `core_ext/array/extract_options.rb`. That is a
file-attribution artifact of the compare, not a closure bug — which is exactly
why **option 3 was not taken**: `ar-closure.ts` is not wrong, so editing it
would have moved the denominator with no require-graph justification, the one
thing the story forbids.

Landed in trails#6818. The four members are `Hash#to_xml`, `Hash.from_xml`,
`Hash.from_trusted_xml` (→ `hash-utils.ts`) and `Array#to_xml` (→
`array-utils.ts`). The XmlMini surface pulled in with them, and the boundary
this RFC now owns, is exactly:

- `XmlBuilder.instruct()` / `.target()` — the `Builder::XmlMarkup#instruct!` /
  `#target!` roles `Hash#to_xml` and `Array#to_xml` call directly
  (`core_ext/hash/conversions.rb:83`, `core_ext/array/conversions.rb:199`).
- An indent width on `IndentedXmlStringBuilder`, so `indent: 0` reproduces
  Builder's unformatted output.
- `ActiveSupport::XMLConverter` and its `DisallowedType` / `DISALLOWED_TYPES`
  (`core_ext/hash/conversions.rb:140-262`), which is `Hash.from_xml`'s whole
  body.

Nothing else in `xml_mini.rb` was touched; the backend/parsing half stays RFC
0101's. AR closure moved 8917/8943 → 8940/8948.

### `acts_like?` markers move to `@blazetrails/date` (arm D of `time-with-zone-residue-structural-blockers`)

Rails hangs `acts_like_date?` / `acts_like_time?` by reopening `Date`, `Time`
and `DateTime` (`core_ext/date/acts_like.rb:5-9`,
`core_ext/time/acts_like.rb:5-9`, `core_ext/date_time/acts_like.rb:6-13`).
trails cannot reopen its equivalents — `Date.parse` answers a
`Temporal.PlainDate` and `DateTime.parse` a
`Temporal.PlainDateTime | Temporal.ZonedDateTime`, values of a third-party
package — which is why the marker #6465 put on a class at the Rails path was
inert: nothing constructs one.

Of the three options the blocker priced, the owner chose **(2) move the markers
into `@blazetrails/date`**, where the values are constructed. They live in
`packages/date/src/acts-like.ts` as `actsLikeDate` / `actsLikeTime` predicates
over this package's own value types, and `Object.actsLike`
(`core-ext/object/acts-like.ts`) calls them instead of carrying its own
`isRubyTime` / `isRubyDate` copies. Option (1) — installing the markers on the
`Temporal` polyfill prototypes at import time — was rejected as a global side
effect on a third-party package that also overstates the mapping; option (3), a
`SCOPED_SKIP_GROUPS` entry, was not seeded.

The cost is explicit: these two members are credited at a path Rails does not
have, so **RFC 0098's reachable ceiling drops by 1 member** (`Time#acts_like_time?`).

## Done means

Every in-closure activesupport file reports 0 missing members in `pnpm parity:api` (or its non-portable members carry SKIP_GROUPS reasons), and the "AR closure" rollup (`0092/ar-closure-rollup-in-parity-summaries`) reads 100%.
