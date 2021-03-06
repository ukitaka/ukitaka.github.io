---
layout: post
title:   Swiftの型システムを読む その21 - UnresolvedDotExprの型推論(Simplify~Expr書き換え)
---

## 前回の復習

1. オーバーロードの候補が1つなら直ちに`resolveOverload`をする
2. オーバーロードの候補が2つ以上なら`Disjunction`な制約生成後に、`simplify`のタイミングで`resolveOverload`する。

今回はこのあたりを中心に確認する。

+ Disjunctionからどうやってどれを使うか決めているか
+ `resolveOverload`がしたら何が起こるか
+ `Expr`書き換え時の挙動は？

## ConstraintSystem::solveSimplified

`solve`系の関数がいくつかあってややこしいが、`solveSimplified`は`solveRec`から呼ばれる関数。主な目的は型変数に具体的な型をバインディングすることみたい。ここでDisjunctionのうちの1つが選ばれる。

```cpp
// Disjunctionをバラす
auto constraints = disjunction->getNestedConstraints();

// 一つ一つ取り出して、DisjunctionChoiceを作る。
for (auto index : indices(constraints)) {
    auto currentChoice = DisjunctionChoice(this, constraints[index]);
    // ....
```

`DisjunctionChoice`に対して`solve`が呼ばれる。ここからまた`simplify`されたり`solveRec`されたりする。

```cpp
if (auto score = currentChoice.solve(solutions, allowFreeTypeVariables)) {
  // ...
}
```

```cpp
Optional<Score>
DisjunctionChoice::solve(SmallVectorImpl<Solution> &solutions,
                         FreeTypeVariableBinding allowFreeTypeVariables) {
  CS->simplifyDisjunctionChoice(Choice);
  bool failed = CS->solveRec(solutions, allowFreeTypeVariables);
  return failed ? None : Optional<Score>(CS->CurrentScore);
}
```


`simplifyDisjunctionChoice`ではさらに`simplifyConstraint`にたどり着いて結果`resolveOverload`される。

```cpp
case ConstraintKind::BindOverload:
  resolveOverload(constraint.getLocator(), constraint.getFirstType(),
                  constraint.getOverloadChoice(),
                  constraint.getOverloadUseDC());
  return SolutionKind::Solved;
```

## ConstraintSystem::resolveOverload

上で`resolveOverload`が呼ばれると、それはシンプルな`Bind`に置き換えられる。

```cpp
// Add the type binding constraint.
addConstraint(ConstraintKind::Bind, boundType, refType, locator);
```

+ `boundType`は`ConstraintGenerator::visitUnresolvedXXX`で作られた型変数を指す。
+ `refType`はその関数/メソッド/メンバーの型

また、`ConstraintSystem`の`resolvedOverloadSets`に記録され、`Expr`書き換えのタイミングで使用される。

```cpp
resolvedOverloadSets
    = new (*this) ResolvedOverloadSetListItem{resolvedOverloadSets,
                                              boundType,
                                              choice,
                                              locator,
                                              openedFullType,
                                              refType};
```


## Expr書きかえ

`Expr`書き換えでは、`getOverloadChoiceIfAvailable` `getOverloadChoice`などの関数経由で、上で記録した`resolvedOverloadSets`が取得される。
それを使って`buildMemberRef`される。

```cpp
Expr *visitMemberRefExpr(MemberRefExpr *expr) {
  auto memberLocator = cs.getConstraintLocator(expr,
                                               ConstraintLocator::Member);
  auto selected = getOverloadChoice(memberLocator);
  bool isDynamic
    = selected.choice.getKind() == OverloadChoiceKind::DeclViaDynamic;
  return buildMemberRef(expr->getBase(),
                        selected.openedFullType,
                        expr->getDotLoc(),
                        selected.choice.getDecl(), expr->getNameLoc(),
                        selected.openedType,
                        cs.getConstraintLocator(expr),
                        memberLocator,
                        expr->isImplicit(),
                        selected.choice.getFunctionRefKind(),
                        expr->getAccessSemantics(),
                        isDynamic);
}
```

(タイトルは`UnresolvedDotExpr`のになってるけど、短くてわかりやすいので代わりに`visitMemberRefExpr`を載せてます。ロジックはだいたい同じです。)

## まとめ

+ ConstraintSystem::solveSimplifiedで1つ選ばれる
+ ConstraintSystem::resolveOverloadで型変数がバインドされ、書き換えのために記録される。
+ 記録された情報を元に、ASTを組み立てる
