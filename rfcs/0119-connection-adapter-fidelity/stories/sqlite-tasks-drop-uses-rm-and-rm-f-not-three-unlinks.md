---
title: "SQLiteDatabaseTasks#drop hand-unlinks three files where Rails calls FileUtils.rm + one array rm_f"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
packages: []
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

`SQLiteDatabaseTasks#drop` (`packages/activerecord/src/tasks/sqlite-database-tasks.ts:74-96`)
removes the database file and its two WAL sidecars with three separate
`fs.unlinkSync` calls, each wrapped in its own `try`/`catch`:

```ts
try { fs.unlinkSync(file); } catch (error) { ...NoDatabaseError on ENOENT... }
for (const suffix of ["-shm", "-wal"]) {
  try { fs.unlinkSync(file + suffix); } catch { /* ignore */ }
}
```

Rails is two sends (`activerecord/lib/active_record/tasks/sqlite_database_tasks.rb:22-29`):

```ruby
def drop
  db_path = db_config.database
  file = File.absolute_path?(db_path) ? db_path : File.join(root, db_path)
  FileUtils.rm(file)
  FileUtils.rm_f(["#{file}-shm", "#{file}-wal"])
end
```

`FileUtils.rm` raises `Errno::ENOENT` (which `DatabaseTasks.drop`'s rescue
turns into the "does not exist" message via `NoDatabaseError`), and
`FileUtils.rm_f` takes the **array of both sidecars in one call** and swallows
missing files by contract — no per-file `try`.

This surfaced while porting `SqliteDBDropTest#test_removes_file_with_absolute_path`
and `#test_removes_file_with_relative_path`
(`vendor/rails/activerecord/test/cases/adapters/sqlite3/sqlite_rake_test.rb:100-120`),
which assert exactly that call shape:

```ruby
assert_called_with(FileUtils, :rm, [@database_root]) do
  assert_called_with(FileUtils, :rm_f, [["#{@database_root}-shm", "#{@database_root}-wal"]]) do
```

The ported trails tests (PR #6270,
`packages/activerecord/src/adapters/sqlite3/sqlite-rake.test.ts`) had to assert
three individual `unlinkSync` calls instead of one `rm` plus one array-arg
`rm_f`, and carry a comment recording the divergence. That comment is the
receipt this story retires.

## Converged shape

Give the `FsAdapter` seam (`packages/activesupport/src/fs-adapter.ts`) the
`rm` / `rmF` pair Rails uses, so `drop` reads as:

```ts
fs.rm(file);
fs.rmF([`${file}-shm`, `${file}-wal`]);
```

`rmF` takes the list in one call and ignores missing entries; `rm` raises so
the existing `ENOENT -> NoDatabaseError` translation keeps its raise site.
Then rewrite the two ported assertions to the Rails shape (one `rm` call with
the file, one `rmF` call with the two-element array) and delete the deviation
comment above them.

## Acceptance criteria

- [ ] `SQLiteDatabaseTasks#drop` is Rails' two calls, not three unlinks and two
      `try` blocks.
- [ ] The sidecars go in **one** call taking the array, as `rm_f` does.
- [ ] `sqlite_rake_test.rb`'s `test_removes_file_with_absolute_path` /
      `test_removes_file_with_relative_path` assert the Rails call shape and
      the deviation comment in `sqlite-rake.test.ts` is deleted.
- [ ] `pnpm parity:test` keeps `sqlite_rake_test.rb` at 17/17 with
      gate-mismatch 0, and the assertion ratchet stays green.
