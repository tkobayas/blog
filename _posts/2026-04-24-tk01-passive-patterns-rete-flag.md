---
layout: post
title: "Passive patterns — what `?/persons` becomes in Rete"
date: 2026-04-24
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [drlx, antlr, drools, passive, rete, grammar-addition]
---

#10 closed tonight. Three commits, 101 tests. The grammar change is one token; the IR gets one new field; the runtime wiring is one line. Most of the work was understanding what that one line does to Drools' Rete network.

## What `?` actually does

The DRLXXXX spec calls it "passive": `var p : ?/persons[age > 18]` is a pattern that "only propagates when prior data is pushed into it." Insert a person alone — nothing fires. Insert a matching location first, then the person — still nothing. Insert a location after you already have persons — the rule fires once for each matching person.

I wanted to know whether Drools actually models this. `Pattern.setPassive(boolean)` exists in `drools-base/src/main/java/org/drools/base/rule/Pattern.java:237` — a getter/setter on a boolean field. On its own, nothing.

The flag is consumed in `drools-core/src/main/java/org/drools/core/reteoo/BetaNode.java:172,272-274`:

```java
rightInputIsPassive = pattern.isPassive();
// ...
shouldFlush = memory.linkNode(this, reteEvaluator, !rightInputIsPassive) | shouldFlush;
// ...
shouldFlush = memory.setNodeDirty(this, reteEvaluator, !rightInputIsPassive) | shouldFlush;
```

That's the whole mechanism. When the right input of a BetaNode receives a new fact, the tuple still flows through the join and populates right memory. But `linkNode` and `setNodeDirty` — the calls that mark the network as needing re-evaluation — get their boolean argument flipped to `false` when the pattern is passive. The rule isn't scheduled to fire.

The match still exists. It's just sleeping. When the reactive side later pushes a fact through, the match wakes up with everything else.

## Scope: oopath only

The spec notes `?` can also apply to group CEs — `?and(...)`, `?or(...)` — but there's no canonical example. Three scope options went on the table: oopath-only, oopath + group CEs, or bound-only. I picked oopath-only. Attaching passivity to `oopathExpression` means bound (`var p : ?/persons[...]`) and bare (`and(/locations, ?/persons)`) both get it with the same grammar line. Group-CE passive is a different semantic question — it pushes passivity down to children — and deserves its own spec pass.

## The grammar change is one token

```antlr
oopathExpression
    : QUESTION? '/' oopathRoot ('/' oopathChunk)*
    ;
```

`QUESTION` was already defined in `JavaLexer.g4:160`. One line in `DrlxParser.g4`. Everything downstream comes free: every site that references `oopathExpression` — top-level `rulePattern`, `groupChild`, `notElement`, `existsElement` — now accepts `?`.

## Last session's lesson paid off

The last session's factor-out of `boundOopath` fanned out to three unplanned readers — `DrlxToRuleAstVisitor`, `DrlxToJavaParserVisitor`, `DrlxParserTest`. Mid-plan panic ensued.

So this time Task 1 was a read-only grep: find every reader of `OopathExpressionContext`, confirm each navigates by named accessor (`.oopathRoot()`, `.oopathChunk()`) rather than by token position. Five files found. All clean. Adding an optional `QUESTION?` at token index 0 can't affect named-accessor lookups.

No mid-plan surprises. The audit was cheap — just a grep — and it paid off.

## The one cascade fix

`NotTest.protoRoundTrip_withNot` constructs `PatternIR` directly with the full positional arg list. Adding a 7th field to the record (`boolean passive`) broke that call site at compile. The fix was `, false` appended.

Finding every such site upfront is `grep 'new PatternIR('`. Three main sites (the two visitor helpers and the proto deserializer), two test sites — one of which was this one. All mechanical.

## Visibility relaxation for backward-compat

The round-trip test had to cover two things: that `passive=true` on an IR survives serialize → deserialize, and that an old proto without field 7 deserializes to `passive=false`. The first can go through the public `save`/`load` path; the second can't — `save` always writes with the current schema, so you can't produce a proto missing field 7 through it.

`toProtoLhs` and `fromProtoLhs` in `DrlxRuleAstParseResult.java` were `private static`. I relaxed them to package-private so the test (in the same package) could call them directly. Still not public API; still scoped to one package. The alternative was routing the entire test through a `CompilationUnitIR` round-trip — three times the boilerplate for no coverage difference. The commit message says why.

## The runtime wiring

`DrlxRuleAstRuntimeBuilder.buildPattern` constructs a Drools `Pattern` from a `PatternIR`. The entire wiring:

```java
Pattern pattern = new Pattern(lambdaCompiler.nextPatternId(), 0, 0,
                              objectType, parseResult.bindName(), false);
pattern.setPassive(parseResult.passive());   // the one line
pattern.setSource(new EntryPointId(parseResult.entryPoint()));
```

One line. The behavioural test — insert `Location("paris")`, then insert `Person(adult)`, assert `fireAllRules() == 0` — failed pre-fix returning `1`, passed post-fix returning `0`. That's the line taking effect end-to-end.

## What landed

Three commits on `main` (`3cffa03..195fe39`), 94 → 101 tests:

- Grammar: `QUESTION?` prefix on `oopathExpression` + 2 parse-tree tests
- IR: `boolean passive` on `PatternIR` + `bool passive = 7` on `PatternParseResult` + 2 proto round-trip tests
- Runtime: `pattern.setPassive(parseResult.passive())` + 3 behavioural tests

DRLXXXX §"Passive elements" gets canonical oopath-form coverage. Group-CE passive is still open.
