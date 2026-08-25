---
title: "Inflector plural rule mis-transcribed: index/vertex/appendix diverge"
status: draft
updated: 2026-08-14
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

The `(matr|vert|ind)(?:ix|ex)` plural rule was mis-transcribed, and the ported
test fixture was then altered to match the broken implementation rather than
the implementation fixed to match Rails.

`packages/activesupport/src/inflector/inflections.ts:159`:

```ts
inflect.plural(/(matr|vert|append)ix|ice$/i, "$1ices");
```

Rails, `activesupport/lib/active_support/inflections.rb:28`:

```ruby
inflect.plural(/(matr|vert|ind)(?:ix|ex)$/i, '\1ices')
```

Three defects in one line:

1. **`append` where Rails has `ind`.** `index` is not in the alternation at
   all, and `appendix` is wrongly included.
2. **`ix|ice$` where Rails has `(?:ix|ex)$`.** The alternation is unparenthesised,
   so it reads as `((matr|vert|append)ix)` OR `(ice$)` — the `$` binds only to
   the `ice` branch, and the `ex` spelling is never matched.
3. Consequently `vertex` misses (only `vertix` would match) while `matrix`
   passes by luck.

Measured against real ActiveSupport (`ruby -e 'require "active_support/all"'`)
on the same words:

| word       | Rails        | trails       |
| ---------- | ------------ | ------------ |
| `index`    | `indices`    | `indexes`    |
| `vertex`   | `vertices`   | `vertexes`   |
| `appendix` | `appendixes` | `appendices` |
| `matrix`   | `matrices`   | `matrices`   |

`appendix` also fails to round-trip: `singularize("appendices")` returns
`appendice`, because the singular rules
(`inflections.ts:189-190`) correctly mirror Rails
(`inflections.rb`, `(vert|ind)ices$ → \1ex` and `(matr)ices$ → \1ix`) and so
know nothing about `append`.

**The fixture was changed to match the bug.** Rails'
`activesupport/test/inflector_test_cases.rb` has

```ruby
"index"  => "indices",   # line 27
"vertex" => "vertices",  # line 93
"matrix" => "matrices",  # line 94
```

while `packages/activesupport/src/inflector.test.ts` asserts
`index: "indexes"` (line 49), `vertex: "vertexes"` (line 76) and
`appendix: "appendices"` (line 77). Rails' fixture has no `appendix` entry.

Everything else checked in the same pass matches Rails exactly, including the
irregulars and uncountables — `octopus → octopi`, `virus → viri`,
`cactus → cactus`, `leaf → leafs`, `person → people`, `child → children`,
`mouse → mice`, `medium → media`, `datum → data`, `analysis → analyses`,
`quiz → quizzes`, `half → halves`, and the uncountables `sheep`, `fish`,
`series`, `information`, `police`. So this is one bad rule, not a systemic
gap.

## Converged shape

```ts
inflect.plural(/(matr|vert|ind)(?:ix|ex)$/i, "$1ices");
```

and restore the three fixture values to the Rails ones, dropping the
`appendix` entry Rails does not have (or asserting `appendixes`, which is
what Rails produces via the generic `/$/ → s` fallback).

## Acceptance criteria

- `pluralize` returns `indices`, `vertices`, `matrices`, `appendixes`.
- `singularize` round-trips each of them.
- `inflector.test.ts` matches `inflector_test_cases.rb` for these words; the
  fixture is corrected toward Rails, not the implementation toward the
  fixture.
- A regression check that the fixture values equal Rails' — these three drifted
  precisely because nothing compared them.
