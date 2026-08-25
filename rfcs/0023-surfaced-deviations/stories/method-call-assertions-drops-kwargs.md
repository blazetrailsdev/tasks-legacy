---
title: "assert_called_with / expect_called_with drop Ruby kwargs"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
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

`assert_called_with` and `expect_called_with` take Ruby kwargs and forward them
to the mock; the trails port drops them entirely.

Rails
(`vendor/rails/activesupport/lib/active_support/testing/method_call_assertions.rb:20-34`):

```ruby
def assert_called_with(object, method_name, args, returns: false, **kwargs, &block)
  mock = Minitest::Mock.new
  expect_called_with(mock, args, returns: returns, **kwargs)
  object.stub(method_name, mock, &block)
  assert_mock(mock)
end

def expect_called_with(mock, args, returns: false, **kwargs)
  mock.expect(:call, returns, args, **kwargs)
end
```

`Minitest::Mock#expect` (minitest 5.20.0, `lib/minitest/mock.rb:98-124`) takes
`args` (positional) and `kwargs` as SEPARATE expectations, and `#verify` /
`__call` compare both — a call whose keyword arguments differ fails even when
the positional args match.

trails (`packages/activesupport/src/testing/method-call-assertions.ts`, after
PR #6650):

```ts
export function assertCalledWith<T extends object>(
  object: T,
  methodName: keyof T & string,
  args: unknown[],
  { returns = false }: { returns?: unknown } = {},
  block?: () => void,
): void;
export function expectCalledWith(
  mock: Mock,
  args: unknown[],
  { returns = false }: { returns?: unknown } = {},
): void;
```

There is no `kwargs` parameter on either, `Mock` has no `kwargs` field, and
`assertMock` compares only the positional list. Rails' own
`cache_store_setting_test.rb` uses the kwargs arm
(`assert_called_with(Dalli::Client, :new, [%w[localhost], { compress: false }])`
— there the hash is positional, but `activerecord` and `actionpack` call sites
pass real kwargs), so a port of those tests cannot express what Rails asserts.

This is not a language shortcoming: a Ruby kwargs hash is an ordinary trailing
object in the trails idiom, so the parameter can be carried through.

## Converged shape

Add the `kwargs` parameter to both functions in Rails' position and forward it,
give `Mock` an `expectedKwargs` field alongside `expected`, and teach
`assertMock` to compare it — a mismatch raising the same
`MockExpectationError` the positional path raises.

## Acceptance criteria

- `assertCalledWith` / `expectCalledWith` carry kwargs through to the mock and
  `assertMock` compares them.
- A trails test covers a kwargs mismatch failing while the positional args
  match (the case that silently passes today).
- No new rows in any baseline; `pnpm parity:api:extra --package activesupport`
  still reports 0 novel.
