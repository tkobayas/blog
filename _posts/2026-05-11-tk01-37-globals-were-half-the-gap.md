---
layout: post
title: "#37 part 1 — globals were half of the gap"
date: 2026-05-11
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [drools, ruleunits, datastore, globals, mvel3, scope-creep]
---

# #37 part 1 — globals were half of the gap

Started the day on #37 (DataStore CRUD: `add` / `remove` / `update` in `do` blocks).
Quickly realised the existing test pattern wouldn't work — every test in
`drlx-parser-core` builds rules with `kieBase.newKieSession()` and inserts via
`session.getEntryPoint("persons").insert(...)`. The rule consequence under #37 wants
to call `alerts.add(...)` on a unit-field reference — and that needs the RuleUnitInstance
machinery, not raw `KieSession`. Different abstraction.

So #37 became two pieces of work: build the test plumbing, then do the actual parser
change.

## The wrapper

Upstream's `RuleUnitProvider.createRuleUnitInstance(unit)` looks for a generated
`RuleUnit<T>` class registered via `ServiceLoader`, produced by drools-ruleunits codegen
scanning `.drl` files on the classpath. DRLX compiles rules from a string at runtime —
no codegen, no service entry. That path is closed.

I asked Claude to think through alternatives. We landed on a test-only wrapper that
bypasses the upstream provider entirely: cast the DRLX-built `KieBase` to
`InternalRuleBase`, wrap it in `RuleUnitExecutorImpl` (a `ReteEvaluator`), and run the
same bind step `AbstractRuleUnitInstance` runs — for each public `DataSource<?>` field
on the unit, subscribe an `EntryPointDataProcessor` to the matching named entry point,
and set the DataSource as a global of the same name. Then I changed my mind on placement:
src/test would have been fine, but the wrapper is small and represents a legitimate DRLX
public API (given a built KieBase, run it rule-units-style), so it moved to src/main.

Three commits landed cleanly: `MyUnit` promoted to `RuleUnitData` (initialise each
`DataStore<X>` field via `DataSource.createStore()`), a `TestDataObserver<T>` utility
that captures inserts/updates/removes on a DataSource via the standard `DataProcessor`
contract, and the `DrlxRuleUnitInstance<T>` wrapper itself implementing the upstream
`RuleUnitInstance` interface. Pushed `b034bf7`, `4b70f5d`, `f99b693` to `origin/main`.

## The probe

Before designing #37's parser work, I wanted to see what already worked. The issue body
claims "Parser already handles method calls in `do` blocks." So a one-off test:

```java
rule CopyAdults {
    Person p : /persons[ age > 30 ],
    do { persons1.add(p); }
}
```

