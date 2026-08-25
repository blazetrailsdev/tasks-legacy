---
title: "SpellChecker#correct threads invented {word,index,score} objects instead of Ruby's word array"
status: draft
updated: 2026-08-11
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "did-you-mean"
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced by the RFC 0096 naming burndown (PR #6378) as an a3 (structural)
row that must not be closed by renaming a local:
`spell-checker.ts#correct` calling `normalize` — Ruby passes `ref:c`, trails
passes `ref:word`.

Ruby (`vendor/did_you_mean/lib/did_you_mean/spell_checker.rb:12-33`) works on a
bare array of dictionary words and re-derives what it needs:

```ruby
words = @dictionary.select { |word| JaroWinkler.distance(normalize(word), normalized_input) >= threshold }
words.reject! { |word| input.to_s == word.to_s }
words.sort_by! { |word| JaroWinkler.distance(word.to_s, normalized_input) }
words.reverse!

threshold   = (normalized_input.length * 0.25).ceil
corrections = words.select { |c| Levenshtein.distance(normalize(c), normalized_input) <= threshold }
```

trails (`packages/did-you-mean/src/spell-checker.ts:34-76`) instead builds an
array of `{ word, index, score }` candidate objects in one pass and threads
`c.word` / `c.score` through the later steps. That is an invented
intermediate shape: it collapses Ruby's four sequential array operations into
one loop plus a comparator, so the Rails reader cannot line the bodies up.

The `index` field exists only to emulate MRI's unstable `sort_by` + `reverse!`
tie order — see the comment at `spell-checker.ts:52-56`. Any convergence has to
keep that observable ordering; check the real MRI behaviour with `ruby` (it is
on PATH) rather than deriving it.

## Acceptance criteria

- `correct` mirrors `spell_checker.rb:12-33` step by step: a `words` array of
  dictionary words, then the reject, then the sort, then the reverse, then the
  two `corrections` selects — same locals (`words`, `threshold`, `corrections`,
  `normalizedInput`), same order.
- The `{ word, index, score }` object disappears; if tie ordering still needs a
  stable-sort workaround, it is confined to the sort step and justified there
  as a language shortcoming, not spread through the body.
- `pnpm parity:api:calls:args:report` shows the `did-you-mean` `naming` row for
  `correct -> normalize` gone, with no new `shape` row.
- The package's existing tests pass unchanged (do not rename them).
