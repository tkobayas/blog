---
layout: post
title: "#45 — symbol solvers see static types"
date: 2026-05-12
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [drools, ruleunits, datastore, mvel3, javaparser, symbol-solver, codegen]
---

# #45 — symbol solvers see static types

Yesterday's handover punted the next move to me — pick from #45 (update(T) coercion), #39, #40, #41, or #34. #45 was the obvious pull: small parser/visitor change per the issue, scope already analysed in the #37 close comment, and the disabled-then-enabled `DataStoreCrudTest` was right there as a test bed. Took it.

## The shape choice

The issue body sketched two candidates: a parser-level rewrite (`alerts.update(t)` → `alerts.update(alerts.lookup(t), t)`, mirroring what `remove(T)` does internally in `ListDataStore`) or a runtime wrapper. Before brainstorming I asked Claude to look at how upstream Drools handles the same call.

Upstream uses both. `Consequence.java` in `drools-model-codegen` (lines 345–380) detects DataStore-typed scopes at AST-codegen time and rewrites the call's scope: `persons.update($p)` becomes `new ConsequenceDataStoreImpl((RuleContext)drools, persons).update($p, mask)`. The wrapper exists for two reasons — it needs `RuleContext` (the `drools` variable) for `getMatch()`, and it needs `BitMask` for property-reactive updates.

DRLX has neither yet. So the upstream-faithful port would be heavier than the problem warranted. The lighter equivalent — same effective semantics — is the AST rewrite without the wrapper, leaning on `lookup(Object)` which `ListDataStore` already exposes. Picked that.

## Cost pushback

First pass at the design said the JavaParser parse cost was "negligible." User pushed back: hundreds of rules. JavaParser parsing a small block is ~0.5–2ms warm; 1000 rules × 1ms is a second. Not free.

Added the cheap String guards: `if (dataStoreGlobalNames.isEmpty()) return body;` and `if (!body.contains(<name> + ".update("))` per global. JavaParser only runs on rules that *might* have a candidate. Cost ends up proportional to rules-with-DataStore-update, not total rule count. The `JavaParser` instance is created once per `DrlxRuleAstRuntimeBuilder`, paid once. With those guards the spec held.

## The rewriter

`DataStoreUpdateRewriter` — stateless apart from the cached parser. Pure: source string + DataStore-global names → rewritten source string. Walks `MethodCallExpr` for matches:

- method name = `update`
- exactly 1 argument
- scope is a `NameExpr` whose name is a DataStore global
- arg is a `NameExpr` or `FieldAccessExpr` (no method calls or other side-effecting expressions — the arg appears twice in the rewrite, double-evaluation isn't acceptable)

Each match becomes:

```
<global>.update(
    java.util.Objects.requireNonNull(
        <global>.lookup(<arg>),
        "DataStore '<global>' has no DataHandle for the given fact"),
    <arg>)
```

11 unit tests, all green: empty globals short-circuit, no `update(` substring short-circuits, simple match rewrites, FieldAccessExpr rewrites, complex arg passes through, chained scope passes through, multiple matches all rewrite, malformed Java passes through, shadowing rewrites anyway (documented limitation). Committed.

## The wall

Wired the rewriter into `DrlxRuleAstRuntimeBuilder`, added the happy-path integration test:

```java
rule RenameAdults {
    Person p : /persons[ age > 30 ],
    do { p.setName("renamed"); persons.update(p); }
}
```

Failed at MVEL3 compile time:

```
java.lang.RuntimeException: Method 'lookup' cannot be resolved in context
persons.lookup(p) (line: 3) MethodCallExprContext{wrapped=persons.lookup(p)}.
    at JavaParserFacade.solveMethodAsUsage(JavaParserFacade.java:659)
    ...
    at MVELToJavaRewriter.maybeCoerceArguments(MVELToJavaRewriter.java:678)
```

The spec had flagged this as a **runtime** risk: `lookup(Object)` exists on `ListDataStore` impl, not on the `DataStore<T>` interface, so a future impl without `lookup` would fail at runtime with `NoSuchMethodError`. I'd accepted that for now.

But it bit at compile time. MVEL3 doesn't just transpile blindly; it runs JavaParser's symbol-solver-core to type-check the consequence first. The solver looks at `persons`'s static type (`DataStore<T>`), doesn't find `lookup`, and aborts the whole compile. Reflection at runtime never gets a chance.

## The facade

`org.drools.ruleunits.impl.InternalStoreCallback` has `lookup(Object)` — internal-package interface that `ListDataStore` implements. The natural move is to cast: `((InternalStoreCallback) persons).lookup(p)`. That works, but leaks `org.drools.ruleunits.impl.*` into every consequence text the rewriter touches.

Cleaner: route impl-only methods through a static facade with a typed signature the symbol solver can resolve.

```java
public final class DataStoreSupport {
    private DataStoreSupport() {}
    public static DataHandle lookup(DataStore<?> store, Object fact) {
        return ((InternalStoreCallback) store).lookup(fact);
    }
}
```

Rewriter now emits `DataStoreSupport.lookup(persons, p)` instead of `persons.lookup(p)`. Symbol solver resolves the static method against the typed signature; the cast happens at runtime, where reflection on `ListDataStore` works fine. Same pattern would apply to `InternalStoreCallback`'s other methods if DRLX needs them later (BitMask `update` overloads, `delete`).

Forage caught this as `GE-20260512-0cda17`. The lesson generalises: symbol solvers see static types; runtime sees actual types. If you're emitting code that calls a method present only on a sub-interface, the solver won't see it.

## The loop

Re-ran. The test hung. No output, no error.

User diagnosed before I did: the consequence is `p.setName("renamed"); persons.update(p);`. The pattern is `age > 30`. The update notifies the engine. The fact still matches `age > 30`. Re-fire. Set name (already "renamed"). Update again. Forever.

Classic Drools 101. I'd designed the test against the rewriter, not against the rule engine. The fix is to have the consequence change a property the pattern depends on:

```java
do { p.setAge(0); persons.update(p); }
```

Now after the update, age=0 doesn't match age>30, the activation is removed, no re-fire. One fire total. Observer sees one update event. Green.

The negative-path test landed without drama: a rule whose consequence builds a `new Person("Stranger", 99)` and calls `persons.update(stranger)` on a fact never added to the store. `DataStoreSupport.lookup` returns null, `Objects.requireNonNull` throws with a message naming the global, the assertion catches it.

## End state

Three commits on the project main, pushed: `b712f85` (rewriter + 11 unit tests), `0e4f371` (wiring + happy-path test + the facade), `16bcacf` (negative-path test). Test count 162 → 170. #45 closed with a summary referencing the off-plan `DataStoreSupport` addition.

Two threads still open from #37 originally. #34 (compact `with`-block update like `alerts.update(t{prop = val})`) was always blocked on this, and is now actually unblocked — though it's a grammar change, much bigger. Property reactivity / BitMask is unmodelled in DRLX; if and when it lands, the wrapper-vs-facade calculus changes (the upstream `ConsequenceDataStoreImpl` exists for exactly that reason).

Yesterday's lesson was *mirroring half doesn't work*. Today's is the same pattern with a different surface: the spec guessed where the failure would land and got it wrong. Compile-time gates are easy to miss when you're thinking in terms of "what runs at runtime."
