---
rfc: "0118-rfc-story-cap-and-succession"
title: "Open-story pressure signal and a supersede that carries stories"
status: draft
created: 2026-08-23
updated: 2026-08-23
owner: "@deanmarano"
packages: []
clusters:
  - open-story-signal
  - carrying-supersede
---

# RFC 0118 — Open-story pressure signal and a supersede that carries stories

## Summary

Large RFCs accumulate stale context as they run, and someone is already paying
that tax by hand: 113 stories carry an ad-hoc `## Re-verified <date>` section.
This RFC adds a **warning** in `tasks new` when an RFC exceeds 40 open
(non-`done`, non-`closed`) stories, and gives `tasks rfc --supersede` a
`--carry` mode that actually moves open stories to the successor — which it
does not do today. Together they make RFC succession a real, cheap operation so
campaigns close and re-open with a fresh, current scope rather than growing
without bound.

## Motivation

Measured against `rfcs/` on 2026-08-23: 116 RFCs, 6839 stories, 1314 open.

**A cap on total stories would fire on the wrong RFCs.** Total and open counts
are decorrelated, because a healthy burndown RFC accumulates `done` stories
without accumulating work:

| RFC                                         | total | open |
| ------------------------------------------- | ----- | ---- |
| `0106-wide-call-set-direct-burndown`        | 213   | 7    |
| `0051-migration-schema-statements-fidelity` | 341   | 37   |
| `0098-activesupport-ar-closure-port`        | 138   | 9    |
| `0096-naming-identifier-burndown`           | 121   | 7    |
| `0112-one-rails-thing-n-trails-things`      | 131   | 78   |
| `0061-ci-failures`                          | 133   | 1    |

Across the 10 `active` RFCs, open-story count has median 11, p75 37, p90 78,
max 78. A 25-story cap on _totals_ would have rolled `0106` over eight times
while it was successfully burning down to 7 open.

**The staleness tax concentrates in parked RFCs, not large ones.** Of the 113
`## Re-verified` stories, **82 (73%) are in `0025-fidelity-verification-tooling`**
— status `postponed`, 272 stories. `0023-surfaced-deviations`, the largest RFC
in the repo at 1504 stories, has **zero**. The correlate is being parked, not
being large.

**`--supersede` does not carry stories.** `scripts/cli.ts:2247-2258` writes
`status: superseded` and `superseded-by` into the RFC README and touches
nothing else. 219 stories currently sit under superseded RFCs. Chaining today
produces an empty successor and strands the predecessor's open work.

## Design

### 1. Open-story pressure signal in `tasks new`

`newStory` counts stories under the target RFC whose `status` is neither `done`
nor `closed`. When the new story would push that count above **40**, it prints
a warning **after** creating the story, naming the succession command. It never
refuses.

Warning, not refusal, is load-bearing. `tasks new` is the mechanism CLAUDE.md
directs every agent to for out-of-scope discoveries ("do NOT open it yourself —
add a new story"). A hard refusal at that moment converts "file the follow-up"
into "drop it on the floor", which is the exact debt the rule exists to
prevent.

40 sits just above the active p75 of 37 and well under p90 of 78. Today it
catches `0112` (78 open) and `0105` (74 open) and nothing else.

### 2. `tasks rfc <retired> --supersede <successor> --carry`

Note the direction of the existing verb: `rfc <slug> --supersede <other>` marks
**`<slug>`** as `superseded` and writes `superseded-by: <other>`. `--carry`
follows that direction — every open story under `<retired>` moves to
`rfcs/<successor>/stories/`, with its `rfc:` frontmatter field rewritten (the
field must match the parent dir, or `validate.mjs` rejects the commit).
Terminal stories (`done`, `closed`) stay under the retired RFC as its
historical record. A slug collision with an existing story in the successor is
a hard error, not a silent overwrite.

**Story slugs are preserved verbatim.** `set-deps` addresses stories by bare
slug, so a rename during migration would break every cross-RFC dep pointing at
a carried story.

### 3. No intake exemption

`0023` and `0061` need no special case under an open-count rule: `0061` has 1
open story and `0023` is `postponed`. An explicit "intake" RFC kind would
create a label that everything migrates to; the open-count metric exempts
intake RFCs for free, by measuring the thing that actually matters.

## Non-goals

- **Forced re-justification at the rollover boundary.** The one real sample is
  `0025`'s sweep: of 82 stories re-verified, 63 came back `ready`, 18 `draft`,
  and exactly **1** was closed. Re-verification confirms stories rather than
  killing them; 82 re-checks to retire 1 story is not a trade worth scheduling.
  The right trigger for re-verification is **unparking** a postponed RFC, not
  every succession. Filed separately if wanted.
- **A cap on total story count.** See Motivation — it is decorrelated from
  workload and punishes RFCs that are closing stories fast.
- **Hard enforcement / a validate-lib failure.** Deliberately warning-only for
  the first cut; revisit once we have data on whether the warning is heeded.
- **Auto-creating the successor RFC.** `new-rfc` auto-commits and auto-pushes
  to main; triggering that from a warning path is too much action for a signal.

## Alternatives considered

- **Cap on total stories.** Rejected: decorrelated from open workload.
- **Hard refusal in `tasks new`.** Rejected: blocks an agent mid-task and
  incentivizes working around the sanctioned follow-up mechanism.
- **Age-based staleness cap.** Rejected as a primary signal: stories carry no
  `created:` field, and `updated:` — the only proxy — is refreshed by the very
  sweeps it would be measuring. Measured on `updated:`, open stories run median
  10 days, p90 27, max 80, with only 9 stories over 60 days and all nine under
  `draft` RFCs.
- **An explicit `intake` RFC kind.** Rejected: unnecessary under an open-count
  metric, and it creates a gameable exemption label.
- **Carry only `ready` stories.** Considered; rejected in favour of carrying
  all open stories, so nothing schedulable or in-flight is stranded behind a
  superseded parent that `validate-lib.mjs` downgrades out of the ready queue.

## Rollout

1. Phase 1 — open-story count helper and the `tasks new` warning.
2. Phase 2 — `--carry` story migration on `tasks rfc --supersede`.

## Verification

- No `active` RFC exceeds 40 open stories without either dropping below it or
  being superseded, measured 60 days after the warning ships.
- A `--carry` supersede leaves **zero** open stories under the predecessor,
  asserted by a CLI test.
- New ad-hoc `## Re-verified` sections added after this ships trend to zero
  outside of RFC unparking.

## Open questions

1. **Should the threshold be configurable per RFC?** A campaign may legitimately
   want a wider scope. Recommendation: no, not until the fixed 40 demonstrably
   misfires — a per-RFC override is the first thing that would be set to
   infinity.

## Changelog

- 2026-08-23: initial RFC
