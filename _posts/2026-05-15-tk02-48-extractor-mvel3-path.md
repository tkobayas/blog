---
layout: post
title: "#48 — the extractor gets its MVEL3 path"
date: 2026-05-15
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [drlx, drools, accumulate, mvel3, java]
---

# #48 — the extractor gets its MVEL3 path

The v1-limit test in `AccumulateTest` flipped today. The previous entry's framing was: "when someone needs arithmetic, the MVEL3 path goes in." #48 is that. Five commits, four new tests, the reflection helper trio deleted, 209 → 213 green.

## The path that was deferred is exactly the path that landed

The shape #39 sketched in its plan-but-didn't-build held up. A new `DrlxValueExtractor` class implementing `Function<Object, Object>` + `EvaluatorSink`, paired with a new `createValueExtractor` method on `DrlxLambdaCompiler` that mirrors `createEvalExpression` line for line — `tryLoadPreCompiled` first, then `createBatchValueExtractor` registering a `PendingLambda` for late binding. The only difference is the output type: `Object` instead of `Boolean`, plus `<Object>out(Object.class)` on the MVEL3 builder. Pre-build metadata flow, batch compile, mismatch handling — all of it falls out for free because the pattern is established.

`DrlxLambdaAccumulator` didn't move. Its extractor field is still `Function<Object, Object>`; the new class implements that interface, so the wiring was a one-line swap in `buildSingleAccumulate`. Then the deletes: `buildSimpleExtractor`, `isIdentifier`, `findGetter`, and the `java.lang.reflect.Method` import — replaced by passing `Declaration.of("p", srcClass)` into the MVEL3 map builder.

## Map mode beat pojo mode for a structural reason

The compile-mode decision wasn't perf, it was syntax. The accumulate-arg expression in the source is `p.age + 1` — with the binding prefix. Two ways to compile this through MVEL3:

```java
// Map mode: declarations carry the binding name
MVEL.<Object>map(Declaration.of("p", Person.class))
    .<Object>out(Object.class)
    .expression("p.age + 1")
    // -> runtime: HashMap{"p": fact}, evaluator.eval(map)

// Pojo mode: fact is the receiver, no binding prefix
MVEL.pojo(Person.class, propertyDecls)
    .<Object>out(Object.class)
    .expression("age + 1")   // syntactic rewrite required
```

Pojo mode is faster — no HashMap allocation per accumulate call — but the expression syntax has to be rewritten to strip the binding prefix. That's a regex over user-authored MVEL3 with strings and comments in scope. The general rewrite is fragile. Map mode preserves the expression syntax, matches what `DrlxLambdaBetaConstraint` already does for beta constraints, and falls naturally to the outer-binding-refs extension that's been parked as the next follow-up.

The HashMap-per-call cost shows up in microbenchmarks; it doesn't show up in the 209→213-test wallclock. Defer.

## Codex caught two real things in the spec

Asking Codex to review the spec before writing the plan paid off. Two findings, both medium:

> Medium: `DrlxValueExtractor` is specified to include the source class name in runtime error messages, but its state does not include `srcClass`. Either add `Class<?> srcClass` to the constructor/state or drop that promise from the error-handling section.

I dropped the promise. The expression string is on state, and the fact's actual class is reachable via `fact.getClass()` at the throw site if a future improvement wants it.

> Medium: the spec says outer-binding references inside extractor expressions are out of scope, but `count(x)` is explicitly unchanged and ignores its argument. That means `count(q.factor)` would likely continue to skip extractor construction and avoid validating `q` entirely.

I made the existing v1 behavior explicit in the spec: `count`'s argument is parsed but not validated; `count(garbage)` and `count(q.factor)` both run as plain `count()`. Compile-validating `count`'s arg is a deliberate future improvement, not scope creep for this issue.

The interesting bit is that the inconsistencies were in a 200-line spec that I had reviewed twice and that had been section-approved already. A fresh reader with no investment in the framing caught them; a primary author who'd just designed the thing didn't.

## MVEL3 catches unknown properties at batch compile

The negative test was the small surprise. `sum(p.notAField)` — the expectation written into the plan was that MVEL3 might defer this to evaluation time, in which case the test would have to insert a Person and assert the throw on `fireAllRules()`. It doesn't. Batch compile validates the property reference against the declared type and raises during `DrlxRuleBuilder.build()` before any fact gets near it. The test asserts on `RuntimeException` only — not on the specific message, which could shift across MVEL3 versions — and it passes at build time exactly as written.

That's a quietly useful property of the architecture. Map-mode declarations aren't opaque containers; the MVEL3 transpiler resolves property paths against them just as it does for pojo-mode receivers. Eager validation for free.

## What landed

| Commit | Subject |
|---|---|
| `1f19670` | `feat(builder): add DrlxValueExtractor — MVEL3-backed Function<Object,Object>` |
| `8f3c32d` | `feat(lambda): add createValueExtractor compile path` |
| `eea764f` | `feat(builder): wire MVEL3 extractor in accumulate; drop reflection path` |
| `e85ffca` | `test: cover arbitrary-expression accumulate args` |
| `27c528f` | `test: extractor with unknown property fails at build` |

`drlx-parser-core` suite: 209 → 213 green. Reactor BUILD SUCCESS. #48 closed at `27c528f`. The remaining accumulate children are still on the menu — `MultiAccumulate` folding, inline-from form, `acc()` keyword variants, multi-pattern sources via `and(...)`, custom imported functions. Outer-binding extractor refs join that menu as a separate follow-up that the map-mode foundation now makes a small delta.
