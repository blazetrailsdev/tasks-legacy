---
title: "No-arg establishConnection must resolve from the configurations registry, not fall back to disk"
status: done
updated: 2026-08-22
rfc: "0112-one-rails-thing-n-trails-things"
cluster: split-stores
deps: []
deps-rfc: []
est-loc: 150
pr: 6777
claim: "2026-08-20T17:22:15Z"
assignee: "converge-event-children-invention"
blocked-by: null
closed-reason: null
---

## Context

Rails' no-arg `establish_connection` does not touch the filesystem:

```ruby
def establish_connection(config_or_env = nil)
  config_or_env ||= DEFAULT_ENV.call.to_sym
  db_config = resolve_config_for_connection(config_or_env)
  connection_handler.establish_connection(db_config, owner_name: self, ...)
end
```

(`vendor/rails/activerecord/lib/active_record/connection_handling.rb:50-54`.)
`config/database.yml` reaches `Base.configurations` once, at boot, via the
Railtie — never from `establish_connection`.

trails' `autoConnect` (`packages/activerecord/src/connection-handling.ts`)
instead falls back to `loadConfigFile(modelClass)` whenever the registry is
empty, which reads `config/database.json` off disk through a trails-only
`_configPath` static and the `Trails.root` seam. `loadConfigFile` and
`_configPath` have no Rails counterpart, and the fallback means an empty
registry silently produces a connection from a file rather than surfacing the
same error Rails would.

Noticed while converging `configurations` storage onto the global registry
(PR #5386). That PR left the fallback untouched — it is a separate deviation
with its own blast radius, since the disk path is what several suites and the
`Trails.root` test at `connection-handling.test.ts` ("loads config/database.json
from the injected root") currently exercise.

## Acceptance criteria

- No-arg `establishConnection` resolves only through the global
  `configurations` registry, as connection_handling.rb:50-54 does.
- `loadConfigFile` / `_configPath` either move to the boot seam that populates
  `Base.configurations` (the trailties Railtie analogue,
  `packages/trailties/src/database.ts`) or are deleted if that seam already
  covers it — they must not be reachable from `establishConnection`.
- An empty registry surfaces the Rails error rather than reading a file.
- The `Trails.root` config-loading test is rehomed onto whatever seam keeps the
  behavior, not deleted — the capability is still wanted, just not from
  `establish_connection`.
- Check the trailties boot path first: if it already loads database config into
  the registry, this is mostly a deletion.
- Suites pass: connection-handling, connection-management, test-databases,
  database-tasks, trailties database.

## Re-homed from `0023-surfaced-deviations` (2026-08-18)

Moved by the RFC 0023 backlog triage pass into `0112-one-rails-thing-n-trails-things`, which was carved out
of that register for this deviation class. Nothing about the finding changed —
every Rails and trails `file:line` citation above is as originally filed.
