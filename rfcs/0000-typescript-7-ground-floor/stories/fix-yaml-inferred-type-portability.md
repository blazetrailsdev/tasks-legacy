---
title: "Fix TS2883 in activesupport/yaml.ts — yaml.d.ts is silently not emitting"
status: ready
updated: 2026-08-25
rfc: "0000-typescript-7-ground-floor"
cluster: build-infra
packages: ["activesupport"]
deps: []
deps-rfc: []
est-loc: 15
priority: 1
pr: null
claim: null
assignee: null
blocked-by: null
---

## Context

`packages/activesupport/src/yaml.ts:23` ends with:

```ts
export const { parse, stringify } = yaml;
```

where `yaml` is a lazy accessor whose fallback is
`{ parse: missing, stringify: missing } as unknown as typeof import("yaml")`.

Under `typescript@7.1.0-dev.20260825.1` this produces:

```text
activesupport/src/yaml.ts(23,14): error TS2883: The inferred type of
  '{ parse, stringify }' cannot be named without a reference to
  '.../node_modules/yaml/dist/parse/cst' from '../node_modules/yaml/dist/parse/cst.js'.
  This is likely not portable. A type annotation is necessary.
```

`TS2883` is a stricter portability check on the 7.1 line. **The consequence is
not just a diagnostic — `activesupport/dist/yaml.d.ts` never emits.** Verified:
after a full 7.1 `tsc --build`, `packages/activesupport/dist/` contains
`yaml.js` and `yaml.js.map` and no `yaml.d.ts`.

That single failure cascades into **7 further diagnostics** across three
packages, which is 8 of the 10 in the whole 18-project build:

- 5× `TS7016` "Could not find a declaration file for module
  `@blazetrails/activesupport/yaml`" — in `actionview/src/helpers/debug-helper.ts`
  (+ its test), `activemodel/src/attribute-set/codecs/yaml.ts`,
  `activerecord/src/coders/yaml-column.ts`,
  `activerecord/src/adapters/postgresql/hstore.test.ts`.
- 2× `TS7006` implicit `any` on `_key` / `value` in
  `activemodel/src/attribute-set/codecs/yaml.ts:11`, flowing from that untyped
  import.

**This is worth fixing independently of TypeScript 7.** The declaration file for
a published subpath is not being emitted, and 5.9.3 hides it: 5.9.3 emits a
`yaml.d.ts` that embeds a non-portable reference into a `node_modules` path,
which is exactly what `TS2883` exists to catch. Every consumer of
`@blazetrails/activesupport/yaml` is currently typed by accident.

## Acceptance criteria

- [ ] `packages/activesupport/src/yaml.ts` produces no `TS2883` under the
      pinned 7.1 compiler.
- [ ] `packages/activesupport/dist/yaml.d.ts` **exists** after a build, and its
      exported types for `parse` / `stringify` are the ones consumers expect —
      not `any`.
- [ ] The 7 downstream diagnostics listed above are gone, with no
      `// @ts-expect-error`, `any`, or `eslint-disable` added at the consumer
      sites — they should be fixed by the declaration existing.
- [ ] `pnpm build` and `pnpm typecheck` stay green on the repo's current
      `typescript@5.9.3`, and the emitted `yaml.d.ts` under 5.9.3 no longer
      embeds a `node_modules/yaml/dist/**` path reference.

## Definition of done

Adding `// @ts-expect-error` at the `TS2883` site does not close this story —
that keeps `yaml.d.ts` from emitting, which is the actual defect. The remedy the
diagnostic names is an explicit type annotation.

## Verification

```bash
pnpm build && ls packages/activesupport/dist/yaml.d.ts
grep -c "node_modules" packages/activesupport/dist/yaml.d.ts   # expect 0
# and under the 7.1 pin:
node /path/to/ts71/bin/tsc --build   # expect: only the TS4094 pair remains
```
