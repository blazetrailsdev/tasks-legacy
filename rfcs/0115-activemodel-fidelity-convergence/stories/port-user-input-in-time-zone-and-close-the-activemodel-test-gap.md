---
title: "Port test_user_input_in_time_zone and close activemodel's last parity:test gap"
status: ready
updated: 2026-08-24
rfc: "0115-activemodel-fidelity-convergence"
cluster: null
packages: ["activemodel"]
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

`parity:test` reports **activemodel at 961/963 (99.8%)**, 56/56 files mapped, 0
misplaced (PR #6961's comparison job, 2026-08-24). The two-test gap is not two
unported tests — it is one genuinely missing test and one documented skip:

| Ruby file                                     | OK  | Skip  | Miss  | Tot |
| --------------------------------------------- | --- | ----- | ----- | --- |
| `type/time_test.rb`                           | 2   | 0     | **1** | 3   |
| `validations/numericality_validation_test.rb` | 40  | **1** | 0     | 41  |

Every other activemodel file is at Miss=0.

### The missing test: `test_user_input_in_time_zone`

`vendor/rails/activemodel/test/cases/type/time_test.rb:24-38`:

```ruby
def test_user_input_in_time_zone
  ::Time.use_zone("Pacific Time (US & Canada)") do
    type = Type::Time.new
    assert_nil type.user_input_in_time_zone(nil)
    assert_nil type.user_input_in_time_zone("")
    assert_nil type.user_input_in_time_zone("ABC")
    assert_nil type.user_input_in_time_zone(" " * 129)

    offset = ::Time.zone.formatted_offset
    time_string = "2015-02-09T19:45:54#{offset}"

    assert_equal 19, type.user_input_in_time_zone(time_string).hour
    assert_equal offset, type.user_input_in_time_zone(time_string).formatted_offset
  end
end
```

**It is not absent — it was split and renamed.** `packages/activemodel/src/type/time.test.ts`
carries four tests whose names all begin "user input in time zone" but none of
which is the Rails name, so `parity:test` cannot match any of them:

- `:88` `it("user input in time zone wraps plain time in Time.zone")` — Eastern zone, plain `"14:30:00"`
- `:97` `it("user input in time zone answers a zoneless value when Time.zone is unset")`
- `:103` `it("user input in time zone returns null for null")` — this one DOES carry all four of Rails' `assert_nil` arms (`null`, `""`, `"ABC"`, `" ".repeat(129)`)
- `:110` `it("user input in time zone passthrough for ZonedDateTime")`

So most of the Rails body is covered, spread across renamed tests. What no
trails test covers is the **offset round-trip**: a full ISO string carrying the
zone's own `formatted_offset`, asserting both `.hour === 19` and that
`formatted_offset` comes back unchanged. trails' closest test passes a bare
`"14:30:00"` with no offset at all.

This is the rename prohibition in CLAUDE.md ("NEVER rename or reword test
names. Test names are how `parity:test` matches our tests to Rails tests")
having been broken four times in one file, and `parity:test` is correctly
reporting the consequence.

### The skip: `numericality_validation_test.rb`

`packages/activemodel/src/validations/numericality-validation.test.ts:434-447`
carries `it.skip("validates numericality with exponent number")` behind a
documented `PERMANENT-SKIP`: Ruby's `"10000000000000001".to_i` is an
arbitrary-precision Integer greater than `10_000_000_000_000_000`, while every
JS number is a double, so both sides round to `1e16` and the comparison is
(correctly, for a double) less-than-or-equal. The note says only a BigInt-typed
parse pipeline through Comparability could express it.

That reason is a genuine language shortcoming as written, and this story does
**not** require converging it. But `PERMANENT` is a strong claim and JS does
have `BigInt`, so the reason deserves one re-read: check whether the numericality
parse path could carry a BigInt for an integer-valued string without disturbing
the float arms. If it plainly cannot, leave the skip and say so in the PR — that
is a valid outcome. Do not reword the skip reason to sound better.

## Acceptance criteria

- `packages/activemodel/src/type/time.test.ts` gains
  `it("user input in time zone", ...)` — the Rails name **exactly** — mirroring
  `time_test.rb:24-38` line by line: `Time.use_zone("Pacific Time (US & Canada)")`,
  the four nil assertions, then the offset round-trip asserting `.hour === 19`
  and that `formattedOffset` matches the zone's.
- The four existing `user input in time zone ...` tests are trails-only extras.
  Move them to `type/time.trails.test.ts` rather than leaving them shadowing the
  Rails name in the mirrored file — that is where TS-only extras belong.
- **Do not rename any other test to make numbers move.** If a name looks wrong,
  the implementation is what changes.
- `parity:test` shows activemodel `type/time_test.rb` at Miss=0, and the package
  at **962/963** (the remaining 1 is the documented skip). State the before/after
  in the PR body.
- The numericality `PERMANENT-SKIP` is either converged with a BigInt-carrying
  parse path, or left exactly as-is with a one-line finding in the PR body
  saying why it cannot be. Its skip reason text is not reworded.
- `pnpm vitest run packages/activemodel/src/type/time.test.ts` (and the new
  `.trails.test.ts`) green.

## Notes

`Time.zone.formatted_offset` for "Pacific Time (US & Canada)" is DST-dependent,
and the Rails test derives `offset` from the zone at runtime rather than
hardcoding it — mirror that, do not inline `-08:00`, or the test breaks half
the year.
