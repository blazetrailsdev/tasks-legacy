---
title: "AttributeSet#deepDup threads a clone cache instead of transform_values(&:deep_dup)"
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

## Context

Surfaced by PR #6558 (RFC 0096 wave-4 naming burndown for activemodel). A
`class: "naming"` call-argument row on `packages/activemodel/src/attribute-set.ts`
survives because trails' `deepDup` is a restructured port, not a line-by-line
one.

Rails, `activemodel/lib/active_model/attribute_set.rb:73-75`:

    def deep_dup
      AttributeSet.new(attributes.transform_values(&:deep_dup))
    end

trails, `attribute-set.ts:182-191`:

    deepDup(): AttributeSet {
      const newAttrs = new Map<string, Attribute>();
      const cache = new Map<Attribute, Attribute>();

      for (const [name, attr] of this.attributes) {
        newAttrs.set(name, this.cloneAttribute(attr, cache));
      }

      return new AttributeSet(newAttrs);
    }

`cloneAttribute(attr, cache)` (attribute-set.ts:409) threads an identity cache
that Rails has no counterpart for, and it is shared with the in-place variant at
attribute-set.ts:220 — so this cannot be converged by editing `deepDup` alone.
Rails' `Attribute#deep_dup` (`attribute.rb`) is a plain per-attribute dup with no
cross-attribute memo, so the cache is trails-invented surface and the question
is whether anything actually depends on shared identity across a `deepDup`.

## Converged shape

    deepDup(): AttributeSet {
      return new AttributeSet(transformValues(this.attributes, (attr) => attr.deepDup()));
    }

with the clone cache removed (or, if a caller genuinely depends on it, that
caller converged instead and the cache deleted). The `attribute-set.ts` / `new`
naming row then clears in `pnpm parity:api:calls:args:report`.

## Acceptance criteria

- [ ] `deepDup` mirrors attribute_set.rb:73-75, with no clone cache.
- [ ] The in-place variant at attribute-set.ts:220 is converged to its own Rails
      counterpart rather than left holding the cache alone.
- [ ] The `new` naming row clears; no new `shape` row; no baseline row added.
- [ ] activemodel + activerecord dirty/attribute suites green on all three lanes.
