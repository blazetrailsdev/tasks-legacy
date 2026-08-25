---
title: "remove_prefix_and_suffix uses Regexp.escape, not a file-local escape helper"
status: draft
updated: 2026-08-10
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

Surfaced by the RFC 0095 call-argument dimension (PR #6334) as a `naming` row
that is really an invented helper.

**Rails** (`activerecord/lib/active_record/schema_dumper.rb:366-374`):

    def remove_prefix_and_suffix(table)
      return table if @options[:table_name_prefix].blank? && @options[:table_name_suffix].blank?

      prefix = Regexp.escape(@options[:table_name_prefix].to_s)
      suffix = Regexp.escape(@options[:table_name_suffix].to_s)
      table.sub(/\A#{prefix}(.+)#{suffix}\z/, "\\1")
    end

**trails** (`packages/activerecord/src/schema-dumper.ts:704-711`) declares a
local `escape` arrow function — `(s) => s.replace(/[.*+?^${}()|[\]\\]/g, "\\$&")`
— and calls `escape(prefix)` / `escape(suffix)` where Rails calls
`Regexp.escape`. It also drops the `.to_s` and reads the options with `?? ""`
rather than Rails' `blank?` guard on the raw values.

`Regexp.escape` is not a per-call-site helper: it is a Ruby stdlib primitive
several ported files need, and re-declaring it inline in each one is exactly the
extra-surface pattern `pnpm parity:api:extra` measures.

## Converged shape

One shared `Regexp.escape` equivalent in the activesupport/Ruby-stdlib shim
layer (check for an existing one first — `regexpEscape` may already exist), used
by name at both call sites, with the `blank?` guard and `.to_s` ported as
written.

## Acceptance criteria

1. `removePrefixAndSuffix` mirrors `schema_dumper.rb:366-374` line for line: the
   `blank?` early return on both options, `Regexp.escape(...to_s)` for each, and
   the single `sub`.
2. No file-local `escape` helper remains in `schema-dumper.ts`.
3. If a shared escape helper has to be added, it carries the Ruby name and lives
   where the other Ruby-stdlib shims live — not in `schema-dumper.ts`.
