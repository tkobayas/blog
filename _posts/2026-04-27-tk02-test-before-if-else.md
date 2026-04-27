---
layout: post
title: "test before if/else"
date: 2026-04-27
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [test, eval, restructure, drools]
---

# test before if/else

## The bridge that needed its own ticket

Started the session intending to land #12 (if/else branching, Form A) — DRLXXXX
has a clean syntax for it (`if (cond) { … } else { … }`) and the runtime story
looked like pure desugaring to existing `or` + `and` infrastructure. The plan
was straightforward.

The plan also included inventing an internal `EvalIR` to carry the if-condition
into the IR, plus a small bridge from MVEL3-compiled boolean lambdas to
drools-base's `EvalExpression`. That bridge was Task 4 of the original 14-task
plan, and Claude flagged it as the highest-risk piece. I asked for a deep dive
into the drools API before committing — an hour of reading beats a day of
debugging.

The investigation surfaced a few API details that mattered. `EvalExpression.evaluate(...)`
actually takes `BaseTuple, Declaration[], ValueResolver, Object` — four
arguments, not the three Claude initially assumed. DRLX already has an
`EvaluatorSink` interface; every lambda-carrying class implements it, and
`compileBatch` iterates pending lambdas calling `target.bindEvaluator(resolved)`.
No refactor needed for deferred resolution.

But the most consequential find was structural. The bridging code (proto, IR,
runtime mapping, lambda compiler entry, EvalExpression impl) was *all*
infrastructure for a feature DRLXXXX already specifies independently: `test`.
DRLXXXX § "test" defines `test <expression>` as legacy `eval`'s replacement,
intended as a user-facing `ruleItem`. The original plan shipped the IR
internally for if/else and added the user-facing grammar in a sibling ticket
later.

I asked Claude: shouldn't we do the sibling first?

## test first, if/else second

The reasoning was simple. If `test` ships standalone, the riskiest bridging
code lives in its own ticket — exercised by `test`-specific tests, where bugs
surface in isolation rather than conflated with if/else desugar bugs. After
`test` lands, #12 becomes pure grammar + visitor desugar onto the existing
infrastructure. Each ticket gets a cleaner diff and a cleaner test surface.

Claude agreed. We rewrote the #12 spec to depend on #23, expanded #23's scope
from "grammar only" to "full `test` implementation," wrote a fresh #23 plan
and a slimmer #12 plan, and updated the GitHub issue descriptions to reflect
the new dependency.

## Mirroring DrlxLambdaBetaConstraint

The work landed in nine commits. Most surprises were favourable. `EvaluatorSink`
already existed, so plan-Task-4's "PendingLambda generalization" became a
no-op observation. `DrlxLambdaBetaConstraint` had the exact pattern Claude
needed — `MVEL.<Object>map(mvelDeclarations).<Boolean>out(Boolean.class).expression(expr)`
for the compile call, `Map<String,Object>` input for evaluation,
`tuple.get(decl).getObject()` for fact dereferencing. Mirroring that file
produced a working `DrlxEvalExpression` and `createEvalExpression` on the first
compile.

The programmatic E2E test — building a `RuleIR` by hand with
`[PatternIR(p), EvalIR("p.age > 30")]` and firing it through a real `KieSession`
— passed on the first run:

```
18:16:03 INFO  o.m.MVELBatchCompiler - Batch-compiling and persisting 2 lambda sources
Person{name='Alice', age=40, address=null}
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
```

Alice (40) matched, Bob (25) was filtered out. The batch compiler picked up the
lambda, the runtime built an `EvalCondition`, rete invoked our
`evaluate(BaseTuple, Declaration[], ValueResolver, Object)`, the boolean came
back true.

## ALWAYS mode kills the limitation

The spec carried a property-reactivity limitation note: drools-base's
`EvalCondition.isPatternScopeDelimiter() == true`, so a `test` guard referencing
an outer pattern's property wouldn't extend that pattern's listened-property
mask. Users would need explicit watch lists. Or so we thought.

The runtime test with a property update — insert Alice with age 25, fire (no
match), bump age to 40, update fact handle, fire again — printed the
consequence. The rule re-fired. No explicit watch list. Claude went looking
for why.

The reason was in `DrlxRuleAstRuntimeBuilder.java:105`:

```java
PropertyReactivityUtil.computeMaskOfChunks(
        cls, PropertySpecificOption.ALWAYS);
```

DRLX configures patterns with `ALWAYS` mode — every property is watched by
default. The scope-delimiter contract on `EvalCondition` only matters in
`ALLOWED` mode, where `@PropertyReactive` annotations restrict watches. In
`ALWAYS` mode, the pattern's mask covers everything, regardless of where the
property is referenced.

The "limitation" was theoretical. We removed it from the spec, locked the
actual behaviour into a regression test (`test_refiresOnPropertyUpdate`), and
noted that if DRLX ever switches to `ALLOWED` mode, this assumption becomes
load-bearing again.

## End state

#23 closed. 125 tests passing in DRLX (was 111). #12 picks up tomorrow on a
foundation that's verified end-to-end — the if/else plan is now ten tasks of
grammar and desugar, no risky bridging code remaining.
