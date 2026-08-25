---
title: "check_schema_file's abort message drops `bin/rails` and the conditional Rails.root sentence"
status: ready
updated: 2026-08-25
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
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

Surfaced while landing `kernel-abort-ports-as-a-catchable-throw` (PR #7032),
which converged the _raise_ half of `check_schema_file` (`Kernel.abort`
semantics) but deliberately left the _message_ half alone as out of scope.

Rails
(`vendor/rails/activerecord/lib/active_record/tasks/database_tasks.rb:482-488`):

```ruby
def check_schema_file(filename)
  unless File.exist?(filename)
    message = +%{#{filename} doesn't exist yet. Run `bin/rails db:migrate` to create it, then try again.}
    message << %{ If you do not intend to use a database, you should instead alter #{Rails.root}/config/application.rb to limit the frameworks that will be loaded.} if defined?(::Rails.root)
    Kernel.abort message
  end
end
```

trails (`packages/activerecord/src/tasks/database-tasks.ts`, `checkSchemaFile`):

```ts
const message = `${filename} doesn't exist yet. Run \`db:migrate\` to create it, then try again.`;
abort(message);
```

Two divergences from a message string CLAUDE.md requires to match exactly
("Errors. Same error class, same message string, same raise site"):

1. **`bin/rails` is dropped** from the first sentence. This is not a trails
   naming decision — the repo keeps `bin/rails` verbatim in ported message
   strings elsewhere in the same package:
   `packages/activerecord/src/migration.ts:244`
   (`"Migrations are pending. To resolve this issue, run:\n\n        bin/rails db:migrate"`)
   and `migration.ts:2213`
   (`"Environment data not found in the schema. To resolve this issue, run: bin/rails db:environment:set"`).
   So `check_schema_file` is the odd one out, not the convention.
2. **The conditional second sentence is missing entirely.** Rails appends the
   "If you do not intend to use a database…" clause whenever `::Rails.root` is
   defined, interpolating it into the `config/application.rb` path. trails has
   the analogue — `trailsRoot` / `setTrailsRoot` in
   `packages/activesupport/src/trails-root.ts` — so the `defined?(::Rails.root)`
   guard has a faithful spelling (`trailsRoot != null`) rather than being
   language-forced.

## Converged shape

Build the message in Rails' two steps, at Rails' site:

```ts
let message = `${filename} doesn't exist yet. Run \`bin/rails db:migrate\` to create it, then try again.`;
if (trailsRoot != null) {
  message += ` If you do not intend to use a database, you should instead alter ${trailsRoot}/config/application.rb to limit the frameworks that will be loaded.`;
}
abort(message);
```

Confirm `trailsRoot`'s unset spelling before wiring the guard — Ruby's
`defined?(::Rails.root)` is false when the constant is absent, so the TS guard
must be false in the equivalent unconfigured state and must not throw.

## Acceptance criteria

- `checkSchemaFile`'s message matches `database_tasks.rb:484-485` verbatim,
  including `bin/rails db:migrate` and the conditional `Rails.root` sentence.
- The `defined?(::Rails.root)` arm is ported through `trailsRoot`, not dropped.
- `DatabaseTasksCheckSchemaFileTest` keeps its Rails test names and covers both
  arms (root set and root unset).
- `pnpm parity:api:calls` / `:args` clean; parity deltas non-negative.
