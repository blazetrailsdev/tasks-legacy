---
title: "Locator.locateMany drops mismatched-app GIDs and parseAllowed filters arity; Rails does neither"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "globalid"
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

## Context

Two related divergences in the `Locator` facade, both currently recorded in
JSDoc as intentional "safer than Rails" behavior. CLAUDE.md treats such a note
as debt, not permission, so both need converging or an explicit block.

### 1. `locate_many` drops mismatched-app GIDs

Rails hands **every** allowed gid to the first gid's locator
(`vendor/globalid/lib/global_id/locator.rb:63-70`):

```ruby
def locate_many(gids, options = {})
  if (allowed_gids = parse_allowed(gids, options[:only])).any?
    locator = locator_for(allowed_gids.first)
    locator.locate_many(allowed_gids, options)
  else
    []
  end
end
```

The documented contract is "All GlobalIDs must belong to the same app, as they
will be located using the same locator" — Rails relies on the caller and does
not police it.

trails filters to only those gids whose app matches the first allowed gid's
(`packages/globalid/src/locator.ts:299-306`, `normalizeApp` comparison), so
mixed-app input silently returns fewer records than Rails would. The JSDoc
calls Rails' behavior "unsafe with BlockLocators that only know one app's
models" and says "we make this explicit".

### 2. `parse_allowed` filters modelId arity

Rails' `parse_allowed` only parses, resolves the model class and applies
`only:` (`locator.rb:216-222`); arity validity is checked later, by
`BaseLocator#model_id_is_valid?` at dispatch time (`locator.rb:180-187`).

trails additionally drops arity-mismatched gids inside `parseAllowed`
(`locator.ts:366-380`), and the JSDoc says it "Extends Rails behavior" so the
`locateMany` first-gid-app selection isn't anchored on a doomed entry — which
only matters because of divergence 1.

The two are entangled: converging 1 likely removes the motivation for 2.

## Converged shape

- `Locator.locateMany`: `parseAllowed` → if empty return `[]`, else
  `locatorFor(allowed[0])` and hand it **all** of `allowed`. Drop the
  `sameApp` filter and the JSDoc paragraph justifying it.
- `Locator.parseAllowed`: parse, `safeConstantize`, `findAllowed` only. Drop
  the `modelIdArityMatches` call and its JSDoc note, letting
  `BaseLocator#modelIdIsValid` do that job where Rails does it.

Watch the facade-level arity check in `Locator.locate` (`locator.ts:263`),
which has its own justification and is a separate question from these two.

## Acceptance criteria

- `Locator.locateMany` matches `locator.rb:63-70` — no app filtering.
- `Locator.parseAllowed` matches `locator.rb:216-222` — no arity filtering.
- Both JSDoc "we make this explicit" / "Extends Rails behavior" notes are
  deleted rather than reworded.
- `global_locator_test.rb` still passes; if a test depended on the filtered
  behavior, it was asserting a trails invention and should be re-checked
  against the Rails original.
