---
title: "Reconcile activerecord's non-zero gate entry with the enrollment contract"
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

**#6997 enrolled `activerecord` in `GATED_PACKAGES` at `novel: 399`, which this
RFC's enrollment contract forbids.** The two changes landed the same day and did
not see each other.

This RFC states (README, §The enrollment contract):

> **Contract.** A package enrolls when `novel = 0` and `noCounterpartFiles = 0`.
> Its mark is `{ novel: 0, total: <measured>, noCounterpartFiles: 0,
allowed: <measured> }`, every dimension only-shrink.

The mark committed by #6997 is:

```json
"activerecord": { "novel": 399, "total": 1424 }
```

with `noCntrp: 383` measured. So the gated set now contains a package that
satisfies neither enrollment condition, and the invariant "everything in
`GATED_PACKAGES` has `novel = 0`" — which
`extra-surface-mark-dimensions` and the rest of this RFC's rollout may
reasonably assume — no longer holds.

### The two mechanisms are not the same thing

PR #6997 was not trying to enroll under this contract; it was freezing a
population so it cannot grow while RFC 0119's connection-adapter burndown runs.
Those are different tiers:

- **Freeze** — pin at the measured high-water mark, only-shrink, to stop growth
  during a burndown. What `activerecord` has now.
- **Enroll** — the door closes at `novel = 0` and stays there. What this RFC's
  contract describes, and what `arel` has.

Both are only-shrink and both use the same `GATED_PACKAGES` + mark mechanism,
which is why they collided. The gate cannot currently tell them apart, so
nothing stops a package being frozen at a high number and then read as
"enrolled".

### Why this needs deciding rather than quietly leaving

If a freeze tier is legitimate, this RFC's contract needs to say so explicitly,
name the two tiers, and state which dimensions apply to each — otherwise the
next enrollment PR either copies `activerecord` (freezing at a high number and
calling it enrolled) or reverts it. If a freeze tier is NOT legitimate,
`activerecord` should come out of `GATED_PACKAGES` until its `novel` reaches 0,
and RFC 0119 loses the ratchet its end condition now depends on.

Note the freeze is doing real work today: without it, the largest package's
extra surface was growing unchecked, which is the exact failure this RFC's
Summary attributes to the pre-0117 state.

## Acceptance criteria

- This RFC's §The enrollment contract resolves the conflict one way or the
  other: either it recognises a **freeze** tier distinct from **enrolled**, with
  its own stated dimensions and its own exit condition, or it does not and
  `activerecord` is removed from `GATED_PACKAGES`.
- If a freeze tier is recognised, the mark or an adjacent register records which
  tier each gated package is in, so `novel: 399` cannot be misread as enrolled.
  A frozen package must not be reported or documented as having passed
  enrollment.
- This RFC's §Summary is corrected: it currently reads "`parity:api:extra` gates
  exactly one package" and quotes `GATED_PACKAGES = ["arel"]`, both false since
  #6997.
- `scripts/api-compare/extra-surface-mark.ts`'s module comment is reconciled
  with whatever this RFC decides; it currently justifies the activerecord
  enrollment on burndown grounds without referencing this contract.
- Whichever way it goes, RFC 0119's §End condition is updated to match — it
  currently depends on the activerecord mark existing.

## Notes

Filed from the #6997 post-merge sweep. #6997 also added `unmarkedPackages`, a
guard that fails the gate when a `GATED_PACKAGES` entry has no committed mark —
relevant here because removing `activerecord` from the gate means removing BOTH
the name and its mark, not just one.
