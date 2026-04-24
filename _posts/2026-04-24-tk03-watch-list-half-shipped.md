---
layout: post
title: "Watch list — half shipped, and why"
date: 2026-04-24
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [drlx, drools, property-reactivity, rete, codex]
---

**#13 closed.** Property-reactive watch list — the second `[]` block on an oopath root — shipped: grammar, IR, proto, visitor, runtime validation, end-to-end tests. What I didn't know going in: the feature only half-works. For rules like `[][basePay]` (watch-only, no conditions) the filter does its job. For `[salary > 0][basePay]` (watch list on top of a constraint) the alpha node doesn't care what the watch list says and everything modifies through. That limitation is real, documented, and filed as #19.

## The ticket

DRL7 has `@Watch(basePay, bonusPay)` — pattern-level annotation that adds properties to the `Pattern.listenedProperties` set. DRLX spec candidate #10 replaces the annotation with a second `[]` block on `oopathRoot`: `Employee e: /employees[salary > 0][basePay, bonusPay]`. Issue #13 asked for grammar + visitor + runtime wiring. Not specified: wildcard support, exclusion semantics, how strict the compiler should be.

## Picking the scope

Three flavours offered at brainstorming:

- **A.** Plain identifiers only.
- **B.** Plain + Drools wildcards (`*`, `!*`, `!foo`). Matches DRL7 at the syntax level.
- **C.** B plus auto-binding constraint-referenced fields (DRL7's `PatternBuilder` does this).

Went with B. C would have meant walking the constraint AST to find field references — a separate project's worth of work. Asking "how does current DRL handle a bad entry?" was the interesting question. Turned out DRL's `PatternBuilder.processListenedPropertiesAnnotation` does three compile-time checks: unknown property, duplicate property, duplicate wildcard — plus a fourth (class not `@PropertyReactive`) that I skipped because DRLX has no TypeDeclaration concept yet.

## The build — first six tasks, no surprises

Grammar was two edits: allow the first `[]` block to be empty (to enable `[][watch]`), add a second optional block, add a `watchItem` rule covering `!?` followed by `*` or identifier. Three parse-only tests went red → green. Commit.

IR, proto, parse-result — straight mechanical. Four `PatternIR(...)` call sites updated with `, List.of()` as a placeholder, visitor's `extractWatchedProperties` followed the pattern of `extractPositionalArgs`, runtime builder's validation mirrored DRL's `processListenedPropertiesAnnotation` one-to-one. All 96 existing tests green at every commit.

## Test 7 red. Not what I expected.

First behavioural tests: plain watch list + watch-only empty-conditions. Both failed. `expected: 0 but was: 1`. The watch list wasn't filtering anything.

Debug output:

```
[DBG] pattern ReactiveEmployee listenedProperties=[basePay, bonusPay]
[DBG] registering TypeDeclaration for org.drools.drlx.domain.ReactiveEmployee propertyReactive=true
[DBG] kBase.getTypeDeclaration(ReactiveEmployee)=TypeDeclaration{...} isPropertyReactive=true
```

Everything I could see was green. Pattern had the right listened-properties. TypeDeclaration registered. RuleBase recognised the class. Still re-firing on every modify.

## First fix: `update(fh, obj)` ≠ `update(fh, obj, "prop")`

Reading `NamedEntryPoint.java:287-320` solved half the puzzle:

```java
public void update(final FactHandle factHandle, final Object object) {
    update((InternalFactHandle) factHandle, object, allSetBitMask(), Object.class, null);
}
```

The 2-arg update **hard-codes** the modification mask to `AllSetBitMask.get()`. Every property treated as modified. No matter how carefully Drools has tagged a class as property-reactive, the 2-arg update says "everything changed." The 3-arg form — `ep.update(fh, emp, "salary")` — is what calls `calculateUpdateBitMask` and produces a real targeted mask. Inside rule consequences, `modify($x) { setX(...) }` compiles to the 3-arg form. From the outside, you must pass the names.

Updated all tests. Test 2 (`[][basePay]`) went green.

Test 1 (`[salary > 0][basePay, bonusPay]`) stayed red.

## The actual problem — alpha mask wasn't going to cooperate

Trail:

- `AlphaNode.modifyObject` (drools-core:129-143) gates on `context.getModificationMask().intersects(inferredMask)`.
- `inferredMask` is built by `AbstractTerminalNode.initInferredMask()` — calls `alphaNode.updateMask(getDeclaredMask())`, which in `ObjectSource.updateMask` unions alpha's declaredMask with downstream (RTN) mask.
- RTN's declaredMask = `pattern.getPositiveWatchMask` → watch-list mask `[basePay, bonusPay]`. Correct.
- Alpha's declaredMask = `constraint.getListenedPropertyMask(pattern, objectType, settableProperties)` (`AlphaNode.java:314-320`).
- Default `Constraint.getListenedPropertyMask` returns `AllSetButLastBitMask.get()` — all bits except bit 0 (the trait bit). Effectively "listen to every property."
- `DrlxLambdaConstraint` doesn't override.

So for any pattern with even a single alpha constraint, alpha's inferred mask = AllSet ∪ anything = still AllSet. The watch list's restrictive effect is defeated before the mask reaches the RTN.

In executable model (`LambdaConstraint.java:166-188`) the override exists — the constraint reports which fields it touches, using pre-computed evaluator metadata. DRLX doesn't have that metadata threaded through yet.

## Second opinion

Scope decisions on newly-discovered Drools behaviour deserve a cross-check. Sent the analysis to Codex with paths to verify. Came back confirming every link in the chain with exact file:line references, plus one nuance I hadn't thought about (`AllSetButLastBitMask`'s specific meaning — the "last" is bit 0, the trait bit, not the literal final bit). No simpler config fix. The missing piece really is `DrlxLambdaConstraint.getListenedPropertyMask()`.

## Scoping down

Two choices:
1. Ship the feature as-is, document the condition-interaction limitation, file the full fix as follow-up.
2. Expand #13 to include the constraint-mask override — a substantial separate feature.

Went with (1). The watch-list syntax, validation, proto serialisation, and runtime wiring are all solid. The only thing missing is one method on one constraint class — and that's its own project. #19 filed with paths, line numbers, and the executable-model reference.

Tests restricted to the empty-conditions form: `[][basePay, bonusPay]`, `[][*]`, `[][!*]`, `[][*, !bonusPay]`. Plus error paths (unknown / duplicate / wildcard). Plus proto round-trip. 108 tests passing, 0 failures.

## What I'd tell myself on Monday

Three things to remember:

1. **`ep.update(fh, obj)` without property names is `AllSetBitMask`.** For every test or external integration that cares about property reactivity, use the 3-arg form. Inside consequences, `modify { setX(...) }`. Without the names, Drools treats every modification as "all changed."

2. **TypeDeclaration registration matters for programmatic KieBase building.** `KiePackagesBuilder` (executable model) registers TypeDeclarations in the class's *own* package, not the rule's. Mirror that. Without it, `isPropertyReactive(ruleBase, objectType)` returns false and the whole property-reactive machinery stays dormant.

3. **`AllSetButLastBitMask` is "all except bit 0".** The traitable bit is bit 0; `CUSTOM_BITS_OFFSET = 1`. For property-reactivity math on classes that don't use traits, "all except 0" is effectively "all real properties" — which is why defaulting to it makes constraints catch-alls.

Next candidate is either #12 (if/else branching) or #19 (the constraint mask we just discovered). The second one would finish what this one started — and the tests to validate it are already written.
