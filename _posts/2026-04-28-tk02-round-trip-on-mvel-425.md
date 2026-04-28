---
layout: post
title: "round-trip on mvel#425 — workaround out, upstream in"
date: 2026-04-28
type: phase-update
entry_type: note
subtype: diary
projects: [drlx-parser]
tags: [mvel3, transpiler, javaparser, if-else]
---

# round-trip on mvel#425 — workaround out, upstream in

Yesterday's [if/else entry](2026-04-28-tk01-if-else-exposed-23-gaps.md) shipped
five gap-fixes, one of them ugly: the visitor's negation used `(cond) == false`
instead of `!(cond)` because MVEL3's transpiler dropped the bean-getter rewrite
somewhere inside `!(...)`. I filed [mvel/mvel#425](https://github.com/mvel/mvel/issues/425)
with a clean reproducer and moved on. Today, trace it.

## The dispatcher had a hole shaped like UnaryExpr

The bug isn't subtle once you know where to look. `MVELToJavaRewriter.rewriteNode`
dispatches on `node.getClass().getSimpleName()` — most cases recurse into
children explicitly, and a `default: rewriteChildren(node)` arm catches anything
unknown. `EnclosedExpr` (parens) has no case — its child gets walked by the
default and everything works.

`UnaryExpr` was different. It *had* a case, but the case only recursed into the
operand for two specific (op, literal-type) combinations:

```java
case "UnaryExpr":
    UnaryExpr unaryExpr = (UnaryExpr) node;
    if (unaryExpr.getOperator() == UnaryExpr.Operator.MINUS ||
        unaryExpr.getOperator() == UnaryExpr.Operator.PLUS) {
        if (unaryExpr.getExpression() instanceof BigDecimalLiteralExpr ||
            unaryExpr.getExpression() instanceof BigIntegerLiteralExpr) {
            // sign-fold, then rewriteNode(unaryExpr.getExpression())
        }
    } else if (unaryExpr.getOperator() == UnaryExpr.Operator.BITWISE_COMPLEMENT) {
        if (unaryExpr.getExpression() instanceof BigIntegerLiteralExpr) {
            // ~bigint, rewriteNode + replace with .not()
        }
    }
    break;  // ← every other path falls through here, no recursion
```

For `LOGICAL_COMPLEMENT` (`!`), neither branch matched, the case fell through
`break`, and the operand — `(p.age > 18)` — was never visited. Same defect for
`-someBean.value`, `~p.somePrimitive`, `++p.x`. The result was silently wrong
output: no compile error from the rewriter, just generated Java referencing a
private field directly and failing `javac` at the next layer.

## Two failing tests, then the fix

Tests first, both red:

```java
@Test
void testGetterRewriteInsideLogicalNot() {
    test( "    var p = new Person(\"yoda\");\n" +
          "    boolean b = !(p.age > 18); \n",
          "    var p = new Person(\"yoda\");\n" +
          "    boolean b = !(p.getAge() > 18);\n");
}

@Test
void testGetterRewriteInsideUnaryMinus() {
    test( "    var p = new Person(\"yoda\");\n" +
          "    int x = -p.age; \n",
          "    var p = new Person(\"yoda\");\n" +
          "    int x = -p.getAge();\n");
}
```

The fix restructures the case so the literal-folding paths short-circuit
explicitly and a final `else` recurses:

```java
case "UnaryExpr":
    UnaryExpr unaryExpr = (UnaryExpr) node;
    if (... MINUS|PLUS && BigLiteral ...) {
        // existing sign-fold
    } else if (... BITWISE_COMPLEMENT && BigInteger ...) {
        // existing ~bigint
    } else if (... BITWISE_COMPLEMENT && BigDecimal ...) {
        throw new UnsupportedOperationException(...);
    } else {
        rewriteNode(unaryExpr.getExpression());
    }
    break;
```

Side-fixes the unary-minus-on-property case and the bitwise-complement-on-
non-BigInteger case along the way. MVEL3 suite green at 722/0. Committed on
the `mvel-425` branch as `[mvel-425] MVEL.map: bean-getter rewrite skipped
inside !(...) — fix UnaryExpr recursion`.

## Reverting the workaround in DRLX

Upstream PR merged, mvel/mvel#425 closed. Now the workaround in
`DrlxToRuleAstVisitor.buildIfElse` could go. Two sites, both replaced:

```java
// before
String negated = "(" + prior + ") == false";
// after
String negated = "!(" + prior + ")";
```

`IfElseParseTest` had assertions hardcoded to the workaround:
`endsWith("== false")`. Updated to `startsWith("!(")` and the symmetric
`doesNotStartWith("!(")` for the active-condition checks. DRLX suite still
141/141. Committed and pushed as `8bcf5c0`; #24 auto-closed.

## Once you write a case, you lose the default's safety net

The `case "UnaryExpr"` failure has a particular shape worth naming. The rewriter
has a `default: rewriteChildren(node)` arm that's always-correct for any node
type that needs no special handling — that's what makes `EnclosedExpr` "just
work." But once you write an explicit `case`, the default doesn't run for that
type any more. If your case forgets to recurse for some operator/operand
combination, the whole sub-tree silently disappears from rewriting.

The repaired case is closer in spirit to what `default` would have done: a final
`else` that recurses into the operand. The literal-folding special cases are
*additions* to the default behaviour, not replacements for it. That's the shape
any explicit case in this dispatcher should have — if you write a case, end it
with the default's behaviour unless you have a reason to short-circuit.

The DRLX workaround was always documented as temporary. It survived one session.
