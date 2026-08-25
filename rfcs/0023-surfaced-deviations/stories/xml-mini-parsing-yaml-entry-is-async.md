---
title: "XmlMini PARSING['yaml'] returns a Promise where Rails' proc table is uniformly sync"
status: draft
updated: 2026-08-13
rfc: "0023-surfaced-deviations"
cluster: null
packages:
  - "activesupport"
deps: []
deps-rfc: []
est-loc: 120
priority: null
pr: null
claim: null
assignee: null
blocked-by: null
closed-reason: null
---

## Context

`ActiveSupport::XmlMini::PARSING` (`vendor/rails/activesupport/lib/active_support/xml_mini.rb:83`)
is a uniform table of synchronous procs:

```ruby
"yaml" => Proc.new { |yaml| YAML.load(yaml) rescue yaml },
```

`packages/activesupport/src/xml-mini.ts` (landed in #6465) ports every entry
synchronously **except** `yaml`, which is `async` and returns a Promise:

```ts
yaml: async (yaml) => {
  try {
    const { parse: parseYaml } = await import("./yaml.js");
    ...
  } catch { return yaml; }
},
```

Callers therefore cannot treat the table uniformly — `PARSING[type](value)` is a
value for 13 keys and a Promise for one, which no Rails caller has to think
about, and which `Hash.from_xml`'s typecast pass will have to special-case when
it lands.

Why it shipped this way: `yaml` is an `optionalDependency` of
`packages/activesupport` and `xml-mini.ts` IS in the package's root graph
(exported from `index.ts`). A static `import ... from "yaml"` breaks the whole
`@blazetrails/activesupport` root import when the optional package is absent; a
static import of the existing `./yaml.js` shim drags its module-scope
`await import(...)` into the root graph, which reds the Website `iife`
service-worker build and the cjs test-compare build (see
`[[remove-top-level-await-from-activesupport-yaml]]` and
`[[configuration-file-static-yaml-import-is-an-eager-edge]]`, both closed, for
the two prior instances of that trap). The maintainer chose the call-time
`import()` over promoting `yaml` to a hard dependency.

## Converged shape

A synchronous `PARSING["yaml"]` matching the other 13 entries. The options, in
rough order of preference:

1. Resolve the parser once at a point that is already async and cache it, so the
   proc itself is sync — e.g. `XmlMini.parse` (already `Promise`-returning,
   `xml-mini.ts`) primes it before any typecast runs.
2. A zero-import slot (CLAUDE.md, "Call-time constant resolution") that
   `yaml.ts` populates, letting the proc read it at call time. Note the
   trade-off measured in #6465's review: a slot read must not silently degrade
   behaviour when unpopulated.
3. Promote `yaml` from `optionalDependencies` to `dependencies` and use a plain
   static import. Simplest and fully sync; costs every activesupport consumer
   the dependency.

Note that #6465 also removed the `deprecation.ts -> index.ts` barrel edge by
giving `error-reporter.ts` a module binding — the same "break the cycle at its
source rather than route around it" move may apply here.

## Acceptance criteria

- `PARSING["yaml"]` is synchronous and the table is uniform; no entry returns a
  Promise.
- `xml_mini_test.rb`'s `ParsingTest > yaml` still passes with the Rails
  assertions, including that a non-String input is handed back unchanged
  (Ruby's `YAML.load` raises on it and the `rescue` returns the input).
- The Website `build:sw` and the cjs test-compare build both still pass — verify
  with `pnpm --filter @blazetrails/website exec svelte-kit sync && pnpm --filter @blazetrails/website build:sw`, not just vitest.
- `pnpm parity:api` / `pnpm parity:test` deltas non-negative.
