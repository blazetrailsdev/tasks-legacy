---
title: "time-with-zone-residue-structural-blockers"
status: in-progress
updated: 2026-08-23
rfc: "0098-activesupport-ar-closure-port"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: null
priority: null
pr: 6924
claim: "2026-08-23T16:12:28Z"
assignee: "time-with-zone-residue-structural-blockers"
blocked-by: null
closed-reason: null
---

## Re-scope (2026-08-18)

This story carried three independent arms. Two have been split out and are no
longer blocked; **this story is now arm (D) only** — where the `acts_like`
markers live.

| arm                                                                          | disposition                                                                                                                                                                                                                                                                                                                                  |
| ---------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| (A) `TimeWithZone` four-arg ctor + `TimeZone` period lookups                 | **Split out.** Its blocking premise was stale: `TimezonePeriod` (`values/time-zone.ts:543`) and `periodForUtc` (`:1076`) already landed, so "does TimeZone grow a Period type" is answered — it did. Now `port-time-zone-local-period-lookups` (ready) + `widen-time-with-zone-ctor-onto-rails-four-argument-shape` (deps on it). 9 members. |
| (B) `to_formatted_s` / `readable_inspect` / `default_inspect` collision      | **Split out** as `split-time-ext-by-receiver-onto-the-rails-layout` (ready). Not a judgement call: `parity:api` matches on the Rails layout and `time-ext.ts` hosts four Rails files across three receivers. 2 members.                                                                                                                      |
| (C) `preserve_timezone` / `system_local_time?` / `active_support_local_zone` | Already landed in #6547.                                                                                                                                                                                                                                                                                                                     |
| (D) `acts_like` markers                                                      | **Stays here, still blocked.** 1 member (`acts_like_time?`).                                                                                                                                                                                                                                                                                 |

## The remaining question

`Date#acts_like_date?` / `Time#acts_like_time?` are markers Rails defines by
reopening `Date` and `Time`. trails cannot reopen its equivalents: `DateTime.parse`
returns `Temporal.PlainDateTime | Temporal.ZonedDateTime`, never an instance of
a class activesupport owns. PR #6465 put the marker on a class at the Rails
path, which was inert — nothing the parser returns is an instance of it.

Two admissible resolutions, neither obviously right:

1. **Install the markers on the Temporal polyfill prototypes at import time.**
   Keeps the Rails file path, so `parity:api` credits the member where Rails
   defines it. Costs a global side effect on a third-party package, and
   overstates the mapping — a `PlainDateTime` is not only ever a `DateTime`.
2. **Move the markers into `@blazetrails/date`,** where the values are
   constructed. Honest about ownership and side-effect-free, but gives up the
   Rails file path, so the member is credited at a path Rails does not have —
   or not credited at all.

A third option worth pricing before choosing either: **decide the marker has no
faithful trails counterpart** and record it in `SCOPED_SKIP_GROUPS` with that
reason. That is only admissible if `acts_like?` genuinely cannot be answered —
not merely because both ports are awkward — and it lowers 0098's reachable
ceiling by 1 member, which must be stated in the RFC rather than absorbed.

## Acceptance criteria

- [ ] One of the three resolutions above is chosen and recorded in RFC 0098,
      with the reason at the call site if code lands.
- [ ] `Time#acts_like_time?` is either credited by `pnpm parity:api` or carries
      a reasoned `SCOPED_SKIP_GROUPS` entry naming this decision.
- [ ] Whatever ships is exercised through a value the parser actually returns —
      not through a class instance nothing constructs, which is how #6465 came
      to be inert.
- [ ] If option 1: the import-time side effect is confined to one module and
      documented as a deliberate global, and the `PlainDateTime`-is-not-a-
      `DateTime` overstatement is stated where the marker is installed.
- [ ] `pnpm parity:api:extra` clean; no new baseline rows.

## Decision (2026-08-21, backlog triage)

**Arm (D): move the `acts_like?` markers into `@blazetrails/date`.** Owner
decision among the three options the blocker names. Honest ownership — the
markers describe `@blazetrails/date`'s own types — and it explicitly gives up
the Rails file path for that member, so 0098's reachable ceiling drops by 1;
state that in the RFC changelog. Do NOT install markers on the Temporal
polyfill prototypes at import time (global side effect on a third-party
package), and do not seed a SCOPED_SKIP_GROUPS entry instead.
