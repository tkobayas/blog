---
layout: post
title: "epic #26 issue triage — 18 created, 3 already done"
date: 2026-04-30
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [planning, issues, drlxxxx-spec]
---

# epic #26 issue triage — 18 created, 3 already done

I sat down with the DRLXXXX.md spec and created sub-issues for epic #26
(DrlxCompiler enhancement round 2). The spec is dense — over 1,200 lines
covering everything from inline casts to CEP sequencing operators. Claude
and I walked through it section by section, extracting features that the
compiler doesn't handle yet.

We filed 18 issues (#27–#44), covering: accumulate, groupBy, queries,
windows, match/switch, edge-triggered actions, passive patterns, inline
cast, positional syntax, pluggable operators, DataStore CRUD, multiple
do blocks, property accessors, with-blocks, list/map access, coercion,
drain, and ExistenceDriven mode.

Then came the audit. Claude checked each issue against the actual codebase
— grep for keywords, check grammar rules, look for test classes. Three
issues turned out to be already fully implemented: inline cast (#27 —
`InlineCastTest.java` exists with full coverage), positional syntax (#28 —
`PositionalTest.java` with 7 tests), and passive patterns (#29 —
`PassivePatternTest.java` with 3 end-to-end tests). Closed all three.

Three more needed narrowing: property accessors (#33 — read-side already
works via MVEL3, only consequence-side setters are the gap), list/map
access (#35 — MVEL3 grammar handles it, just needs integration tests),
and expression inline cast (#36 — `#` cast parsing works, the missing
piece is the coercion framework for units and dates). Revised those.

The remaining 12 are genuinely unimplemented. The high-priority ones —
accumulate, groupBy, queries, DataStore operations — are the backbone
of real-world rule authoring. The medium tier (match/switch, windows,
edge actions, multiple do blocks) adds expressiveness. The low tier
(drain, with-blocks, pluggable operators, coercion) is specialized.

The lesson: always audit before filing. Three of eighteen issues would
have been wasted work if someone picked them up without checking.
