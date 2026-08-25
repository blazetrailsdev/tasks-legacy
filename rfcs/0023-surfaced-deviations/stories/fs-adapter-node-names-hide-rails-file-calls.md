---
title: "fs-adapter's node-shaped names hide Rails File calls"
status: draft
updated: 2026-08-12
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 150
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Surfaced in #6437. Rails' cache FileStore reaches the filesystem through Ruby's
`File`/`Dir`/`FileUtils`, and the call-parity gate matches on those Ruby method
names. trails routes every filesystem call through the fs-adapter
(`packages/activesupport/src/fs-adapter.ts`), whose members are named after
node's `fs` (`existsSync`, `statSync(...).isDirectory()`), so a body that makes
exactly the Rails call still reads as a missing call.

Five baselined rows in
`scripts/api-compare/call-mismatches-exclude/activesupport/cache/file-store.json`
are this one cause:

- `ensure_cache_path` / `exist?` — `File.exist?(path)`, file_store.rb:201
- `read_serialized_entry` / `exist?` — file_store.rb:123
- `write_serialized_entry` / `exist?` — file_store.rb:133
- `search_dir` / `exist?` — file_store.rb:205
- `search_dir` / `directory?(ref:name)` (args) — `File.directory?(name)`, file_store.rb:209

`delete_entry`/`exist?` and `delete_empty_directories`/`realpath` carry the same
cause from the RFC 0047 seed.

The converged shape is a Ruby-named `File` façade over the adapter — `exist?` →
`exist`, `directory?` → `isDirectory`, `realpath`, `delete`, `dirname` — so the
ported bodies name what Rails names and the rows delete. That is a shared
activesupport surface, not a FileStore-local helper, and it needs the
Ruby→TS naming rule for `?`-predicates in `scripts/parity/conventions.ts` checked
first so the façade's members actually credit.

## Acceptance criteria

- Cache FileStore's filesystem calls go through Ruby-named members rather than
  the node-shaped adapter names.
- The five rows above (plus the two seeded siblings if they fall out) are deleted
  from the call-mismatch baseline.
- `pnpm parity:api` delta non-negative.
