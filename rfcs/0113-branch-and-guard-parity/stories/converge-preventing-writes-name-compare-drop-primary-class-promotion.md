---
title: 'core.ts isPreventingWrites promotes primary classes to "Base"; Rails compares klass.name plainly'
status: ready
updated: 2026-08-25
rfc: "0113-branch-and-guard-parity"
cluster: invented-arm
packages: []
deps: []
deps-rfc: []
est-loc: 60
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Carried over from PR #6188, which closed
`delegate-preventing-writes-to-connection-descriptor` (RFC 0023). That story's
fourth acceptance criterion was:

> Assess the invented `primaryClassQ() ? "Base" : k.name` normalization against
> Rails' plain name compare (`core.rb:213`). If any suite depends on it, capture
> which and why at the call site; if none does, drop it so `core.ts`'s walk is
> the single faithful implementation.

PR #6188 did the first half — the duplicated stack walk in
`abstract-adapter.ts` is gone and there is now a single implementation — but it
**moved** the normalization into that survivor rather than assessing it. It now
lives in `isPreventingWrites` at
`packages/activerecord/src/core.ts:619-631`:

```ts
const targetName =
  typeof klass.primaryClassQ === "function" && klass.primaryClassQ() ? "Base" : klass.name;
if (targetName === className) return entry.preventWrites;
```

Rails is a plain compare with no promotion
(`vendor/rails/activerecord/lib/active_record/core.rb:211-213`):

```ruby
def self.preventing_writes?(class_name) # :nodoc:
  connected_to_stack.reverse_each do |hash|
    return hash[:prevent_writes] if !hash[:prevent_writes].nil? && hash[:klasses].include?(Base)
    return hash[:prevent_writes] if !hash[:prevent_writes].nil? && hash[:klasses].any? { |klass| klass.name == class_name }
  end
  false
end
```

Rails does not need the promotion because
`ConnectionHandler::ConnectionDescriptor#name`
(`vendor/rails/activerecord/lib/active_record/connection_adapters/abstract/connection_handler.rb:62-64`)
already answers `"ActiveRecord::Base"` for a primary class, so a primary
descriptor's name matches `Base.name` directly. trails' `PoolConfig`
normalizes that name to `"Base"` instead, and the promotion is the read-side
compensation for the write-side divergence. See
[[converge-connection-descriptor-name-to-rails-primary-class-form]], which is
the other half of the same divergence — converging that one should make this
promotion deletable outright.

## Converged shape

`isPreventingWrites` compares `klass.name === className` with no
`primaryClassQ` branch, matching `core.rb:213` exactly.

Method: delete the promotion, run the prevent-writes suites named in the parent
story (`adapter-prevent-writes`, `base-prevent-writes`,
`abstract-adapter-preventing-writes.trails`, `connection-handling`,
`connection-swapping-nested`, `database-selector`, `shard-keys`, plus the three
adapter-specific ones) on sqlite, PG and MariaDB. If a suite genuinely depends
on it, that dependency is the real bug — capture which one and why, and fix the
descriptor-name side rather than restoring the promotion.

## Acceptance criteria

- [ ] `core.ts`'s `isPreventingWrites` is Rails' plain name compare, or the one
      suite that requires otherwise is named with its root cause.
- [ ] Prevent-writes suites green on sqlite, PostgreSQL and MySQL/MariaDB.
- [ ] No test names change.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0113-branch-and-guard-parity`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
