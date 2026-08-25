---
title: "SqlLiteral#toYAML is invented surface; Rails' encode_with dumps a plain scalar"
status: draft
updated: 2026-08-21
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "arel"
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

`Arel::Nodes::SqlLiteral` subclasses `String` and its only YAML surface is

    # vendor/rails/activerecord/lib/arel/nodes/sql_literal.rb:18-20
    def encode_with(coder)
      coder.scalar = self.to_s
    end

so dumping one yields a plain YAML **scalar** — the literal SQL text, nothing
more — because Psych serializes it as the String it is.

trails has neither the name nor the shape. `packages/arel/src/nodes/sql-literal.ts`
carries a `toYAML()` with no Rails counterpart:

    toYAML(): string {
      const escaped = this.value.replace(/\n/g, "\\n");
      return `---\n!sql_literal\nvalue: ${JSON.stringify(escaped)}`;
    }

That emits a tagged mapping (`!sql_literal` with a `value:` key), not a scalar,
and escapes newlines itself rather than letting the emitter do it. Two separate
divergences: an invented public name, and output Rails would never produce.

Surfaced while sweeping free-form comments out of `arel` — the deleted comment
read "Minimal YAML-ish representation for test parity (no external deps)",
which is a description of the deviation, not a justification for it.

## Acceptance criteria

- [ ] `toYAML` is gone; the Rails name `encodeWith` carries the behaviour, with
      the coder-shaped argument the rest of the port uses (or, if nothing in
      trails consumes a coder, the method is deleted outright rather than
      renamed — check callers first).
- [ ] The emitted YAML is the scalar `to_s`, matching Psych's output for a
      String subclass — no `!sql_literal` tag, no `value:` key, no hand-rolled
      newline escaping.
- [ ] `pnpm parity:api:extra --package arel` no longer reports `toYAML` as
      extra surface.
- [ ] Any test asserting the old tagged-mapping text is updated to assert the
      Rails shape, not renamed.
