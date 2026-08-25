---
title: "Converge Notifications.subscribe onto arity-preserving callback forwarding"
status: ready
updated: 2026-07-27
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 40
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`Notifications.subscribe` (`packages/activesupport/src/notifications.ts`) still
wraps its callback in `(event: Event) => callback(event)`, forcing the `Fanout`
`createSubscriber` arity check (`notifications/fanout.ts`) to classify every
subscriber as `event_object`. Rails' `subscribe` forwards the block with its
arity intact — `notifier.subscribe(pattern, callback, monotonic: false)`
(`vendor/rails/activesupport/lib/active_support/notifications.rb:245`) — so a
five-arity subscriber receives `(name, start, finish, id, payload)`.

PR #4946 fixed `monotonicSubscribe` and `subscribed` to forward the callback
unwrapped (via the exported `NotificationCallback` union), but deliberately
left `subscribe` wrapped — that Event-only wrapping is the established #4913
deviation with many callers. The result is an inconsistency: `subscribe` is
Event-only while its two siblings support the timed 5-arg form.

Consequence in tests: the timed `subscribe` case in
`TimedAndMonotonicTimedSubscriberTest` (`notifications.test.ts`) uses a 1-arg
callback + `duration >= 0` check instead of Rails' 5-arg `[Time, Time]`
assertion (`vendor/rails/activesupport/test/notifications_test.rb` —
`test_timed_subscribe`, ~113-125).

## Acceptance criteria

- [ ] `Notifications.subscribe` forwards the callback to the notifier with its
      arity intact (reuse `NotificationCallback`), so a 5-arg subscriber is
      classified timed and receives `(name, start, finish, id, payload)`.
- [ ] Audit callers of `Notifications.subscribe` that rely on always receiving
      an `Event`; keep them working (1-arg callers are unaffected).
- [ ] The timed `subscribe` test asserts the Rails shape (start/finish are
      `Temporal.Instant`/wall-clock), mirroring `test_timed_subscribe`.
- [ ] parity:api and parity:test deltas non-negative.
