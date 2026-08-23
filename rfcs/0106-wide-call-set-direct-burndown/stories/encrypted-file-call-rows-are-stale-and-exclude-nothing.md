---
title: "encrypted-file.json's last 2 call rows are stale — the artifact flags no calls in the file"
status: ready
updated: 2026-08-23
rfc: "0106-wide-call-set-direct-burndown"
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

`scripts/api-compare/call-mismatches-exclude/activesupport/encrypted-file.json`
still carries 2 rows after RFC 0106 wave 5g (#6906) migrated the other 5:

    encryptor | new     (encrypted_file.rb:113 `MessageEncryptor.new([key].pack("H*"), ...)`)
    writing   | chomp   (encrypted_file.rb:89  `content_path.basename.to_s.chomp(".enc")`)

Both are **stale**: `pnpm parity:api:build --package activesupport --file
encrypted-file.ts --dry-run` reports 0 rows would migrate and emits no
`no body-bearing declaration` warning, and the regenerated artifact
(`scripts/api-compare/output/call-mismatches.json`) carries **zero** flagged
rows for `encrypted-file.ts`. So neither row corresponds to a call the
comparator still reports missing — they are excluding nothing.

This is unlike the sibling story
`cache-ts-call-rows-have-no-body-bearing-declaration`, where the migrator does
warn and the rows are still live. Here there is nothing to justify and nothing
to tag; the rows are dead weight in an only-shrink baseline.

## Converged shape

Confirm against a forced regen (`API_COMPARE_FORCE=1 pnpm parity:api --calls`)
that no call in `encrypted-file.ts` is flagged, then delete both rows by hand —
only-shrink, do NOT `--write`/reseed (a reseed rewrites the whole exclude tree).
Delete `activesupport/encrypted-file.json` entirely once empty rather than
committing `[]`.

Deleting the rows lowers the source's unreviewed count below its committed
high-water mark, so the gate will then report a STALE mark. Narrow it with:

    pnpm parity:api:calls:tighten activesupport/encrypted-file.json

If either row turns out to be live after all (the comparator reports it under a
different `rubyName` key), converge that call instead — Ruby `String#chomp` and
`MessageEncryptor.new` both have trails ports — or migrate it to a
`@missingRailsCall` receipt with a `PERMANENT` / `CONVERGEABLE (story …)`
permanence claim.

## Acceptance criteria

- [ ] Both rows are gone and the shard file deleted, not committed as `[]`.
- [ ] The mark is tightened, not reseeded.
- [ ] `pnpm parity:api:calls` and `pnpm parity:api:calls:args` green.
