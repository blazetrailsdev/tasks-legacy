---
title: "CommandRecorder's command tuple has no block seat, so changeTable's bulk path can't record Rails' lambda"
status: ready
updated: 2026-08-25
rfc: "0051-migration-schema-statements-fidelity"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 200
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

# `CommandRecorder`'s command tuple has no block seat, so `changeTable`'s bulk path can't record Rails' lambda

## Context

Surfaced while converging the tuple to Rails' pair in PR #7020 (RFC 0051),
which deliberately stopped at `[cmd, args]`. The pair is now right; the missing
third element is what's left.

Rails' `record` pushes a THREE-element array — the block rides along:

`vendor/rails/activerecord/lib/active_record/migration/command_recorder.rb:104-111`

```ruby
def record(*command, &block)
  if @reverting
    @commands << inverse_of(*command, &block)
  else
    @commands << (command << block)
  end
end
```

and `replay` splats it back apart with the block restored
(`command_recorder.rb:148-152`):

```ruby
def replay(migration)
  commands.each do |cmd, args, block|
    migration.send(cmd, *args, &block)
  end
end
```

trails (`packages/activerecord/src/migration/command-recorder.ts`) records
`[cmd, args]` and `replay` iterates `[cmd, args]`, so there is nowhere for a
block to live. Two consequences are already flagged at their call sites and
would both retire with this:

1. `changeTable`'s bulk path carries a `@missingRailsCall bulk_change_table`
   tag (command-recorder.ts, `changeTable` JSDoc). Rails records that path as a
   lambda — `-> t { bulk_change_table(table_name, commands) }`
   (`command_recorder.rb:142`) — which needs the block seat. trails instead
   records `["changeTable", [tableName, commands]]`.
2. `replay` therefore carries an extra `cmd === "changeTable" && Array.isArray(args[1])`
   branch that re-dispatches each sub-command by hand. Rails' `replay` is the
   three lines above and has no such branch.

## Converged shape

`_commands` entries are `[cmd, args, block]` as `@commands` holds
(`command_recorder.rb:109`); `record` appends the block, `inverseOf` forwards it
(`command_recorder.rb:114-123` takes `&block` and passes it to the `invert_*`
method), and `replay` destructures all three and re-supplies the block, so
`changeTable`'s bulk path records Rails' lambda and `replay`'s extra branch
disappears along with the `@missingRailsCall` tag.

## Acceptance criteria

- [ ] `_commands` entries carry the block seat, matching `command_recorder.rb:109`.
- [ ] `changeTable`'s bulk path records the `bulk_change_table` lambda
      (`command_recorder.rb:142`) and its `@missingRailsCall` tag is deleted.
- [ ] `replay` is Rails' three-line body (`command_recorder.rb:148-152`) with no
      `changeTable` special case.
- [ ] `command-recorder{,.trails}.test.ts` and `invertible-migration.test.ts`
      green; SQLite, PostgreSQL and MySQL/MariaDB lanes green.
