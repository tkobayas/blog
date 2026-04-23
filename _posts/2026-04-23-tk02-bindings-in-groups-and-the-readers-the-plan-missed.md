---
layout: post
title: "Bindings in groups — and the readers the plan missed"
date: 2026-04-23
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [drlx, antlr, drools, group-elements, bindings, grammar-refactor]
---

#17 — bound patterns as children of `and(...)`, `or(...)`, `not(...)`, `exists(...)` — landed tonight. 7 commits, 84 → 94 tests passing. Same day as #11; the two went back-to-back.

The feature itself was small. #11's refactor γ had already factored the trailing `,` out of every CE rule and created a single `groupChild` non-terminal. All bindings-in-groups needed was one more alternative in that non-terminal and a visitor dispatch arm. `PatternIR` already had `typeName` and `bindName` fields; the runtime already built a bound `Pattern` whenever those fields were non-empty.

The story is what the plan didn't see coming.

## Scope decided in three multiple-choice questions

Q1: which CE paren forms accept bindings? **All four.** The alternative would have reintroduced the asymmetry refactor γ just got rid of. Q2: OR scoping? **Branch-local** — a binding inside `or(...)` never escapes the group. Drools' full consensus rule (same name on every branch → visible outside) is a can of worms I didn't want to open for this ticket. Q3: grammar shape? **Factor out a new non-terminal** `boundOopath : identifier identifier (':' | '=') oopathExpression`. Both `rulePattern` (with trailing `,`) and `groupChild` (without) reuse it.

Three "ok"s and the spec was done. Straightest path from "next ticket flagged in #11's blog" to "ready to plan" I've had on this project.

## Execution went sideways (briefly)

The plan was tight: write one failing test, change grammar, update visitor, commit them together. Task 1's grammar edit compiled clean. Task 2 was: rebuild, expect the test still fails — parser now reaches the visitor, visitor doesn't know about `boundOopath` yet.

The rebuild didn't reach the parser. It failed at compile:

```
DrlxToRuleAstVisitor.java:[262,30] cannot find symbol
  symbol:   method identifier(int)
  location: variable ctx of type org.drools.drlx.parser.DrlxParser.RulePatternContext
```

Of course. `rulePattern` no longer held the `identifier` accessors directly — they'd moved onto `BoundOopathContext` when we factored out the new non-terminal. The plan *did* anticipate this for `DrlxToRuleAstVisitor`; Task 4 was there to rewrite its accessors. Fine. Fixed.

Build again. Same kind of error, different file:

```
DrlxToJavaParserVisitor.java:[437,30] cannot find symbol
```

The *frozen-parent* LSP visitor had its own `visitRulePattern` that read `ctx.identifier(0/1)` and `ctx.oopathExpression()`. Not in the plan. Patched it — a two-line delegation to a new `visitBoundOopath` that did the actual work. Build again.

Still failing. `DrlxParserTest` — the low-level parse-tree assertion test — read `pattern.identifier(0).getText()` directly off `RulePatternContext` to check node shape. Two assertions in two test methods. Simple `.boundOopath().` hop to fix.

At that point we'd hit three unplanned readers. I stopped to grep before continuing: `identifier(0).getText()` → just the four we'd fixed. `oopathExpression()` on a `RulePatternContext` → same set. Done.

Lesson for next time a grammar non-terminal gets factored out: the primary visitor is the one the plan thinks about, but a surface change like this fans out to every Java reader of the old accessors. Frozen-parent visitors and test files are easy to miss because they're not in the primary code path. Grep-able; the plan should do the grep upfront.

## `orBindingNotVisibleAfterGroup` — Drools actually tells you

I'd been hedging on the branch-local boundary test. The spec said "either Drools throws at build, or the session silently produces zero fires — assert whichever Drools does." The plan carried the same hedge: fix-forward with a zero-fire assertion if the throw doesn't happen.

It threw. First try. `RuntimeException` during `KieBase.newKieSession()` — Drools notices the unresolved binding and refuses. The test just wraps `withSession` in `assertThatThrownBy` and calls it good. Concrete enforcement of Decision #2 at the system boundary, without DRLX having to police anything itself.

## `GroupElement.pack()` didn't eat any bindings

The other thing I'd flagged as a maybe-gotcha was `NestedGroupTest.andWithNestedNotCarryingBinding`: outer AND binds `p`, nested NOT references `p` in its condition. Drools' `pack()` rewrites the LHS tree at compile time (the NOT/EXISTS unwrap lives in drools-base's `GroupElement.java:167-181`), and the spec flagged it as possibly dropping bindings across the rewrite.

First-try green. Whatever `pack()` does, it preserves the binding table across the nesting boundary — at least for `AND` containing `NOT`. Didn't push further; the test is what it is.

## What landed

- 7 commits on `main` (`6f5c4cf..3cffa03`).
- 4 `BindingInGroupTest` + 3 `OrBindingScopeTest` + 2 `NotExistsBindingTest` + 1 extension to `NestedGroupTest` = 10 new tests. 84 → 94 passing.
- Grammar: new `boundOopath` non-terminal, reused by both top-level `rulePattern` and `groupChild`.
- Visitor: new `buildPatternFromBoundOopath` helper; `buildPattern(RulePatternContext)` collapses to a one-line delegator.
- LSP tolerant visitor: `visitBoundOopath` override with real type/bind names (not the `var/_` placeholder used for bare oopaths).
- `docs/DRLXXXX.md` picked up a new "Binding scope in group CEs" paragraph — and, notably, got checked into git for the first time. It had been sitting untracked in the project tree for a while.

What's deliberately still out: OR-branch consensus-rule enforcement (same name on every branch → visible outside), shadowing detection across nesting, `?`-prefixed passive bindings (part of #10), and `$`-prefixed identifier support. DRLXXXX §16 feels done.
