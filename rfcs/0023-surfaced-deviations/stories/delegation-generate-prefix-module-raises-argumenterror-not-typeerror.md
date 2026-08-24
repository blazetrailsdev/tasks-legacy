---
title: "Delegation.generate raises ArgumentError where Ruby raises TypeError for prefix: true with a Module target"
status: draft
updated: 2026-08-24
rfc: "0023-surfaced-deviations"
cluster: null
packages: []
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

# `Delegation.generate` raises ArgumentError where Ruby raises TypeError for `prefix: true` with a Module target

## Context

Shipped in PR #6995. `vendor/rails/activesupport/lib/active_support/delegation.rb:26-29`:

    if prefix == true && /^[^a-z_]/.match?(to)
      raise ArgumentError, "Can only automatically set the delegation prefix when delegating to a method."
    end

`to` may be a Module (`:36`). `Regexp#match?` has no implicit String conversion
for a Module, so MRI raises `TypeError: no implicit conversion of Class into
String` from that line before the `ArgumentError` can be reached. trails cannot
reproduce that incidentally — a TS regex test has to be guarded by an explicit
`typeof to !== "string"` check — so
`packages/activesupport/src/delegation.ts` currently raises the `ArgumentError`
the line is guarding for:

    if (prefix === true && (typeof to !== "string" || /^[^a-z_]/.test(to)))

The deviation is documented in `generate`'s JSDoc, and it is the error CLASS
that differs, not the guard's placement or its effect.

## Converged shape

Raise a `TypeError` with MRI's message for the non-string arm, keeping the
`ArgumentError` for a string `to` that fails the pattern, so both raise sites
match `delegation.rb:27` exactly. Confirm MRI's message text with `ruby -e` —
`ruby` is on PATH — rather than deriving it.

## Acceptance criteria

- [ ] `prefix: true` with a Module target raises `TypeError` with MRI's message.
- [ ] `prefix: true` with a non-method-shaped String still raises the
      `ArgumentError` at `delegation.rb:27`.
- [ ] The deviation note is removed from `Delegation.generate`'s JSDoc, not
      reworded.
