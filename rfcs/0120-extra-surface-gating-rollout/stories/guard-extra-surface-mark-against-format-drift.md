---
title: "extra-surface-mark.json has no drift or format check"
status: draft
updated: 2026-08-24
rfc: "0120-extra-surface-gating-rollout"
cluster: null
packages: []
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

`extra-surface-mark.json` is the only ratchet register in `scripts/api-compare/`
with no drift check.

The CI step "Ratchet baseline reseed drift" (`.github/workflows/ci.yml:1489`,
RFC 0083) regenerates and diffs two registers:

```sh
baseline='scripts/api-compare/call-mismatches-exclude'
mark='scripts/api-compare/call-mismatches-unreviewed'
```

`scripts/api-compare/extra-surface-mark.json` is not among them. The extra-surface
ratchet only ever **reads** that file (`loadMarks`), so exactly the hole RFC 0083
identifies for the call baselines applies here too: a change can leave the mark
stale or malformed and still pass the gate.

Surfaced concretely in #6997. Seeding the `activerecord` entry required
hand-writing JSON, because `tightened()` skips a package with no existing mark
(`extra-surface-mark.ts`, the `if (!mark || !now) continue` arm), so
`--tighten` cannot seed a new package and there is no reseed command by design.
The hand-written file was verified byte-identical to `writeMarks` output by a
`--tighten` round-trip (same md5, `narrowed 0 dimension(s)`) — but that check was
manual and ad hoc, and nothing in CI would have caught it had it been wrong.

The format contract is narrow enough to be worth pinning: `writeMarks` sorts
keys and writes `serializeBaseline(...)`, which is
`JSON.stringify(value, null, 2) + "\n"` (`baseline-json.ts:26-28`). A mark file
with unsorted keys, different indentation, or a missing trailing newline would
produce a spurious diff on the next legitimate `--tighten` and be blamed on that
PR.

### Two candidate shapes

1. **Extend the existing CI step** to regenerate the mark and diff it, matching
   how the call baselines are handled. Needs a write path that does not lower
   the mark as a side effect — `--tighten` narrows, so it cannot be used as a
   no-op regenerate.
2. **A format-only assertion** in `extra-surface-mark.test.ts`: read the
   committed file, round-trip it through the same serializer, assert byte
   equality. Cheaper, catches the realistic failure (hand-edited mark), and
   needs no CI change.

(2) is probably right — the call-baseline drift check exists because a reseed
_rewrites_ those registers, whereas this mark is small and only ever moves down.

## Acceptance criteria

- A committed `extra-surface-mark.json` that is not byte-identical to what
  `writeMarks` would produce for its own contents fails — in CI or in the unit
  test, whichever shape is chosen.
- The check covers key ordering, indentation and the trailing newline, since
  those are what `serializeBaseline` fixes and what a hand-edit gets wrong.
- Adding a package to the mark by hand (the #6997 path) is either made
  unnecessary by a seed command, or is covered by the check so a malformed seed
  cannot merge.
- No reseed command that can _raise_ a mark is introduced — the only-shrink
  contract is the point, and `parity:api:extra:tighten` stays the sole writer.
