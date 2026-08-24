---
title: "Kernel.abort ports as a catchable throw in migrate_status and check_schema_file"
status: ready
updated: 2026-08-24
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 60
priority: 39
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

Rails' `DatabaseTasks` calls `Kernel.abort` in two places, and trails ports both
as a plain `throw new Error(...)` with the same message:

| Rails                                                                                                                                                           | trails                                                                                     |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| `migrate_status` — `Kernel.abort "Schema migrations table does not exist yet."` (`vendor/rails/activerecord/lib/active_record/tasks/database_tasks.rb:303-305`) | `DatabaseTasks.migrateStatus` throws (`packages/activerecord/src/tasks/database-tasks.ts`) |
| `check_schema_file` — `Kernel.abort "#{filename} doesn't exist yet..."` (`database_tasks.rb:482-487`)                                                           | `DatabaseTasks.checkSchemaFile` throws (same file)                                         |

`Kernel.abort` writes to stderr and **exits the process** with status 1; it is
not an exception a caller can catch and continue from. A `throw` is, and the
difference is observable: a caller wrapping either method in a `try` sees a
recoverable error where Rails' process would already be gone, and the CLI's
generic error-to-exit-code path reformats the message rather than emitting it
bare on stderr.

Both sites are cited as language-forced in JSDoc today, which is the
placeholder PR #6980 chose deliberately for `migrate_status` (its story allowed
either a fix or a citation). This story is the fix.

## Converged shape

A single `abort(message)` helper with `Kernel.abort` semantics — bare message
to stderr, process exit status 1 — used at both sites, so `migrateStatus` and
`checkSchemaFile` are non-returning where Rails' are. It has to stay testable
(the suite must not exit), so route it through the existing process adapter
(`@blazetrails/activesupport/process-adapter`, which already owns `setExitCode`
and is stubbed in tests) rather than calling `process.exit` directly.

Check whether other `Kernel.abort` call sites have been ported since —
`grep -rn "Kernel.abort" vendor/rails/activerecord/lib` — and route them through
the same helper.

## Acceptance criteria

- [ ] One helper with `Kernel.abort` semantics, used by both `migrateStatus`
      and `checkSchemaFile`; the two `@noRailsEquivalent` / language-forced
      JSDoc citations at those sites are removed.
- [ ] The message text still matches Rails verbatim at both sites.
- [ ] Exiting is routed through the process adapter, so the vitest suite does
      not exit; a test asserts the abort path sets a non-zero exit code.
- [ ] `packages/activerecord-cli/src/db-migrate-status.test.ts` and the
      `DatabaseTasksCheckSchemaFileTest` cases keep their names and pass.
