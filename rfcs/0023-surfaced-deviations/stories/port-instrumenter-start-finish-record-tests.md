---
title: "port-instrumenter-start-finish-record-tests"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
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

Six tests in `packages/activesupport/src/notifications/instrumenter.test.ts`
(`start` :48, `finish` :55, `record` :62, `record yields the payload for further
modification` :69, `record works without a block` :76, `record with exception`
:83) carry Rails' `InstrumenterTest` names but drive
`Notifications.instrument` + `subscribe` — the static API — instead of the
methods the Rails tests actually exercise:

- `test_start` / `test_finish`
  (`vendor/rails/activesupport/test/notifications/instrumenter_test.rb:53-63`)
  call `instrumenter.start` / `instrumenter.finish` and assert
  `notifier.starts` / `notifier.finishes`.
- `test_record*` (`:65-104`) call `instrumenter.new_event(...)` and then
  `event.record { ... }`.

trails' `Instrumenter`
(`packages/activesupport/src/notifications/instrumenter.ts`) implements none of
those: it has only `instrument` / `instrumentAsync` (plus private `_push` /
`_pop`) — no `start`, no `finish`, no `new_event` — and `Event` has no `record`
(`instrumenter.rb:132-143`).

Because `test-compare` matches on name, these six match anyway and report
coverage for methods that do not exist. Surfaced in review of #4894, which
converged the sibling three `instrument` tests in the same file to real ports;
these six are the same mis-port class, left out of that PR for LOC.

Note that #4894 added a faithful instrument-with-exception test to
`notifications/instrumenter.trails.test.ts`; the stale `record with exception`
at `:83` is a separate mis-port and is not the same test.

## Acceptance criteria

- [ ] Port `Instrumenter#start` / `#finish` (`instrumenter.rb:87-95`) and
      `#new_event` (`:83`), or decide with justification that trails' notifier
      surface has no analogue and the tests should go.
- [ ] Port `Event#record` (`instrumenter.rb:132-143`), including its own
      `rescue Exception` arm setting `payload[:exception]` /
      `[:exception_object]` — the same arm #4894 ported to `#instrument`.
      Depends on `converge-instrumenter-event-payload-dup` for the `.dup`
      semantics `record` relies on.
- [ ] Rewrite the six tests to drive the ported methods against a TestNotifier,
      as `instrumenter_test.rb:10-27` does, rather than the static API.
- [ ] **Expect a parity:test dip.** These six currently match by name; if any
      are deleted rather than ported, matched drops by up to 6 (14448 →
      14442 at time of writing). That is a correction of false positives, not a
      regression — call it out in the PR so the non-negative-delta gate is
      assessed with that context.
