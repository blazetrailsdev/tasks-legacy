---
title: "LengthValidator :in/:within duck-types a range and re-derives max instead of calling Range#max"
status: ready
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
packages: []
deps: []
deps-rfc: []
est-loc: 90
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`LengthValidator#initialize`
(`vendor/rails/activemodel/lib/active_model/validations/length.rb:16-20`):

```ruby
def initialize(options)
  if range = (options.delete(:in) || options.delete(:within))
    options[:minimum], options[:maximum] = range.min, range.max
  end
  super
end
```

Rails reads the two bounds off the range with `Range#min` / `Range#max` — so
an exclusive end (`6...20`) is handled by `Range#max` itself, and any object
that is not a Range simply does not respond to them.

trails (`packages/activemodel/src/validations/length.ts:56-78`) instead:

- duck-types the value (`"begin" in range || "end" in range`) rather than
  requiring a Range;
- adds an `Array.isArray(range) && range.length === 2` tuple arm Rails has no
  counterpart for;
- re-derives the exclusive-end adjustment inline
  (`r.excludeEnd ? r.end - 1 : r.end`) instead of calling `max`, so it is
  number-only and silently wrong for a Date-ended range;
- raises an invented `ArgumentError(":in and :within must be a [min, max]
tuple or { begin, end, excludeEnd? } object")` at a raise site Rails does
  not have.

PR #6219 made `Range` a real class in `packages/activesupport/src/range-ext.ts`
with `min`-equivalent (`begin`) and a faithful `max()` that carries Ruby's
`TypeError: cannot exclude non Integer end value`, so the Rails body is now
directly expressible.

## Converged shape

```ts
const range = options["in"] ?? options["within"];
if (range !== undefined) {
  delete options["in"];
  delete options["within"];
  options["minimum"] = (range as Range<number>).begin;
  options["maximum"] = (range as Range<number>).max();
}
```

The Array-tuple arm and the invented `ArgumentError` are deleted, not rehomed —
a non-Range value reaches the same "no minimum/maximum" path Rails leaves it on
(`length.rb:22-30`'s `check_validity!`).

## Acceptance criteria

- [ ] `length.ts`'s `:in`/`:within` normalization is the four-line Rails body,
      reading `begin` and `max()` off a `Range`.
- [ ] The `[min, max]` tuple arm and the bespoke `ArgumentError` are removed.
- [ ] Existing `length-validation.test.ts` cases still pass; any that depend on
      the tuple arm are converged to Rails' `6..20` spelling (test names
      unchanged).
- [ ] `pnpm parity:api:calls` clean — the initializer's call set gains `max`.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