Build via `new DrlxRuleBuilder().build(rule)`, wrap, fire. Expectation: would either
work end-to-end (and #37 was about adding tests, nothing more) or fail in a specific
place that told us where to start.

It failed at parse:

```
UnsolvedSymbolException{context='persons1', name='Solving persons1'}
    at DrlxRuleBuilder.parse(DrlxRuleBuilder.java:112)
```

Method calls in `do` blocks work — for arbitrary expressions like `System.out.println(p)`.
Method calls on a unit-field reference do not, because unit-field names aren't symbols
in the consequence's lexical scope. The parser doesn't know `persons1` exists.

## The smoking gun

Claude went looking at the upstream rule-units codegen for the analogue we were
missing. `PackageModel.addRuleUnitVariable`, line 378:

```java
public void addRuleUnitVariable(String unitName, RuleUnitVariable unitVar) {
    RuleUnitMembers unitMembers = ruleUnitMembers.computeIfAbsent(unitName, k -> new RuleUnitMembers());
    String unitVarName = unitVar.getName();
    unitMembers.globals.put( unitVarName, unitVar.getType() );    // global
    if ( unitVar.isDataSource() ) {
        unitMembers.entryPoints.add( unitVarName );                // also entry point
    }
}
```

Every unit field becomes a **global**. DataSource-typed fields *additionally* become
entry points. DRLX did half of that today — `DrlxRuleAstRuntimeBuilder.buildEntryPointTypeMap`
collected entry points from DataSource fields only, never registered any field as a
global, and only declared entry points implicitly through `Pattern.setSource(new
EntryPointId(...))`. So a DataSource that wasn't used as an LHS pattern source —
exactly the #37 case where `alerts` is consequence-only — had neither global nor entry
point on the package.

Pinned the probe with `@Disabled("blocked on #37 — unit-class fields not yet registered
as globals; mirror PackageModel.addRuleUnitVariable in DrlxRuleAstRuntimeBuilder")` and
committed as `2cc04c9`. The next round had a clear target.

## The spec said "register globals." The implementation needed three things.

Brainstorm → spec → plan, same flow as the wrapper. Spec covered: add a
`buildGlobalTypeMap(unitClass)` helper, register each entry on the package via
`pkg.addGlobal(name, type)`, merge the field types into the MVEL3 consequence type
map after the LHS bindings. Three small changes in one file. Plan written, three TDD
tasks, inline execution.

Task 1 — the helper — landed at `8528cf3` with a unit test. Clean. The trouble started
at Task 2.

**Gap 1 — MVEL3 compile time.** Re-enabled the probe. Same `UnsolvedSymbol` error.
Implemented the type-map merge: after `lambdaCompiler.getTypeMap(root)` populates the
LHS bindings, loop over `globalTypes` and `types.put(name, Type.type(rawClass))` for
each. Re-ran. MVEL3 now compiled the consequence — but the test still failed.

**Gap 2 — eval time.** New error, deeper in the stack:

```
java.lang.RuntimeException: java.lang.NullPointerException
    at DrlxLambdaConsequence.evaluate(DrlxLambdaConsequence.java:71)
```

Line 71 is `evaluator.eval(vars)`. The `vars` map is populated with LHS declaration
values from `knowledgeHelper.getMatch()` — nothing else. MVEL3 compiled the code
assuming `persons1` is a symbol of type `DataStore`, but at eval time the value
isn't in `vars`. Null → NPE on `add(p)`.

Globals at compile time aren't globals at eval time. The spec covered the compile-time
side. The eval-time side needed `DrlxLambdaConsequence` to know which symbols are
globals and call `valueResolver.getGlobal(name)` for each before invoking the
evaluator. Added a `Set<String> globalNames` field, threaded it through
`DrlxLambdaCompiler.createLambdaConsequence`, populated `vars` from
`valueResolver.getGlobal(name)` in `evaluate()`. Re-ran.

**Gap 3 — the entry point that wasn't.** Third error:

```
NullPointerException: Cannot invoke "EntryPoint.insert(Object)" because "this.entryPoint" is null
```

The wrapper's bind step does `reteEvaluator.getEntryPoint("persons1")` and passes that
to `new EntryPointDataProcessor(...)`. If no rule's LHS pattern source is `/persons1`,
the KieBase has no declared entry point named `persons1`, and `getEntryPoint` returns
null. The wrapper then wraps null in an `EntryPointDataProcessor`, the rule fires,
`alerts.add(p)` runs through subscribers, and the EntryPointDataProcessor tries to
`null.insert(p)`.

This was the surprise. The spec was about globals. The fix needed entry points too —
because upstream's `addRuleUnitVariable` *also* declares entry points for DataSource
fields regardless of whether any LHS pattern uses them. DRLX only declared entry points
when a pattern referenced them. That gap had been latent until #37 introduced the first
consequence-only DataSource.

Paused, asked: extend scope or punt? Extending was three lines — find the right API
(`KnowledgePackageImpl.addEntryPointId(String)`, exists), and call it for every
DataSource field. The alternative was a wrapper-side defensive null check that would
silently make the inserted fact bypass the rete network. Extending was the right call.
Added `entryPointTypes.keySet().forEach(pkg::addEntryPointId)` next to the globals
registration. Tests went green. Commit `7995112`.

Task 3 (extra happy-path tests for `remove(T)` and multi-field reference) added two
more tests against the now-working machinery and landed at `cab2862`. Full suite:
162 passing, 0 failures. Pushed `f99b693..cab2862` to `origin/main`.

## Mirroring half doesn't work

The lesson is structural. When porting behaviour from an existing system, the question
isn't "what's the analogous registration site?" — it's "what are *all* the registration
sites, and which ones is the existing port missing?" `PackageModel.addRuleUnitVariable`
does two things in one method body. Today's DRLX was doing one of them, and the missing
one was invisible until a feature with the right shape (consequence-side mutation of a
DataSource that no LHS rule pattern references) showed up.

The spec captured the globals half. The eval-time injection and the entry-point
declaration were both gaps the spec didn't see — and the plan's "common failure modes
to debug" section had a generic "Common failure modes if it does not [pass]" list but
didn't predict the cascade. The cascade only shows up in execution.

## End state

#37 closed as completed (add and remove landed; the original "Add/Remove/Update" framing
was wider than what landed). `update(T)` coercion split out as #45 (the DataStore API
has `update(DataHandle, T)` but no single-arg form — needs translation in DRLX or a
runtime helper). The `with`-block compact update syntax (`alerts.update(t{status = X})`)
is already tracked in #34, which I noticed while creating the new issue — closed my
new duplicate (#46) and added a dependency note to #34 instead.

Epic #26 body updated: #37 moved to "Already implemented", #45 added to High Priority.
The disabled probe from this morning is now an enabled passing test.

Seven project commits, one workspace spec, one workspace plan. The wrapper is public
API now; if anyone wants to run a DRLX-built KieBase against a `RuleUnitData`,
`DrlxRuleUnitInstance.create(kieBase, unit)` is the entry point.
