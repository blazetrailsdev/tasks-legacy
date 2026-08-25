---
title: "automatic_inverse_of validates candidates inside the NameError rescue Rails puts it outside"
status: draft
updated: 2026-08-18
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

# automatic_inverse_of validates candidates inside the NameError rescue Rails puts it outside

## Context

Diagnosed while fixing the CI red on PR #6723
(`through-reflection-source-name-swallows-nameerror`), which removed the blanket
`catch` from `ThroughReflection#sourceReflectionName` and made a genuinely
missing model class surface as `NameError`.

Rails scopes its `begin/rescue` to the association _lookups_ only, and calls
`valid_inverse_reflection?` after the block has closed:

`activerecord/lib/active_record/reflection.rb:762-777`

```ruby
begin
  reflection = klass._reflect_on_association(inverse_name)
  if !reflection && active_record.automatically_invert_plural_associations
    plural_inverse_name = ActiveSupport::Inflector.pluralize(inverse_name)
    reflection = klass._reflect_on_association(plural_inverse_name)
  end
rescue NameError => error
  raise unless error.name.to_s == class_name
  reflection = false
end

if valid_inverse_reflection?(reflection)
```

trails hoists the validation _into_ the `try`, so that
`valid_inverse_reflection?` runs under a rescue Rails never applies to it
(`packages/activerecord/src/reflection.ts`, `automaticInverseOf`):

```ts
for (const n of lookupNames) {
  const r = this.klass._reflectOnAssociation(n);
  if (r && this.validInverseReflection(r)) {
    // <- inside the try
    reflection = r;
    break;
  }
}
```

This matters because `validInverseReflection` reads `reflection.foreignKey`,
which for a through reflection resolves `source_reflection` →
`through_reflection.klass` and can raise `NameError`. Under Rails' structure
that error propagates from `:776`; under trails' it is first offered to the
`error.constantName === this.className` filter at the wrong nesting level. It
happens to re-raise today only because the constant names differ — a same-named
miss would be silently swallowed as `reflection = false`.

trails also flattens Rails' two sequential lookups (singular, then plural only
when the first missed and `automatically_invert_plural_associations` is set)
into one `lookupNames` array scanned with a combined validity test. That is a
second, related divergence in the same body: Rails' plural lookup is guarded on
`!reflection`, trails' is not.

## Converged shape

- `try` wraps only the `_reflectOnAssociation` lookups.
- The plural lookup runs only when the singular one missed, matching `:764`.
- `validInverseReflection` is called once, after the block, on the resolved
  candidate — as at `:776`.
- If picking the first _valid_ candidate rather than the first _found_ one is
  load-bearing for the camel/snake name pair trails must try, that deviation is
  justified at the call site with its Rails line, not left implicit.

## Acceptance criteria

- [ ] The `try` in `automaticInverseOf` encloses only the association lookups.
- [ ] The plural-name lookup is guarded on the singular lookup having missed.
- [ ] A regression test covers a `NameError` whose constant name _matches_
      `class_name` raised from `validInverseReflection` (not from the lookup),
      asserting Rails' behaviour; it must fail on baseline.
- [ ] `pnpm parity:api:calls` / `:args` clean; inverse-association, strict-loading
      and through-association suites green on all three lanes.
