---
title: "BlockLocator#locateMany filters nils where Rails preserves positional alignment"
status: draft
updated: 2026-08-17
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "globalid"
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

`GlobalID::Locator::BlockLocator#locate_many` maps every gid through `locate`
and keeps the results as-is, nils included
(`vendor/globalid/lib/global_id/locator.rb:241-243`):

```ruby
def locate_many(gids, options = {})
  gids.map { |gid| locate(gid, options) }
end
```

The result therefore stays **positionally aligned** with the input: a gid the
block couldn't resolve leaves a `nil` in its slot.

trails filters the nulls out
(`packages/globalid/src/locator.ts:226-241`), so the returned array is shorter
than the input and positional alignment is lost. The JSDoc at that site
currently records this as a deliberate divergence ("We filter nulls to stay
consistent with `BaseLocator#locateMany` (which uses `filter_map` in Rails). If
positional alignment matters, callers should call `locate` per GID directly.").

Per CLAUDE.md that note is a burndown ledger entry, not permission — it is
debt, and the justification does not hold up: `BaseLocator#locate_many` using
`filter_map` is exactly why `BlockLocator#locate_many` using plain `map` is
_deliberate_ on Rails' side. The two locators have different contracts and the
port collapsed them into one.

## Converged shape

Mirror the Ruby: map, keep nils, drop the filtering and the JSDoc divergence
note.

```ts
async locateMany(gids: GlobalID[], options: LocateOptions = {}): Promise<unknown[]> {
  const results: unknown[] = [];
  for (const gid of gids) results.push(await this.locate(gid, options));
  return results;
}
```

The sequential loop is retained deliberately — it is already justified at the
call site (Ruby's `map` is single-threaded, and a block touching global state
would behave unpredictably under `Promise.all`).

Check callers before flipping: `Locator.locateMany` dispatches to whichever
locator is registered, so a caller that assumed a dense array from a
BlockLocator will now see nils.

## Acceptance criteria

- `BlockLocator#locateMany` returns one entry per input gid, nils included,
  matching `locator.rb:241-243`.
- The "Divergence from Rails" JSDoc block at `locator.ts:226` is deleted, not
  reworded.
- Callers that relied on the dense result are updated.
