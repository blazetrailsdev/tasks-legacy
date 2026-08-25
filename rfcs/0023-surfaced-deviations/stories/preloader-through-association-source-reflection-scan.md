---
title: "Preloader::ThroughAssociation#source_reflection is a hand-rolled candidate scan Rails does not have"
status: draft
updated: 2026-08-18
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

# Preloader::ThroughAssociation#source_reflection is a hand-rolled candidate scan Rails does not have

## Context

Surfaced in review of PR #6723 (`through-reflection-source-name-swallows-nameerror`),
which deleted `ThroughReflection#source` — an invented getter — and found this
file reading it as `(this.reflection as any).source`, invisible to both grep and
`tsc`.

Rails' `Preloader::ThroughAssociation` delegates in one line each:

`activerecord/lib/active_record/associations/preloader/through_association.rb:82-88`

```ruby
def through_reflection
  reflection.through_reflection
end

def source_reflection
  reflection.source_reflection
end
```

trails' `packages/activerecord/src/associations/preloader/through-association.ts`
has, in place of those two delegations, two private getters carrying a fallback
ladder with no Rails counterpart at all:

- `throughReflection` (`:463-473`) — falls back to scanning
  `activeRecord._associations` for an entry whose `options.through` names the
  through association, then re-resolves it via `_reflectOnAssociation`.
- `sourceReflection` (`:475-506`) — after the primary
  `reflection.sourceReflection` returns null, re-derives candidate source names
  and probes `throughKlass._reflectOnAssociation` against
  `[name, pluralize(name), singularize(name)]`.

PR #6723 converged the _reader_ inside that ladder (`.source` →
`sourceReflectionNames()`, `reflection.rb:1108-1110`) because it was deleting the
property the old line read, but deliberately left the ladder itself in place —
minimal, in-place fix. The ladder is the actual debt.

Measured while fixing PR #6723: across the whole `packages/activerecord/src/associations/`
suite the `sourceReflection` getter is called several hundred times and **every**
call returns on its first line (`reflection.sourceReflection`). The fallback tail
did not execute once. That is evidence the ladder is dead in the exercised
shapes, but not proof it is dead for all of them — which is exactly what the
convergence needs to establish.

## Converged shape

- Reduce both getters to Rails' one-line delegations
  (`reflection.throughReflection` / `reflection.sourceReflection`).
- If a suite reds, the failure names the real gap — a reflection that does not
  resolve its own source/through where Rails' does — and that gap is the thing to
  fix, in `reflection.ts`, rather than re-adding a scan in the preloader.
- The `pluralize` / `singularize` imports become unused; drop them.

## Acceptance criteria

- [ ] `throughReflection` and `sourceReflection` in
      `associations/preloader/through-association.ts` are one-line delegations
      matching `through_association.rb:82-88`.
- [ ] No `_associations` scan and no `_reflectOnAssociation` candidate probing
      remains in that file.
- [ ] `pnpm parity:api:extra --package activerecord` for
      `associations/preloader/through-association.ts` shows no novel surface
      beyond what the file needs.
- [ ] Preloader, nested-through and has-many-through suites green on SQLite,
      PostgreSQL and MySQL/MariaDB.
- [ ] Any genuine resolution gap uncovered is fixed in `reflection.ts` (or filed
      with its Rails `file:line`), never re-hidden behind a preloader-side scan.
