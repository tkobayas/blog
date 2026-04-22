---
layout: post
title: "exists landed — with two build detours nobody ordered"
date: 2026-04-22
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [drlx, antlr, drools, group-elements, exists, protobuf, maven]
---

#9 — `exists /pattern` and `exists(/a, /b)` — landed this afternoon,
mirroring the `not` landing from the morning. Seven commits, 73 tests
passing. The feature itself was as boring as the spec promised: same
grammar shape, same IR, same AND-wrap fix from the morning that I'd
flagged in the previous entry as "probably applies to EXISTS too."
It did.

What wasn't boring was the two build-system detours.

## Detour one: the proto file I couldn't edit

I wrote the spec saying "Proto (`drlx_rule_ast.proto`): additive
change — new enum value `GROUP_ELEMENT_KIND_EXISTS = 2`." Easy. Then
I went to plan it and noticed `protoc` isn't installed on this
machine. The generated `DrlxRuleAstProto.java` is checked into the
repo with `// DO NOT EDIT` at the top — 8,000 lines of protobuf
runtime, including a serialised binary descriptor that encodes the
proto schema as octal-escaped bytes. Hand-editing the descriptor bytes
to add one enum value means recomputing a chain of varint lengths.
Wrong.

Claude flagged this. Three options — install protoc, hand-edit the
bytes, or defer proto support. I suggested a fourth:
`protobuf-maven-plugin`. Same shape as the `antlr4-maven-plugin`
already in the pom — downloads the protoc binary from Maven Central,
regenerates on every compile, no system dependency. The generated
`.java` lives in `target/generated-sources/protobuf/java/` and gets
fed to `build-helper-maven-plugin` alongside the ANTLR output.

The migration was three edits:
- Parent pom: plugin version properties, `os-maven-plugin` as a
  `<extension>` (it's the thing that populates
  `${os.detected.classifier}`).
- Child pom: new plugin block, extend the source list.
- Delete the checked-in `DrlxRuleAstProto.java`.

Build green, 68 tests still passing, including the NOT proto
round-trip test — proof the regenerated class is byte-equivalent to
the old checked-in one. Committed as an infra refactor with `Refs #9`
before the EXISTS work began.

That made the actual EXISTS proto change one line:

```proto
GROUP_ELEMENT_KIND_EXISTS = 2;
```

Which is what the spec said in the first place.

## Detour two: a tokens file from 2023

Atomic plumbing commit: lexer, grammar, IR, visitor, runtime switch,
proto mapping, frozen visitor, tolerant visitor. Nine files, one
commit. Built. Ran the RED test. Got this:

```
DRLX parse error at line 1:0 - mismatched input 'package'
    expecting {'unit', 'with', 'import', 'package', 'module', ...}
```

`'package'` is literally in the expected set. The parser is rejecting
`package` while saying it expects `package`. I read it three times
before I trusted my eyes.

Ran `NotTest` — broken too, four failures, same error. So I'd broken
something beyond the new EXISTS path. The change that could explain
this had to be the lexer. But all I added was `EXISTS : 'exists';`.

I grep'd the regenerated parser for the token IDs:

```
UNIT=1, RULE=2, NOT=3, IN=4, CONTAINS=5, ...  PACKAGE=51
```

Then the regenerated lexer:

```
UNIT=1, RULE=2, NOT=3, EXISTS=4, IN=5, ...  PACKAGE=52
```

Parser says `_la == PACKAGE` where `PACKAGE = 51`. Lexer emits the
`package` input as token 52. They disagree by one — exactly the slot
EXISTS took.

The parser has `options { tokenVocab = DrlxLexer; }`. It learns token
IDs from a `.tokens` file. ANTLR's `antlr4-maven-plugin` is configured
with `<libDirectory>src/main/antlr4/org/drools/drlx/parser</libDirectory>`
— that's where it looks for imported grammars and tokens vocab files.
And at exactly that path, sitting there ignored by git, was a stale
`DrlxLexer.tokens` from a prior IDE run, pre-dating EXISTS. The plugin
grabbed it in preference to the one it had just regenerated into
`target/`. Parser built against old IDs, lexer generated new ones.
`package` matches neither.

```bash
rm src/main/antlr4/org/drools/drlx/parser/DrlxLexer.tokens
mvn -pl drlx-parser-core -am clean install
# → 69 tests passing
```

The file is in `.gitignore` — never been committed, never will be.
But IDEs drop it there, prior runs drop it there, and once it's
present, every regen quietly uses the stale version. The error
message told me the truth (`mismatched input 'package'`) and then
hid it in the expected-tokens list — which is generated from the
parser's internal table, not from what the lexer is actually
producing. When lexer IDs and parser IDs disagree, the error message
reflects the parser's belief, not the reality.

## What actually shipped

Feature-side, the entry is short: five tests (Q2-B reduced mirror
from the spec), one atomic plumbing commit, no surprises in the
code itself. The AND-wrap generalisation from morning NOT turned
out to be literally one OR-clause extension:

```java
if ((group.kind() == GroupElementIR.Kind.NOT
        || group.kind() == GroupElementIR.Kind.EXISTS)
        && group.children().size() > 1) {
    // synthetic AND wrapper
}
```

The visitor factoring I pitched in brainstorming — extract
`buildGroupElementFromOopaths` instead of writing a twin
`buildExistsElement` — landed clean. Both methods are one-liners
now. When AND/OR land as part of #11, they'll be one-liners too.

The bigger arc of today: two issues closed (#8 and #9), 35 commits
pushed to main across the day, 73 tests, both multi-element forms of
NOT and EXISTS working end-to-end. The morning and afternoon
landings took similar wall-clock time despite the afternoon's
nominal feature being derivative. Build-system yak-shaving is not
nothing.

## What I'd do differently

The proto-plugin migration was the right call, and doing it as a
separate commit before the EXISTS feature was the right shape.
What I'd change: flag the "is this build-system prerequisite
installed" question earlier — before the plan is approved. Claude
caught the protoc issue during planning; it almost slipped past the
brainstorming step entirely. Earlier is better — I have local context
(like "we can use protobuf-maven-plugin") that would have shaped the
plan from the start instead of mid-flight.

The stale `.tokens` gotcha is the kind of thing you only catch once.
Next time I touch the lexer I'll blow away any loose `.tokens` files
in `libDirectory` before the first build — two-line prophylactic,
saves the next half-hour.

## Where next

#11 — `and(...)` and `or(...)` group CEs. The helper is ready. The
IR slots are reserved. But AND and OR aren't single-child in Drools,
so the AND-wrap logic needs a different generalisation — or probably
no wrap at all, since Drools' AND/OR accept N children natively. I'll
check before writing the spec this time. Again.
