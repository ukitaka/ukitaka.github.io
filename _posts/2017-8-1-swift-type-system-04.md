---
layout: post
title: Swiftの型システムを読む その4
---

今回は `-debug-constraints` で出力される `Score` について調べる。

こんなの

```
Score: 0 0 0 0 0 0 0 0 0 0 0 0 0
```

簡単にまとめると、 `Score`は解がどれくらい最適かを表すもの。

 `0 0 0 0 0 0 0 0 0 0 0 0 0`に近い (つまり小さい) 方が良い解であることを表していて、どの解を採用するか検討する時に使うものと思われる。

それぞれの `0` は`ScoreKind` に定義されたものに対応している。

```cpp
/// Describes an aspect of a solution that affects its overall score, i.e., a
/// user-defined conversions.
enum ScoreKind {
  // These values are used as indices into a Score value.

  /// A reference to an @unavailable declaration.
  SK_Unavailable,
  /// A fix needs to be applied to the source.
  SK_Fix,
  /// An implicit force of an implicitly unwrapped optional value.
  SK_ForceUnchecked,
  /// A user-defined conversion.
  SK_UserConversion,
  /// A non-trivial function conversion.
  SK_FunctionConversion,
  /// A literal expression bound to a non-default literal type.
  SK_NonDefaultLiteral,
  /// An implicit upcast conversion between collection types.
  SK_CollectionUpcastConversion,
  /// A value-to-optional conversion.
  SK_ValueToOptional,
  /// A conversion to an empty existential type ('Any' or '{}').
  SK_EmptyExistentialConversion,
  /// A key path application subscript.
  SK_KeyPathSubscript,
  /// A conversion from a string, array, or inout to a pointer.
  SK_ValueToPointerConversion,

  SK_LastScoreKind = SK_ValueToPointerConversion,
};
```


ここに書いてあることをせずに解が見つかる方が望ましく、これらが発生するたびにScoreが +1される。
たとえば一番わかりやすそうな `SK_ValueToOptional` あたりを見て見る。

## SK_ValueToOptional

Swiftでは `A` -> `Optional<A>` への暗黙変換があるので例えば

```swift
let a: Int? = 42
```

は型チェックが通る。この暗黙変換込みで解が見つかった場合はここのScoreが+1される。


`-dump-constraints` で見てみると

```
---Constraint solving for the expression at [study/s0007.swift:1:15 - line:1:15]---
---Initial constraints for the given expression---
(integer_literal_expr type='$T0' location=study/s0007.swift:1:15 range=[study/s0007.swift:1:15 - line:1:15] value=1)
Score: 0 0 0 0 0 0 0 0 0 0 0 0 0
Contextual Type: Int? at [study/s0007.swift:1:8 - line:1:11]
Type Variables:
  #0 = $T0 [inout allowed]

Active Constraints:

Inactive Constraints:
  $T0 literal conforms to ExpressibleByIntegerLiteral [[locator@0x7f8f4e0d9800 [IntegerLiteral@study/s0007.swift:1:15]]];
  $T0 conv Int? [[locator@0x7f8f4e0d9800 [IntegerLiteral@study/s0007.swift:1:15]]];
($T0 literal=3 bindings=(subtypes of) Int)
Active bindings: $T0 := Int
(trying $T0 := Int
  (increasing score due to value to optional) // !!!・・・(1)
  (found solution 0 0 0 0 0 0 0 1 0 0 0 0 0)
)
---Solution---
Fixed score: 0 0 0 0 0 0 0 1 0 0 0 0 0
Type variables:
  $T0 as Int

Overload choices:

Constraint restrictions:
  Int to Optional<Int> is [value-to-optional] // !!!・・・(2)

Disjunction choices:

Conformances:
  At locator@0x7f8f4e0d9800 [IntegerLiteral@study/s0007.swift:1:15]
```


`found solution` の `SK_ValueToOptional` の部分が1になっていることが確認できる。
(1) のところでログが出ていて `increasing score due to value to optional`となっている。つまり `tryTypeVariableBindings` で`Binding`の制約が追加されたタイミングでScoreが付いているようだ。
ちょっと長いので呼び出しだけ整理すると、

```
ConstraintSystem::tryTypeVariableBindings
 ┗ ConstraintSystem::addConstraint
  ┗ ConstraintSystem::addConstraintImpl
   ┗ ConstraintSystem::matchTypes
    ┗ enumerateOptionalConversionRestrictions (ここでRestriction追加)
    ┗ ConstraintSystem::simplifyRestrictedConstraint
      ┗ ConstraintSystem::simplifyRestrictedConstraintImpl
       ┗ ConstraintSystem::increaseScore (+1)
```

こんな感じ。

## Scoreはどこで使われるか？

`ConstraintSystem` は `CurrentScore` と `BestScore` を状態として持っていて、`ConstraintSystem::worseThanBestSolution()` というメソッドで今解いた解が最適かどうかを判別できる。

```cpp
bool ConstraintSystem::worseThanBestSolution() const {
  if (retainAllSolutions())
    return false;

  if (!solverState || !solverState->BestScore ||
      CurrentScore <= *solverState->BestScore)
    return false; //いまのScoreがいまのより良い .つまり低い方がよい。

  if (TC.getLangOpts().DebugConstraintSolver) {
    auto &log = getASTContext().TypeCheckerDebug->getStream();
    log.indent(solverState->depth * 2)
      << "(solution is worse than the best solution)\n";
  }

  return true; // いまのScoreがBestより悪い
}
```


こんな感じで、いま解いた解がよくなかったらSKIPする、などに使われている。

```cpp
// If this solution is worse than the best solution we've seen so far,
// skip it.
if (worseThanBestSolution())
  return true;
```



