---
title: "establish_connection's adapter backfill rewrites a registry config hash; Rails never touches it"
status: ready
updated: 2026-08-25
rfc: "0119-connection-adapter-fidelity"
cluster: null
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

`establishWithDbConfig` (`packages/activerecord/src/connection-handling.ts:851-860`)
backfills the adapter into the config hash when an adapter-less `DatabaseConfig`
carries a derivable URL:

```ts
if (!dbConfig.adapter) {
  configForConnect = _setConfigurationHash(dbConfig, { ...config, adapter: adapterName });
}
```

Rails' `establish_connection`
(`vendor/rails/activerecord/lib/active_record/connection_handling.rb:66-79`)
never touches `configuration_hash` — its URL configs always parse a scheme, so
`configuration_hash[:adapter]` is never absent. trails needs the backfill only
because of scheme-less SQLite shorthand (`:memory:`, bare paths), which
`buildUrlHash` (`database-configurations/url-config.ts:102-119`) handles for
`UrlConfig` but not for a plain `HashConfig`.

Two consequences: the rewrite outlives the `establishConnection` call (the
config generally lives in the `Base.configurations()` registry), and
`_setConfigurationHash` (`database-configurations/database-config.ts:72-88`)
exists solely to serve it — it is the only non-`_database=` write path to a
config hash.

Surfaced in review of PR #5509 (config hashes are now frozen at construction,
mirroring `hash_config.rb:38-41`).

## Acceptance criteria

- [ ] Adapter-less-but-URL-derivable configs get their adapter at build time
      (as `UrlConfig` already does) rather than at connect time, OR the
      deviation is documented with the reason it must stay.
- [ ] If the backfill goes away, `_setConfigurationHash` goes with it and
      `#configuration` is written only by `_database=`, matching Rails'
      `attr_reader :configuration_hash` exactly.
- [ ] The regression test `establishConnection backfills the adapter on an
adapter-less HashConfig` (`connection-handling.test.ts`) is updated to
      the converged behaviour, not deleted.
