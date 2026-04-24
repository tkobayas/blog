---
layout: post
title: "Strict rule assertion — from fire count to named matches"
date: 2026-04-24
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [drlx, drools, testing, junit, refactor]
---

#18 closed tonight. One vendored class, one signature change, 12 migrated files, 14 commits. The whole thing is test infrastructure — zero production-code changes. Interesting because the shape of the refactor fell out of five brainstorming questions, and because Drools answered a couple of questions I had been guessing about.

## The ticket

`int fired = kieSession.fireAllRules(); assertThat(fired).isEqualTo(2);` is the current test shape for anything multi-rule in drlx-parser. It tells you how many rules fired. It does not tell you *which*. For rules that differ only by salience, annotation, or a guard, a test can fire the wrong rule and pass.

Drools 10.2.0 ships `TrackingAgendaEventListener` for this — attaches to a `KieSession` and records rule names across all 10 agenda events. drlx-parser is still on 10.1.0, so the ticket was: copy the upstream class into `src/test`, migrate, delete when 10.2.0 ships.

## Five questions that shaped the diff

The brainstorming phase was unusually tight — each question narrowed one real axis. Five in total:

1. **How many files migrate?** Not all 16 — the 10 that already use an anon `DefaultAgendaEventListener` plus the two multi-rule compiler/builder tests. The four single-rule smoke tests stay as bare fire count.
2. **Where does the vendored class live?** Under its upstream package path (`src/test/java/org/drools/core/event/TrackingAgendaEventListener.java`), not under a `testutil` package. When drlx-parser picks up Drools 10.2.0, deleting the file is a single `rm` — the FQCN resolves from `drools-core` with zero import changes.
3. **Keep fire-count assertions alongside the listener, or replace?** Keep both. The `int` return is the natural cycle boundary for multi-fire tests; the listener accumulates across cycles and carries the names. No `resetAllEvents()` sprinkled through tests.
4. **Policy for future tests?** Bare `fireAllRules() == N` is allowed only for obvious single-rule tests. Anything that exercises multiple rules, firing order, or rule-specific behaviour uses the listener. Recorded in the spec as a durable rule.
5. **How does `withSession` change?** Signature moves from `Consumer<KieSession>` to `BiConsumer<KieSession, TrackingAgendaEventListener>`. The listener is always attached, always passed through. No opt-in helper.

Each of those Q&As produced a concrete line in the spec. No design meeting, no back-and-forth on hypothetical edge cases.

## The atomic signature change

Changing `withSession`'s signature breaks all 13 callers at once. Two options:

- **Overload dance:** add a new `withSession(rule, BiConsumer...)` alongside the old `Consumer` version, migrate one at a time, delete the old overload.
- **Atomic commit:** change the signature, rename every caller's lambda header, compile, commit.

Picked atomic. The compiler catches every caller I missed — nothing silent. 13 files in one commit, but each file's change is a two-character rename: `kieSession ->` becomes `(kieSession, listener) ->`. A single sed line handled them all:

```bash
find . -name "*.java" ! -name "DrlxBuilderTestSupport.java" \
  -exec sed -i 's/withSession(rule, kieSession -> {/withSession(rule, (kieSession, listener) -> {/g; s/withSession(rule, ks -> {/withSession(rule, (ks, listener) -> {/g' {} \;
```

`grep` afterwards confirmed zero stale headers. Tests passed unchanged — the `listener` parameter was accepted and ignored by every caller at this stage. The per-file assertion upgrades followed in 10 separate commits, each independently revertible.

## Multi-fire tests drop `fired.clear()`

`NotTest.notSuppressesMatch` was the prototype for every multi-fire migration.

Before:
```java
assertThat(kieSession.fireAllRules()).isEqualTo(1);
assertThat(fired).containsExactly("OnlyAdults");

fired.clear();
persons.insert(new Person("Charlie", 10));
assertThat(kieSession.fireAllRules()).isZero();
assertThat(fired).isEmpty();
```

After:
```java
assertThat(kieSession.fireAllRules()).isEqualTo(1);
assertThat(listener.getAfterMatchFired()).containsExactly("OnlyAdults");

persons.insert(new Person("Charlie", 10));
assertThat(kieSession.fireAllRules()).isZero();
assertThat(listener.getAfterMatchFired()).containsExactly("OnlyAdults");  // unchanged
```

The listener accumulates. `fireAllRules() == 0` already says nothing new fired in cycle 2; the listener's cumulative content staying at `["OnlyAdults"]` says the same thing from a different angle. No reset, no `resetAllEvents`, no bookkeeping.

## Drools confirms OR rule naming

Writing the plan I had to assume something about Drools' `LogicTransformer`: when it splits a top-level OR into two rule instances, do the instances share the original rule name or get `_0`/`_1` suffixes? I assumed shared — the weaker claim — and flagged it in the plan as a "watch this; update if wrong."

`OrBindingScopeTest.orBranchBindingLocal` fires both OR branches:

```java
assertThat(listener.getAfterMatchFired())
    .containsExactly("BranchLocalJoin", "BranchLocalJoin");
```

Passed first run. Drools keeps the original name across split branches. Same for `OrTest.orBothBranchesFire` — `containsExactly("SeniorOrJunior", "SeniorOrJunior")`. Good to pin that down.

## A surefire quirk I'll remember

Mid-migration I ran `mvn test -Dtest=DrlxRuleBuilderTest` to verify one file. Output:

```
[WARNING] Tests run: 5, Failures: 0, Errors: 0, Skipped: 5
```

All five skipped. The full suite had them passing seconds before. The class has `@DisabledIfSystemProperty(named = "mvel3.compiler.lambda.persistence", matches = "false")` — Surefire with a single-class `-Dtest=` target somehow flips that property or fails to unset it. The full-suite run was green; the targeted run was a ghost.

Noting it here so future-me doesn't spend fifteen minutes thinking the test is broken.

## What landed

14 commits on `main` (`195fe39..113cf22`), 96 tests passing (no change from baseline — this is pure infra):

- `test: vendor TrackingAgendaEventListener from Drools 10.2.0-rc2`
- `test: change withSession to pass TrackingAgendaEventListener to lambda` (the atomic 14-file commit)
- Ten per-file migration commits for the anon-listener tests
- Two inline-pattern commits for `DrlxRuleBuilderTest` and `DrlxCompilerTest`

Issue #18 closed. On the Drools 10.2.0 upgrade path: delete `src/test/java/org/drools/core/event/TrackingAgendaEventListener.java`; no other changes.
