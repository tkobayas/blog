---
layout: post
title: "not(/a, /b): two gotchas in one morning"
date: 2026-04-22
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [drlx, antlr, drools, group-elements, not]
---

Three issues closed today. #14 (unit-class type inference) got its
last two tasks — a doc update and the issue-close sweep. #7 (tree-shape
LHS refactor) closed alongside. And #8 (`not /pattern`) went from
"single-element landed, multi-element deferred" to fully done, with
`not(/a, /b)` working end-to-end.

Twenty-eight commits pushed to `origin/main`. Sixty-eight tests
passing, zero failures. The work read on paper like it should have
been a quiet landing. It wasn't.

## The first gotcha was labelled alternatives

The spec for multi-element said "grammar-only extension — the infra
was built for N children, this just exercises N > 1." I wrote the
plan in that frame. The grammar change I proposed used ANTLR's
labelled alternatives:

```antlr
notElement
    : NOT oopathExpression ','                                        # singleNot
    | NOT '(' oopathExpression (',' oopathExpression)* ')' ','        # parenNot
    ;
```

Clean, idiomatic, explicit. Every existing visitor override on
`NotElementContext` stopped compiling the moment I saved the file:

```
symbol:   method oopathExpression()
location: variable ctx of type org.drools.drlx.parser.DrlxParser.NotElementContext
```

Labels cause ANTLR to split the context class into per-alternative
subclasses (`SingleNotContext`, `ParenNotContext`). The base
`NotElementContext` becomes abstract and loses the accessor methods.
Three call sites broke — the tolerant LSP visitor, the frozen
visitor, the AST builder. The fix was to drop the labels and keep
the grammar as two unlabelled alternatives. Identical parse behaviour,
consumer code untouched.

I'd written "labelled alternatives are idiomatic in this parser" into
the plan as justification for the approach. It is idiomatic elsewhere —
`oopathRoot` uses them — but those are rules with no shared base-class
consumers. I learned the difference by breaking the build.

## The second gotcha was Drools itself

Grammar fixed, visitor looped over the multi-valued
`ctx.oopathExpression()` list, built one `PatternIR` per child, emitted
`GroupElementIR(NOT, [children...])`. The RED test — `not(/persons[age<18],
/orders[amount>1000])` — ran. It failed with:

```
NOT can have only a single child element.
Either a single Pattern or another CE.
```

The spec I wrote the day before said: *"Drools' `GroupElementFactory.newNotInstance()`
accepts N pattern children and implements NOT-with-AND."* Wrong. Drools'
NOT needs exactly one child. For multi-element semantics you wrap the
children in an AND group first, then attach that AND as the single child
of NOT.

I don't know where I got the "accepts N children" claim from. I don't
think I verified it. The infra story — "we built a tree for N children,
just exercise it" — made the Drools layer feel like an implementation
detail. It wasn't.

The fix is three lines in `DrlxRuleAstRuntimeBuilder.buildLhs`:

```java
if (group.kind() == GroupElementIR.Kind.NOT && group.children().size() > 1) {
    GroupElement andInstance = GroupElementFactory.newAndInstance();
    buildLhs(group.children(), andInstance, ...);
    ge.addChild(andInstance);
} else {
    buildLhs(group.children(), ge, ...);
}
```

Single-child NOT still passes through unchanged. Multi-child NOT gets
its synthetic AND wrapper. The IR stays pure — `GroupElementIR(NOT, [a, b])`
semantically means "NOT applied to these children"; the translation to
Drools' specific tree shape belongs in the runtime builder. That's the
right place for the knowledge.

## What actually survived contact

Decision #1 from the spec — "single-in-parens permissive" — held.
`not(/a)` and `not /a` both parse, both produce the same `GroupElementIR`.
Decision #4 — "no bindings inside `not`" — also held for the paren form
without any new visitor-side check; the grammar already rejects `var p :`
inside `oopathExpression`.

Everything else in the spec that I claimed was "unchanged" needed to
change. The tolerant visitor's `ctx.oopathExpression()` call had to
become `ctx.oopathExpression(0)` because the generated accessor flipped
to a list. The runtime builder needed the AND wrapper. Both are small
fixes. Neither was in the plan.

## Where next

#9 — `exists /pattern` and `exists(/a, /b)` — is the obvious next
landing. Same `GroupElementIR(EXISTS, [children])` shape, same
grammar pattern. The infrastructure is now actually proven for N >= 1
children, not just assumed. I think `exists` can ship both bare and
paren forms in one go; the shape is identical to `not` minus the
semantic inversion. The AND-wrapping in the runtime builder probably
applies there too — Drools' EXISTS likely has the same single-child
constraint. I'll check before writing the spec this time.
