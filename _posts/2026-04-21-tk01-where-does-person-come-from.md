---
layout: post
title: "Where does `Person` come from?"
date: 2026-04-21
type: pivot
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [drlx, language-design, type-inference, ruleunit]
---

Tier 2 started well. The plan for `not /pattern` — approved last
night — was a 12-task implementation beginning with an atomic
refactor of `RuleIR` to a tree shape. Task 1 landed clean: sealed
`LhsItemIR permits PatternIR, GroupElementIR`, `ConsequenceIR`
split out to its own field, proto schema rebuilt with a recursive
`LhsItem` message, all 50 existing tests green. Task 2 was a
one-line lexer token. Task 3 added the grammar production. Task 4
was a RED test.

Task 4 failed wrong.

```
line 8:28 mismatched input ',' expecting {'not', …}
```

I expected the visitor to throw `"Unsupported rule item"` — the
test is RED because `visitNotElement` isn't implemented yet, not
because the parser rejects the input. But the grammar rejected it,
because I forgot a comma.

`rulePattern` ends in a required `','`. I wrote `notElement : NOT
oopathExpression` with no trailing comma, and every real rule has
`not /persons[...],` with a comma before the next item. One-char
fix, separate commit: `fix(grammar): notElement requires trailing
comma`. RED then failed for the right reason. Committed. Task 4
done.

## The wall

Task 5 is the GREEN half — implement `visitNotElement`, stamp out
`GroupElementIR(NOT, [PatternIR])`. The runtime switch handles
`NOT → GroupElementFactory.newNotInstance()` from Task 1. Should
be small.

I paused to check `PatternIR` before dispatching.

```java
PatternIR(String typeName,
          String bindName,
          String entryPoint,
          List<String> conditions,
          String castTypeName,
          List<String> positionalArgs)
```

The runtime builder does `typeResolver.resolveType(typeName)` to
get the Java class. For every existing test `Person p :
/persons[...]` supplies `typeName = "Person"` explicitly. For the
DRLX spec's `not /persons[...]` there's no `Person` anywhere. Just
`persons`. The entry-point name.

The compiler has no way to know `persons` means `Person`.

Checking `var p : /persons[...]` — accepted by the grammar,
apparently works in parser-only tests, but no runtime test
exercises it. Because `typeResolver.resolveType("var")` obviously
throws `ClassNotFoundException`. The whole spec form of the
canonical DRLX `/entry` syntax is, right now, unreachable from
the runtime.

This is a hole the plan didn't see. Not a bug — a gap.

## The decomposition

I nearly reached for naive inference. Capitalize, strip trailing
`s`, try to resolve, done. It would've worked for `persons →
Person`, `orders → Order`, `locations → Location`. Fifteen lines.
Ship it.

Stopped. That's a shim, and it wouldn't have helped #9 (`exists`)
or the deferred `var` runtime either, because the real answer —
the one the DRLXXXX spec hints at — is the Drools RuleUnit model.
A unit class with typed DataSource fields:

```java
public class MyUnit {
    public DataStore<Person> persons;
    public DataStore<Location> locations;
}
```

Read that class at build time. Reflect over its fields. Build
`Map<entryPoint, Class<?>>`. Use it wherever `typeName` is absent
or `"var"`. This is the right shape. The naive version is
technical debt I'd replace in three weeks anyway.

Decomposing mid-plan hurts. #8 was moving. Task 4 was fresh. But
the plan hadn't accounted for this, and the honest call was to
pause, file `#14 (type inference from unit class)`, make #8 block
on it, and brainstorm the proper thing.

One more issue surfaced during the brainstorm — the runtime piece
where rules execute against a real `RuleUnitInstance` instead of
untyped `KieSession.getEntryPoint(name)`. That's #15 now.
Deferred, because #14 can land as compile-time only and still
solve the blocker. Keeps the cheapest path through the problem.

## The β decision

One thing I liked from the brainstorm: when a rule provides an
explicit type `Person p : /persons[...]` and the unit class has
`DataStore<Person> persons`, the compiler cross-checks. If they
don't match — `Wrong p : /persons[...]` against `DataStore<Person>`
— it fails loud. Existing tests keep working, and the compiler
now catches a whole class of typo in the bargain.

Option (β) in the spec. Not free but pretty close.

## What the plan says now

#14's plan — 10 tasks + final verification, 1217 lines, committed
`267e253` — lands the unit-class requirement, introduces a shared
`MyUnit.java` test class, migrates every existing DRLX source to
import it, adds `resolveUnitClass` + `buildEntryPointTypeMap` +
`resolvePatternType`, then nine tests in a new `TypeInferenceTest`
class. Final expected state: 60 tests, with one RED — `NotTest#notSuppressesMatch`
from #8 Task 4 staying RED until #8 resumes.

Session ends here. #14 executes next session with a fresh context
window; #8 resumes at Task 5 after #14 closes. The tree shape is
already in, the grammar is already in, the RED test is already
in. The wall was type inference, not any of that.

Next session picks up exactly at "dispatch #14 Task 1."
