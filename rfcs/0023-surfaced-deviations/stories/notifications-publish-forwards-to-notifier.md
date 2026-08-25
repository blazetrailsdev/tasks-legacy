---
title: "Notifications.publish should forward to notifier.publish"
status: draft
updated: 2026-08-15
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# `Notifications.publish` should forward to the notifier, not build an Event

## Context

`vendor/rails/activesupport/lib/active_support/notifications.rb:200-202`:

    def publish(name, *args)
      notifier.publish(name, *args)
    end

trails' `packages/activesupport/src/notifications.ts` (`static publish`) instead
hand-builds an Event, starts it, swaps in the caller's payload object, finishes
it and calls `notifier.publishEvent(event)`. That is `publish_event`
(notifications.rb:204-206), a different method with a different subscriber
path: Rails' `publish` reaches `Fanout#publish` → `iterate_guarding_exceptions`
over `listeners_for(name)` with the RAW args (fanout.rb:289-291), so a
five-arity subscriber sees exactly what the caller passed, and an event-object
subscriber is not synthesised for it at all.

Surfaced reviewing PR #6561, which moved the Event construction onto
`instrumenter.newEvent` + `start!`/`finish!` but deliberately left the shape
alone as out of scope. `Fanout#publish` already exists in
`packages/activesupport/src/notifications/fanout.ts:444`.

## Converged shape

`Notifications.publish(name, ...args)` is `notifier.publish(name, ...args)`.
`publishEvent` stays as it is. Any trails caller depending on the current
Event-synthesising behaviour moves to `publishEvent` with an explicitly built
event, or to `instrument`.

## Acceptance criteria

- [ ] `publish` forwards its arguments to `notifier.publish` and builds nothing.
- [ ] Callers of `Notifications.publish` in the workspace are audited and moved
      where they relied on the synthesised Event.
- [ ] `notifications.test.ts` / `notifications.trails.test.ts` green; any test
      asserting the synthesised Event is re-pointed at `publishEvent`.
