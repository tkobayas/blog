---
layout: post
title: "#39 — accumulate v1, and what sum() actually returns"
date: 2026-05-15
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [drlx, drools, accumulate, sealed-interfaces, mvel3, java]
---

# #39 — accumulate v1, and what sum() actually returns

Two sessions, eleven plan tasks, eight project commits, twenty-seven new tests. `var avgAge = avg(p.age),` now goes end-to-end on a real KieSession. #39 closed at `538185b`.

The spec and plan went in two days ago, both reviewed by Codex. Yesterday's session ran tasks 1–5: grammar, IR records, visitor fold, proto round-trip. Today's session ran tasks 6–9: the function registry, the runtime-builder lowering, the multi-function + count + sum tests, and the error-path coverage. Tasks 10 and 11 were wrap — full-suite regression, push, issue closure.

## The sealed-permit fold

The plan's IR had a `PendingAccumulatorIR implements LhsItemIR` as a transient visitor-time type that would hold the accumulators sitting on top of a pattern until the visitor saw the next non-accumulate item and could fold them into the real `AccumulatePatternIR`. `LhsItemIR` is sealed. The first round of Codex review caught that this can't compile — `PendingAccumulatorIR` isn't in the permits list, and adding it would leak builder semantics into the public type that every downstream switch has to handle.

The fold ended up in the visitor's local scope instead:

```java
PatternIR pendingPattern = null;
List<AccumulatorIR> pendingAccs = new ArrayList<>();

for (RuleItemContext rci : body.ruleItem()) {
    if (rci.accumulateItem() != null) {
        pendingAccs.add(buildAccumulator(rci.accumulateItem()));
        continue;
    }
    flushPending(out, pendingPattern, pendingAccs);   // constructs AccumulatePatternIR
    pendingPattern = null;
    pendingAccs = new ArrayList<>();
    // ... rest of the ruleItem dispatch
}
flushPending(out, pendingPattern, pendingAccs);
```

No new permit; the transient state lives in two local variables that go out of scope at the end of `buildRule`. The sealed contract stays honest. Submitted as a cross-project technique — the impulse to model intermediate state in the same type system as the final result is strong, and with sealed types it fights the design.

## The extractor that didn't get an MVEL3 path

The plan called for a new `createValueExtractor` method on `DrlxLambdaCompiler` — a parallel of the existing `createEvalExpression` but returning a `Function<Object, Object>` instead of a boolean. That would have wired in a new `DrlxValueExtractor` class shaped like `DrlxEvalExpression`, a new `PendingLambda` shape for batch compilation, the metadata-mismatch handling, and `compileBatch` integration. A real-ish chunk of work.

I asked Claude to survey the Drools `SingleAccumulate` API and the existing `DrlxLambdaCompiler` internals before I touched the runtime builder. The survey came back with confirmations on the constructor shape, the result-pattern wiring, and the location of `SelfReferenceClassFieldReader` (which lives at `org.drools.base.base.extractors`, not where the surrounding `ReadAccessor` interface lives — minor undocumented quirk, also submitted to the garden). Then I went back to the spec.

Every v1 example in DRLXXXX uses a simple property access — `p.age`, `p.amount`, or `count()` with no argument. The grammar admits any `expression`, but v1's accumulate semantics never need anything beyond `binding.property`. Building a generic MVEL3 lambda compile path just to support arithmetic in an arg expression is infrastructure spent on a use case that doesn't exist yet.

So v1's extractor is reflection:

```java
private static Function<Object, Object> buildSimpleExtractor(String argExpr, Class<?> srcClass) {
    // ... parse argExpr as "<binding>.<property>"
    Method getter = findGetter(srcClass, property);
    return obj -> {
        try { return getter.invoke(obj); }
        catch (ReflectiveOperationException e) { throw new RuntimeException(...); }
    };
}
```

Arbitrary expressions throw a clear v1-limitation error. The boundary is locked in by a test:

```java
@Test
void complexExtractorExpressionRejectedAsV1Limitation() {
    // sum(p.age + 1) — rejected with a v1-limit message
    assertThatThrownBy(() -> new DrlxRuleBuilder().build(rule))
        .hasMessageContaining("v1 accumulate supports only simple")
        .hasMessageContaining("<binding>.<property>");
}
```

When someone needs arithmetic, the test is the contract: the failure message will surface, the MVEL3 path goes in, and this test gets rewritten to assert success. Until then, no infrastructure I'd later have to defend.

## What sum() actually returns

The first multi-function test (min/max/avg over three Persons) passed clean. The second (`var total = sum(p.age)` over three ints) failed on what looked like the right number with the wrong type:

```
Expected: [100]
Actual:   [100.0]
```

`MinAccumulateFunction` and `MaxAccumulateFunction` preserve the input's Comparable type — `min(int_field)` returns `Integer`. `SumAccumulateFunction` does not. It accumulates into a `double` regardless of what the expression evaluates to, and always boxes to `Double` on `getResult`. The interface has a `getResultType()` method that would have told me this, but it isn't surfaced in any obvious docs page. The class itself is the only authoritative source.

The fix to the test was one character: `containsExactly(100)` → `containsExactly(100.0)`. The fix to the registry's resolver was nothing — the registry isn't trying to predict runtime types, it's just naming the function. The user-facing reality is that `sum` always gives you a `Double`, and the consequence has to type it that way.

Submitted as a gotcha. Anyone wiring Drools accumulate from outside the executable-model path will hit this.

## Source visibility stayed isolated by accident of structure

The spec says the source binding `p` is internal to the accumulate — visible inside the accumulator's argument expression, invisible to the consequence and to later patterns. The implementation makes `p` part of an `innerScope` map that's local to `buildAccumulatePattern` and never merged into the outer `boundVariables`. Easy enough on paper.

Verifying it took a separate route. The consequence body resolves identifiers against a type map built by `DrlxLambdaCompiler.getTypeMap(root)`, which walks `root.getChildren()` looking for `Pattern` and `GroupElement`. The inner source pattern isn't a child of root — it's the source of a `SingleAccumulate`, which is the source of the result pattern that *is* a child of root. `getTypeMap` doesn't follow `Pattern.setSource(...)`. The inner pattern simply isn't reachable from the walker's traversal.

So the scope isolation is enforced twice: once by my `outerScope` discipline in the IR-to-runtime path, and once by Drools' own type-collection only walking the GroupElement tree. The negative test (`results.add(p)` in the consequence) throws a `RuntimeException` from the MVEL3 lambda compiler with no help from anything I wrote.

## What v1 doesn't have

The full list of accumulate work still owed under epic #26:

- An MVEL3-backed extractor for arbitrary argument expressions.
- `MultiAccumulate` folding (collapsing the N×SingleAccumulate shape into one node).
- The inline-from form: `avg(/persons.age)` with no preceding source pattern.
- The `acc()` keyword forms (2-, 3-, and 5-param).
- Multi-pattern sources via `and(...)`.
- Custom accumulate functions imported by user code.

#39 itself is closed. The remaining accumulate work is its own thing now, tracked separately. v1 shipped exactly the shape it was scoped to ship: forms 1 and 2, five built-ins, N×SingleAccumulate lowering, qualified names rejected. 182 tests in, 209 tests out.
