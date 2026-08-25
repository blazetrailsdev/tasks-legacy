---
title: "ColumnSerializer#check_arity_of_constructor raises TypeError where Rails raises ArgumentError"
status: draft
updated: 2026-08-22
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
  - "activerecord"
  - "activesupport"
  - "globalid"
  - "i18n"
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

`Coders::ColumnSerializer#check_arity_of_constructor` raises **`ArgumentError`**:

```ruby
def check_arity_of_constructor
  load(nil)
rescue ArgumentError
  raise ArgumentError, "Cannot serialize #{object_class}. Classes passed to `serialize` must have a 0 argument constructor."
end
```

(`activerecord/lib/active_record/coders/column_serializer.rb:53-56`).

trails throws a **`TypeError`** with the same message
(`packages/activerecord/src/coders/column-serializer.ts:92-102`, after PR #6845).
Same message string, same raise site, wrong error class — CLAUDE.md's fidelity
bar is "same error class, same message string, same raise site".

PR #6845 converged the _call_ (the body now probes through `load(null)` as Rails
does) but deliberately left the error class alone: it is observable by callers
and by `column-serializer.test.ts`, so flipping it is its own change.

The blocker is that trails has **no shared `ArgumentError`**. Every call site
declares a file-local one:

```text
packages/activemodel/src/attribute-assignment.ts:263
packages/activesupport/src/environment-inquirer.ts:5
packages/activesupport/src/logger.ts:16
packages/activesupport/src/hash-utils.ts:18   (exported)
packages/activesupport/src/current-attributes.ts:23
packages/activerecord/src/signed-id.ts:15
packages/activesupport/src/error-reporter.ts:8
packages/activesupport/src/security-utils.ts:3
packages/globalid/src/verifier.ts:3
packages/i18n/src/exceptions.ts:107           (exported)
```

so "raise ArgumentError" has no single answer today. Note the sibling stories
`bigdecimal-raises-typeerror-not-argumenterror` and
`converge-date-civil-arity-argumenterror` are the same class of finding — this
one may want to be scheduled with them, or after a decision on where a canonical
`ArgumentError` lives.

## Converged shape

```ts
checkArityOfConstructor(): void {
  try {
    this.load(null);
  } catch (e: unknown) {
    throw new ArgumentError(
      `Cannot serialize ${this._objectClass.name}. Classes passed to \`serialize\` must have a 0 argument constructor.`,
    );
  }
}
```

with `ArgumentError` resolved from wherever the repo settles the canonical one.

## Acceptance criteria

- [ ] `checkArityOfConstructor` raises trails' `ArgumentError`, not `TypeError`,
      preserving the message verbatim from `column_serializer.rb:55`.
- [ ] Whatever `ArgumentError` it uses is a deliberate choice, not an eleventh
      file-local declaration — either the canonical one, or the decision is
      recorded here.
- [ ] `packages/activerecord/src/coders/column-serializer.test.ts` and
      `serialized-attribute.test.ts` updated for the class change; no test names
      renamed.
- [ ] SQLite, PostgreSQL and MySQL/MariaDB lanes green.
