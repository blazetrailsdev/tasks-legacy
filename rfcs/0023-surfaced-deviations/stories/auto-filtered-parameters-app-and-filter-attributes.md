---
title: "AutoFilteredParameters synthesizes `app` and never writes klass.filter_attributes"
status: draft
updated: 2026-08-18
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
deps: []
deps-rfc: []
est-loc: 160
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# AutoFilteredParameters synthesizes `app` and never writes klass.filter_attributes

## Context

Surfaced in PR #6719 (RFC 0106 wave 4c, encryption slice) while converging
`apply_filter`'s `compact.join` and `excluded_from_filter_parameters?`'s `find`.
Those two rows converged; the surrounding structure did not, and is a larger
gap than either.

`vendor/rails/activerecord/lib/active_record/encryption/auto_filtered_parameters.rb:6-12,52-57`:

```ruby
def initialize(app)
  @app = app
  ...
end

def apply_filter(klass, attribute)
  filter = [("#{klass.model_name.element}" if klass.name), attribute.to_s].compact.join(".")
  unless excluded_from_filter_parameters?(filter)
    app.config.filter_parameters << filter unless app.config.filter_parameters.include?(filter)
    klass.filter_attributes += [ attribute ]
  end
end
```

`packages/activerecord/src/encryption/auto-filtered-parameters.ts` diverges on
three axes:

1. **`app` is synthesized.** The constructor takes a bare `string[]` and a
   private getter fabricates `{ config: { filter_parameters } }` (:41-44) so the
   rest of the class can pretend it has an application. Rails takes the
   application object. The fabricated shape also spells the inner key
   `filter_parameters` — snake_case in TS — because nothing real is behind it.
2. **`klass.filter_attributes += [attribute]` is missing entirely.** Rails
   writes the attribute onto the MODEL's own filter list as well as the app's.
   trails only pushes to the array. Any consumer reading
   `Model.filterAttributes` sees nothing.
3. **Two invented guards in `apply_filter`**: an early
   `if (!Configurable.config.addToFilterParameters) return` (Rails gates
   installation, not each call — configurable.rb wires the hook only when the
   config is on) and a second exclusion check against the bare `attribute` name
   beside the one against `filter`. Rails checks the dotted `filter` only.

## Converged shape

- Constructor takes the application object, `app` becomes the plain
  `attr_reader :app` (auto_filtered_parameters.rb:21) over it, and
  `app.config.filterParameters` is read off it.
- `applyFilter` becomes the four lines above: build the filter, `unless
excluded`, push-if-absent, then `klass.filterAttributes += [attribute]`.
- Delete the `addToFilterParameters` early return and the second exclusion
  check; move the config gate to the installation site so it matches Rails.
- `dispose()` (:22-25) has no Rails counterpart and exists only for test
  teardown — either fold it into the tests or tag it `@noRailsEquivalent`.

Depends on `auto-filtered-parameters-model-name-element` for the tests: both
stories need the two `AutoFilteredParameters` cases in `configurable.test.ts`
moved off their bespoke `class PaymentModel {}` mock onto the canonical `Pirate`
subclasses Rails uses (configurable_test.rb:44-85), since a bare class has
neither `modelName` nor `filterAttributes`. Land that one first, or land them
together.

## Acceptance criteria

- [ ] `AutoFilteredParameters` takes the application object; the fabricated
      `app` getter and its snake_case `filter_parameters` key are gone.
- [ ] `applyFilter` writes `klass.filterAttributes` as well as
      `app.config.filterParameters`, matching
      `auto_filtered_parameters.rb:54-56`.
- [ ] The `addToFilterParameters` early return and the duplicate bare-attribute
      exclusion check are deleted; the config gate lives at the installation
      site.
- [ ] `dispose()` is removed or tagged.
- [ ] A test asserts `Model.filterAttributes` picks up the encrypted attribute —
      the arm currently unobservable.
- [ ] `pnpm parity:api:extra --package activerecord` shows no novel names for
      `encryption/auto-filtered-parameters.ts`.
- [ ] `pnpm parity:api:calls` green; `pnpm parity:test` delta non-negative;
      SQLite, PostgreSQL and MySQL/MariaDB lanes green.
