---
title: "locate_many_signed passes uri strings where Rails passes parsed SignedGlobalIDs"
status: draft
updated: 2026-08-11
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "globalid"
deps: []
deps-rfc: []
est-loc: 80
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`GlobalID::Locator.locate_many_signed` parses each sgid and hands the parsed
**SignedGlobalID objects** to `locate_many`
(`vendor/globalid/lib/global_id/locator.rb:106-108`):

```ruby
def locate_many_signed(sgids, options = {})
  locate_many sgids.collect { |sgid| SignedGlobalID.parse(sgid, options.slice(:for)) }.compact, options
end
```

The port (`packages/globalid/src/locator.ts:308-320`) collects `parsed.uri`
strings instead:

```ts
const uris: string[] = [];
for (const s of sgids) {
  const parsed = SignedGlobalID.parse(String(s), { for: options.for, verifier: options.verifier });
  if (parsed) uris.push(parsed.uri);
}
return Locator.locateMany(uris, options);
```

Two divergences ride along with the type change: Rails slices only `:for` out of
the options for the parse (the port also forwards `verifier`), and Rails'
`.compact` drops parse failures where the port's `if (parsed)` is an equivalent
but differently-shaped guard. Surfaced as an RFC 0095 `naming` call-argument row
(`locate_many` ruby `ref:compact` vs ts `ref:uris`); PR #6353 left it as an
invented conversion rather than a rename.

## Converged shape

Collect the parsed SignedGlobalID objects (not their `uri` strings) and pass
that array to `locateMany`, parsing with only the `for` option as
`options.slice(:for)` does. Check `locateMany` / `parseAllowed` accept a
`GlobalID`-like instance — `GlobalID.parse` already returns its argument
unchanged when handed a GlobalID (`global_id.rb:25`), so the object path should
be the one Rails relies on.

## Acceptance criteria

1. `locateManySigned` passes the parsed SignedGlobalID array to `locateMany`,
   matching locator.rb:107.
2. The parse receives only the `for` option, per `options.slice(:for)`.
3. The `locate_many` `naming` row is gone from
   `pnpm parity:api:calls:args:report`, not baselined.
