---
title: "runCmd raises an invented formatCmdError while the Rails-shaped runCmdError is dead code"
status: draft
updated: 2026-08-09
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activerecord"
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

Rails builds the shell-out failure message in one private method,
`run_cmd_error`, and `run_cmd` raises it
(`vendor/rails/activerecord/lib/active_record/tasks/postgresql_database_tasks.rb:112-122`):

```ruby
def run_cmd(cmd, args, action)
  fail run_cmd_error(cmd, args, action) unless Kernel.system(psql_env, cmd, *args)
end

def run_cmd_error(cmd, args, action)
  msg = +"failed to execute:\n"
  msg << "#{cmd} #{args.join(' ')}\n\n"
  msg << "Please check the output above for any errors and make sure that `#{cmd}` is installed in your PATH and has proper permissions.\n\n"
  msg
end
```

trails has **two** functions and uses the wrong one
(`packages/activerecord/src/tasks/postgresql-database-tasks.ts`):

- `formatCmdError` — a trails invention that `runCmd` actually calls. It emits
  different prose ("Make sure `cmd` is installed…" rather than "Please check the
  output above for any errors and make sure that…") and appends sections Rails
  has no concept of: `Error:`, `Exit status:`, `Signal:`, `stderr:`, `stdout:`,
  and a trailing `(action: <action>)`.
- `runCmdError` — an exported function whose body _does_ match Rails' wording,
  but which nothing calls. Dead surface.

So the message a user sees is not Rails', and the function named after Rails'
method is unreachable. This is the `run_cmd` → `run_cmd_error` row in
`scripts/api-compare/call-mismatches-exclude/activerecord/tasks/postgresql-database-tasks.json`.

Note the extra sections exist because trails uses `spawnSync` rather than
`Kernel.system`: Ruby's `system` streams the child's output to the terminal, so
"check the output above" is literally true there, while `spawnSync` captures it.
That is a genuine platform difference and the captured `stderr`/`stdout` may be
worth keeping — but the _base_ message, and the fact that `run_cmd` calls the
method named `run_cmd_error`, should be Rails'.

## Converged shape

One function, `runCmdError(cmd, args, action)`, carrying Rails' three lines, with
`runCmd` calling it — so the call-set row retires. If the captured child output
is kept, append it below Rails' text and justify that at the call site with the
`spawnSync`-vs-`Kernel.system` cite above; do not let it displace Rails' wording.
Delete `formatCmdError`.

`adapters/postgresql/postgresql-rake.test.ts` "structure dump execution fails"
asserts only `toMatch("failed to execute:")`, which both shapes satisfy, so it
does not pin the divergence — check the sqlite sibling before changing wording.

## Acceptance criteria

- [ ] `formatCmdError` is gone; `runCmd` raises `runCmdError(...)`.
- [ ] The message opens with Rails' three lines verbatim
      (`postgresql_database_tasks.rb:116-121`).
- [ ] The `run_cmd` / `run_cmd_error` row is deleted from the call-mismatch
      baseline and `pnpm parity:api:calls` is green.
- [ ] `pnpm parity:api:extra --package activerecord` does not gain a row.
- [ ] Green on the PG lane.
