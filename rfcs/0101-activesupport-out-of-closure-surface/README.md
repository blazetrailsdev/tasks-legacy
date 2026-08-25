---
rfc: "0101-activesupport-out-of-closure-surface"
title: "activesupport out-of-closure surface: cache stores and XmlMini"
status: postponed
created: 2026-08-12
updated: 2026-08-25
owner: "@deanmarano"
packages:
  - "activesupport"
clusters: []
priority: 3
---

# activesupport out-of-closure surface: cache stores and XmlMini

## Problem

`pnpm parity:api` puts activesupport at 55.2% (1118/2026, main 707c3975b). RFC
0098 owns the 518 members inside the `require "active_support/…"` closure of
activerecord + activemodel. The remainder — the members AR never loads — was
explicitly excluded from RFC 0072 (`activesupport-out-of-closure-unported-entries`)
and has had no home since.

Two clusters in that remainder are already triaged to story granularity, from the
`triage-partially-ported-out-of-closure-activesupport-residue` PR, and are what
this RFC carries:

1. **Cache stores.** `ActiveSupport::Cache`'s coder/serializer layer, the
   `read_serialized_entry` / `write_serialized_entry` half of every store's entry
   hooks, and `MemoryStore::DupCoder`'s `dump_compressed` / `dump_value` /
   `load_value`. trails ports only the Entry half of each store and shims
   `DupCoder` with `structuredClone`.
2. **XmlMini.** The backend and parsing half is unported, and the NokogiriSAX
   hash builder sits inside a function body where Rails has it at module scope.

## Scope

The three cache stories and two XmlMini stories moved here from RFC 0072 when
0072 closed. Nothing else in the out-of-closure remainder is filed yet; new
out-of-closure activesupport work belongs here rather than in 0098.

**Correction (2026-08-21).** "XmlMini is out-of-closure" stays true of
`xml_mini.rb` itself, but four members that route through it were scored against
the AR-closure gap and have been ported under RFC 0098 (trails#6818) rather than
left waiting on this postponed RFC: `Hash#to_xml`, `Hash.from_xml`,
`Hash.from_trusted_xml`, `Array#to_xml`, plus the XmlMini surface they reach
(`instruct!` / `target!`, the builder indent width, and `XMLConverter`).

Only `Array#to_xml` is in the require-closure on its own
(`activemodel/lib/active_model/errors.rb:3`); the three `Hash` members are
scored in-closure by file attribution, because `parity:api` anchors
`hash-utils.ts` on the in-closure `core_ext/array/extract_options.rb`. Neither
`core_ext/hash/conversions.rb` nor `xml_mini.rb` appears in
`output/ar-closure.json`. See 0098's changelog entry of the same date for the
full walk and the boundary.

What remains this RFC's: the rest of `xml_mini.rb` — backend selection and the
parsing half — and NokogiriSAX's module-scope hash builder.

Explicitly NOT in scope: any member inside the AR closure (that is RFC 0098), and
a general "port all of activesupport" campaign — this RFC covers the two clusters
whose gaps are already triaged.

## Done means

The five stories land, and the cache and XmlMini files report 0 missing members
in `pnpm parity:api` (or carry SKIP_GROUPS reasons).
