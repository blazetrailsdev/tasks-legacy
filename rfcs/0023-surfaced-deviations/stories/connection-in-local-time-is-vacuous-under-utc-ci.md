---
title: "default_timezone: local is ignored by the datetime cast; its test is vacuous under TZ=UTC"
status: draft
updated: 2026-08-05
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activemodel"
  - "activerecord"
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

`base.test.ts`'s `connection in local time` (`BasicsTest`, mirrors
`activerecord/test/cases/base_test.rb`'s `test_connection_in_local_time`) is
**vacuous in CI and fails on a developer machine**, because CI runs with
`TZ=UTC` where the assertion degenerates to an identity.

The test establishes a connection with `default_timezone: "local"` and asserts
the `"2004-01-01 00:00:00"` default casts through the _local_ zone:

```ts
const expected = Temporal.PlainDateTime.from("2004-01-01T00:00:00")
  .toZonedDateTime(Temporal.Now.timeZoneId())
  .toInstant();
expect(ft.epochNanoseconds).toBe(expected.epochNanoseconds);
```

Under `TZ=UTC` both sides are `2004-01-01T00:00:00Z`, so the assertion passes
whether or not `default_timezone: "local"` was honored. Under a non-UTC host it
fails:

```text
ARCONN=postgresql pnpm vitest run packages/activerecord/src/base.test.ts -t "connection in local time"
AssertionError: expected 1072915200000000000n to be 1072933200000000000n
```

`1072915200e9` is `2004-01-01T00:00:00Z` — i.e. the value was cast as **UTC**.
`1072933200e9` is `2004-01-01T05:00:00Z`, the correct America/New_York (EST,
-05:00) reading. So the received value shows `default_timezone: "local"` is
**not applied** on the read path; the cast used UTC regardless.

Verified on `origin/main` at `60b5ef2dd` (pre-dates PR #6109, which only
surfaced it) — reproduced on the PG lane on a `TZ=America/New_York` host, and
the sibling `connection in utc time` passes there, so the UTC arm works and only
the local arm is broken.

## Rails reference

`ActiveRecord.default_timezone` is read by the type cast, not the connection:
`activerecord/lib/active_record/type/date_time.rb` → `ActiveModel::Type::DateTime`
→ `fast_string_to_time` / `fallback_string_to_time`
(`activemodel/lib/active_model/type/helpers/time_value.rb:60-80`), where
`default_timezone == :local` selects `Time.local(...)` over `Time.utc(...)`
(`:73-79`). `establish_connection(default_timezone:)` updates the singleton the
cast reads (`activerecord/lib/active_record/connection_adapters/abstract/connection_handler.rb`
→ `ActiveRecord.default_timezone=`).

trails' counterpart is `DateTimeType.cast` → `fastStringToTime`
(`packages/activerecord/src/type/date-time.ts`,
`packages/activemodel/src/type/helpers/time-value.ts`) reading
`ActiveRecord.defaultTimezone`. The `:local` branch is what to check first.

## Converged shape

Make the local branch actually interpret an offset-less string in the host zone,
so `default_timezone: "local"` round-trips. Then make the test assertion
non-vacuous under UTC as well — pin an explicit non-UTC zone for the duration
(the file already has `withTimezoneConfig`) rather than reading
`Temporal.Now.timeZoneId()`, so CI exercises the branch it is meant to cover.

Do NOT change the test name (`parity:test` matches on it).

## Acceptance criteria

- [ ] `default_timezone: "local"` is honored by the datetime cast path; the
      offset-less default reads back as the host-zone instant.
- [ ] `connection in local time` asserts a real offset under `TZ=UTC`, so the
      branch is covered in CI rather than degenerating to an identity.
- [ ] Test passes on SQLite, PG and MySQL on both a UTC and a non-UTC host.
- [ ] `pnpm parity:test` delta non-negative; test name unchanged.
