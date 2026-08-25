---
title: "ActiveModel::Name#initialize raises three ArgumentErrors Rails does not"
status: draft
updated: 2026-08-15
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
deps: []
deps-rfc: []
est-loc: 90
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# `ActiveModel::Name#initialize` raises three ArgumentErrors Rails does not

## Context

Surfaced converging the constructor to Rails' four positional arguments in
PR #6568 (`model-name-constructor-takes-klass`).

`activemodel/lib/active_model/naming.rb:166-185` validates exactly one thing:

    raise ArgumentError, "Class name cannot be blank. You need to supply a name argument when anonymous class given" if @name.blank?

`packages/activemodel/src/naming.ts`'s constructor adds three more raise sites
Rails has no counterpart for:

1. an `invalidNamespace()` `ArgumentError` for a namespace that is not a
   string / string[] / `{ name }`;
2. the same error for a blank or whitespace-only namespace segment;
3. a `'ModelName arguments must not contain "::"'` `ArgumentError` for either
   argument, asserted by two trails-only tests in `naming.test.ts`
   ("ModelName rejects Ruby-shaped strings").

Rails accepts a Module for `namespace` and reads `namespace.name`; a malformed
namespace is a `NoMethodError` at the call site, not a validated argument. The
`::` rejection is the sharpest divergence: Ruby's `@name` is ALWAYS a
`::`-qualified constant path, so trails rejects the exact input Rails expects.

## Converged shape

Keep only Rails' blank-name raise. The namespace stays segment-shaped (a JS
class carries no module path — that part is language-forced and already
documented at the constructor), but a malformed one is not a validated error,
and the two trails-only rejection tests go with the guards.

## Acceptance criteria

- [ ] `naming.ts`'s constructor has exactly one raise site, with Rails' message.
- [ ] The trails-only "ModelName rejects Ruby-shaped strings" tests are deleted
      rather than rewritten.
- [ ] `pnpm parity:test` delta non-negative.
