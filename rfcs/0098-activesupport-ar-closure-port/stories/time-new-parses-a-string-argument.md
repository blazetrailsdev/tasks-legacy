---
title: "Time.new takes MRI's string form, the only path precision: acts on"
status: in-progress
updated: 2026-08-23
rfc: "0098-activesupport-ar-closure-port"
cluster: null
packages: []
deps: []
deps-rfc: []
est-loc: 260
priority: 4
pr: 6931
claim: "2026-08-23T17:44:08Z"
assignee: "encryption-schemes-test-lacks-transactional-fixtures"
blocked-by: null
closed-reason: null
---

## Context

MRI's `Time.new` takes a STRING as its first argument and parses it —
`Time.new("2000-12-31 23:59:59.56789")` — and that form is the ONLY place
`precision:` does anything: it trims the parsed sub-second to N digits.
Confirmed live on `ruby 3.3.11`:

```ruby
Time.new("2000-12-31 23:59:59.56789", precision: 3) == Time.new("2000-12-31 23:59:59.567")
#=> true
Time.new(2020, 1, 1, 0, 0, 0.56789, precision: 3).nsec  #=> 567890000   (NOT trimmed)
Time.new("2000-12-31 23:59:59.5", precision: 0)
#=> ArgumentError: subsecond expected after dot: 23:59:59.5
```

trails' `Time.new` (`packages/date/src/time.ts`) takes `precision:` — PR #6888
added it so `travel_to`'s stub sees a non-empty argument list
(`time_helpers.rb:180-187`) — but routes a String first argument through
`obj2vint`, the numeric conversion, so there is no parse form for `precision:`
to act on. The keyword is therefore accepted and, exactly as in MRI without a
string, changes nothing. That is faithful for the ported shapes and is
documented at the call site, but it leaves the string constructor unported.

## What it blocks

Four Rails assertion arms are dropped in
`packages/activesupport/src/time-travel.test.ts` for want of this form —
`time_travel_test.rb:55`, `:61` (`test_time_helper_travel`,
`test_time_helper_travel_with_block`) and the same pair inside the with-usec
test:

```ruby
assert_equal Time.new("2000-12-31 23:59:59.567"), Time.new("2000-12-31 23:59:59.56789", precision: 3)
```

The sibling `Time.new(precision: 3)` arms in those same tests already landed
with #6888; only the string-form ones remain out.

## Converged shape

`Time.new` takes a String first argument and parses it as MRI's `time_s_init`
does — the ISO-8601-ish `YYYY-MM-DD hh:mm:ss[.frac][±hh:mm|Z]` grammar, with
MRI's own errors (`ArgumentError: subsecond expected after dot: ...` when
`precision:` would truncate every digit, `ArgumentError: can't parse: ...` for
a malformed string). `precision:` then truncates the parsed sub-second to that
many digits before the `Rational` reaches `#atInstant`, and stays a no-op on
every non-string path. A zone suffix in the string binds the same seat the
seventh positional and `in:` bind, so the existing
"timezone argument given as positional and keyword arguments" guard covers the
string-plus-keyword collision too.

Note `packages/activerecord` already has a separate, narrower string→Time
grammar (`fast-string-to-time`, RFC 0023 story
`fast-string-to-time-grammar-diverges-from-ruby-time-new`, closed) — that is
ActiveRecord's cast path, not `Time.new`, and should not be reused here.

## Acceptance criteria

- [ ] `Time.new("2000-12-31 23:59:59.56789", { precision: 3 })` equals
      `Time.new("2000-12-31 23:59:59.567")`.
- [ ] `Time.new("2020-01-01 00:00:00", { in: "+05:00" }).utcOffset` is `18000`,
      and a zone in the string plus `in:` raises MRI's ArgumentError.
- [ ] `precision:` still changes nothing on the positional path:
      `Time.new(2020, 1, 1, 0, 0, 0.56789, { precision: 3 }).nsec` is
      `567890000`.
- [ ] The four `assert_equal Time.new("2000-12-31 23:59:59.567"), ...` arms are
      restored in `time-travel.test.ts` (`time_travel_test.rb:55,61` and the
      with-usec pair). No test renames.
- [ ] `pnpm parity:test:assertions` green.
