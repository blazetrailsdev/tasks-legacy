---
rfc: "0023-surfaced-deviations"
title: "Surfaced deviations & follow-ups — standing backlog for port-discovered work"
status: postponed
created: 2026-06-11
updated: 2026-08-09
owner: "@deanmarano"
packages:
  - "actionpack"
  - "actionview"
  - "activemodel"
  - "activerecord"
  - "activerecord-cli"
  - "activesupport"
  - "arel"
  - "date"
  - "did-you-mean"
  - "globalid"
  - "i18n"
  - "nokogiri"
  - "rack"
  - "trailties"
  - "website"
clusters:
  - rails-deviation
  - followup
related-rfcs:
  - "0005-activerecord-gaps"
  - "0015-ar-framework-gaps"
priority: 2
---

# RFC 0023 — Surfaced deviations & follow-ups

## Summary

A **standing catch-all backlog** for work that surfaces _during_ a PR rather than
from up-front design: Rails deviations noticed in passing and sized follow-ups an
agent chose not to fold into the PR it was scoped to. The btwhooks post-merge and
review-exhausted messages route such findings here via `pnpm tasks new` **when no
existing RFC is a better fit**. This RFC is the fallback bucket — best-fit always
wins; this exists so a real finding is never dropped just because it lacks a home.

## Motivation

The trails port runs an automated PR loop (btwhooks → agent panes). Agents
routinely surface two kinds of work mid-flight:

- **Rails deviations** — a spot where trails diverges from the Rails source, noticed
  while touching nearby code but out of scope for the current PR.
- **Sized follow-ups** — a concrete, estimable chunk of work deliberately deferred to
  keep the current PR small.

Historically these were captured only as **prose** in the post-merge-findings report
(`## Deviations from Rails`, `## Followup work (sized)`). That prose was a dead end —
nothing turned it into backlog, so the findings evaporated. The fix is to file each
substantive item as a real story via `pnpm tasks new`. Most land under a topical RFC
(`0005-activerecord-gaps`, `0015-ar-framework-gaps`, …); this RFC catches the rest.

## Design

A normal RFC dir whose stories are authored **ad-hoc by automation**, not enumerated
up front. Once routing is wired up (see §Open questions — the messages.json swap is a
**separate change** and is _not_ live at merge), the btwhooks `trails.pr_merged` and
`trails.cycles_exhausted` messages will instruct the agent to:

1. Identify each substantive, actionable finding (a real deviation or a sized
   follow-up — not a passing thought).
2. Pick the **best-fit active RFC**; only if none fits, file here.
3. `pnpm tasks new <rfc> <slug> --title "..." --est-loc <N>` — dedupe first via
   `pnpm tasks list --rfc <slug>`.

Filed stories land as `status: draft` (the CLI default), which is the correct triage
posture: they are **surfaced, not yet accepted**. A human (or a triage pass) promotes
`draft → ready`, re-homes the story under a topical RFC, or drops it. This RFC is
intentionally a **holding area**, not a committed work plan — it has no rollout
ordering of its own.

### Intake rule — a story must name what it converges toward

A story filed here **must state the Rails `file:line` its acceptance criteria move
trails toward, and the trails `file:line` that diverges from it.** Both sides,
concretely. If you cannot write those two references, you do not yet have a story —
you have an observation, and it belongs in the PR body or a code comment, not the
backlog.

Concretely, a story is in-scope for this RFC only if:

1. **It names a behavioural divergence from Rails.** Emitted SQL, raised error class
   or message, call order, return value, query count, callback firing — something a
   test could pin. A TS-declaration narrowing that Rails-literal call sites have to
   cast around is _not_ in scope here (that is declaration-shape work; file it against
   the RFC that owns the surface).
2. **Its acceptance criteria are convergence, not a decision.** "Decide whether X is
   acceptable", "audit and record", "establish whether" — these are not stories. They
   accreted here for months and were closed unimplemented in the 2026-08-09 sweep, at
   the cost of re-deriving their evidence. If the deviation is real, the acceptance
   criterion is "trails does what `<rails file:line>` does". If it is a genuine TS
   language wall, the resolution is a justification at the call site in the same PR
   that discovers it — not a story.
3. **It is reachable.** A divergence no code path and no Rails test can reach is a
   comment at the call site, not backlog.

Two named non-goals, both of which produced closes rather than convergence:

- **Infrastructure and tooling** — docker-compose services, lint memory, CI ergonomics,
  the tasks CLI itself. Real work, wrong RFC; this bucket is for port fidelity.
- **Ruby interpreter semantics with no TS analogue** — `NoMethodError`, constructor
  arity enforcement, open-class dispatch, byte-`String`. These belong to
  `0000-corelib-primitives` (postponed) and are closed on sight here.

### Sweep cadence

The bucket is swept periodically against `origin/main`, closing stories that are
already done, unreachable, or not convergence work. **The sweep verifies every story
against today's `main` rather than trusting its prose** — story bodies go stale within
weeks, and a premise like "~123 call sites" routinely turns out to be two. A story
that survives a sweep with a live, cited divergence should be re-homed onto the
topical RFC that owns the surface; 0023 is the waiting room, not the ward.

### Clusters

- `rails-deviation` — a divergence from the Rails source worth tracking.
- `followup` — deferred, in-scope-adjacent work sized for a later PR.

Filing without `--cluster` (cluster `null`) is fine; the cluster is a convenience for
later triage, not a requirement.

## Alternatives considered

- **Reuse `0005-activerecord-gaps` as the fallback:** rejected — `0005` is a scoped,
  AR-specific gaps RFC with its own rollout; dumping unrelated cross-cutting findings
  into it pollutes that plan. A dedicated, plan-free bucket keeps topical RFCs clean.
- **Skip the RFC, keep prose findings:** rejected — that is the status quo the port
  loop is trying to fix; prose findings never re-enter the backlog.

## Rollout

None. This RFC has no phased plan — stories arrive continuously from the PR loop and
are triaged out of here individually. It stays `active` indefinitely. Stories are
authored ad-hoc and none are pre-enumerated; see the live index
(`pnpm tasks list --rfc <this-rfc>`) for current contents.

## Open questions

1. **Activation — the messages.json swap.** This RFC is inert until the btwhooks
   `trails.pr_merged` / `trails.cycles_exhausted` messages point their fallback at this
   RFC's finalized number. That swap is a **separate change** (the messages currently
   fall back to `0005-activerecord-gaps`) and is a **prerequisite** for any story to
   actually land here. Until it lands, this bucket exists but receives nothing.
2. **Periodic triage cadence.** Surfaced stories accumulate as `draft`. A recurring
   sweep should promote, re-home, or drop them so the bucket does not grow unbounded.
   Recommendation: fold into the existing backlog-grooming pass; revisit if volume
   warrants a dedicated routine.

## Changelog

- 2026-06-11: initial RFC — standing fallback bucket for port-surfaced deviations and
  follow-ups. The btwhooks messages.json change that routes findings here via
  `pnpm tasks new` is a separate, not-yet-landed change (see §Open questions).
- 2026-08-09: added the intake rule (a story must cite both the Rails and trails
  `file:line`; decision-only, infrastructure, and corelib-semantics items are out of
  scope) and the sweep cadence, after the 8-shard triage sweep closed 24 of 70 stories
  in one shard alone — most of them decision-only or non-fidelity work.
