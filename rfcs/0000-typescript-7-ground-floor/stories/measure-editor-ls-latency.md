---
title: "Measure editor language-service latency on activerecord (TS 5.9.3 vs TS 7)"
status: draft
updated: 2026-08-25
rfc: "0000-typescript-7-ground-floor"
cluster: developer-experience
packages: ["activerecord"]
deps: []
deps-rfc: []
est-loc: 0
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
---

## Context

**Deliberately postponed (2026-08-25).** The TSE language-service plugin is not
wired into any `tsconfig` `plugins` array or editor config in this repo, so
there is no editor integration to regress or improve yet. This story is parked
with the views pipeline and is not part of the ground-floor work. The rest of
this body is retained so the measurement is cheap to pick up when it matters.

## Original context

RFC `0000-typescript-7-ground-floor` measured every cost axis of staying on
TypeScript 5.9.3 except one, and labelled it **unmeasured** rather than guessing:
editor latency. It is also the axis with the largest plausible upside, because
`packages/activerecord/src` is 165,308 non-test LOC — 4× the next package.

The RFC's other findings sharply narrow what a migration could buy elsewhere:
PR wall-clock saving is **zero** (typecheck is off the CI critical path;
`Build & Type Check` medians 1.4m against a 12.9m run), CI runner saving is
~8.9 hours/week, and the local incremental path is already 8–11s. If editor
latency turns out to be the real cost, that changes the weight of the whole
decision. If it does not, waiting is unambiguously right.

RFC #59 listed "editor open-to-ready time in VS Code with the TS 7 language
service enabled" as Phase 0 evidence and never collected it. This story exists
so the next re-evaluation does not have a hole in the same place.

## Acceptance criteria

- [ ] Open-to-ready latency for `packages/activerecord` is measured under the
      TypeScript 5.9.3 language service and under the TS 7 language service, on
      the same machine, with the host load average recorded for each run.
- [ ] At least three repeated trials per compiler; median and spread reported,
      not a single sample.
- [ ] The measurement method is written down precisely enough to re-run
      (which editor, which build, which file opened, what "ready" means —
      e.g. first hover resolving, or the TS server's own `projectLoadingFinish`
      event).
- [ ] The result is folded into the RFC's Motivation table, replacing the
      "unmeasured" row.
- [ ] If the measurement cannot be made credibly, that is recorded as such with
      the reason — an honest "still unmeasured, because X" closes this story.

## Definition of done

A number quoted from Microsoft's benchmarks or from another project does not
close this story. The measurement is about **this** codebase.

## Notes

The TS server logs its own timings (`TSS_LOG` / "TypeScript: Open TS Server
Log"), including `projectLoadingStart` / `projectLoadingFinish`, which is likely
the most defensible "ready" signal. `tsserver` also exposes `--enableTelemetry`
timings.

Independent of the TS 7 decision: the same measurement is the baseline for any
future editor-performance work.
