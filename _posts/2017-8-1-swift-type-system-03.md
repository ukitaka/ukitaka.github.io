---
layout: post
title: Swiftの型システムを読む その3
---

今回は `let a = 42` の型推論がどう行われているかを確かめて行った時に確認できた主要な関数をメモ。

全体の流れを書いておくと
```
swift:: performTypeChecking
 ┗ TypeChecker::typeCheckTopLevelCodeDecl
  ┗ StmtChecker::typeCheckStmt
   ┗ StmtChecker::visit
    ┗ StmtChecker::visitBraceStmt
     ┗ TypeChecker::typeCheckExpression
      ┗ TypeChecker::solveForExpression
       ┗ ConstraintSystem::solve
        ┗ ConstraintSystem::solveRec
         ┗ ConstraintSysten::solveSimplified
          ┗ ConstraintSystem::tryTypeVariableBindings
           ┗ ConstraintSystem::solveRec (再帰)
```


## `swift::performTypeChecking`

ファイルは  `lib/Sema/TypeChecker.cpp` 。
ここが型チェックのエントリーポイント。

##  `TypeChecker::typeCheckTopLevelCodeDecl`

ファイルは `lib/Sema/TypeCheckStmt.cpp` 。
Top levelにコード書いて挙動を確認するときは、`performTypeChecking`からここが呼ばれる。
その2で見た通り、Top levelは`BraceStmt`として扱われるので、実際はここから `StmtChecker::typeCheckStmt` -> `StmtChecker::visit` -> `StmtChecker::visitBraceStmt` の順で`visitBraceStmt`までたどり着く。
そこから中身に応じて `TypeChecker::typeCheckExpression` もしくは `TypeChecker::typeCheckStmt` へ。


##  `TypeChecker::typeCheckExpression`

ファイルは `lib/Sema/TypeCheckerConstraints.cpp`。

コメントで `High-level entry points` と書いてある通り、Exprの型チェックはここから始まる。
基本的には1つのExprに対して1つの `ConstraintSystem` というクラスのオブジェクトが作られるのだが、まさにここで作られている。
制約を生成して、solveできるかまでを行う。実際のもろもろの処理は `TypeChecker::solveForExpression` -> `ConstraintSystem::solve` まで 到達してから行われる。

## `ConstraintSystem::solve`

ファイルは `lib/Sema/CSSolver.cpp`。

ここの処理の流れが型再構築(型推論)の基本的な流れになると思って差し支えない。

前半には `generateConstraints` や `addConstraint`、`listener->generateConstraints` など制約を生成してそうな処理が並んでいる。

制約を生成したのちに  `solve` を行う。その下の`ConstraintSystem::solve` -> `ConstraintSystem::solveRec`  -> `ConstraintSysten::solveSimplified` -> `ConstraintSystem::tryTypeVariableBindings` -> `ConstraintSystem::solveRec` -> … で適切に再帰されて解かれる。
この段階でも制約が追加される場合がある。

`ConstraintSystem::solveSimplified` の呼び出しは、`component` の数に応じて
+ `component`が1つならそのまま`solveSimplified` を呼び出す。
+ `component`が2つならそれぞれに `solveSimplified`を呼び出して解いたのち組み合わせる。

という挙動をする。
ここでいう `component` はグラフ理論におけるcomponent(成分)のことで、 非連結グラフを構成する連結グラフのこと。
実際各制約は `ConstraintGraph`というクラスによってグラフで管理されていて、例えば制約が
+ `T1 == T2`
+ `T3 == T4`
のようになっている場合、componentは2つとなる。


## `ConstraintSystem::solveSimplified`

ファイルは `lib/Sema/CSSolver.cpp`。

後半部分はまだ読めていないが、最初の10~20行くらいがおそらく主な部分で、

```cpp
std::tie(bestBindings, bestTypeVar) = determineBestBindings();
```

```cpp
return tryTypeVariableBindings(solverState->depth, bestTypeVar,
                               bestBindings.Bindings, solutions,
                               allowFreeTypeVariables)
```

のように、各制約の中から 「ある型変数 T に ある制約を加えてみる」「それで`solveRec`からもう一度試してみる」のように試行を繰り返す。

例えば、`T2 can convert to T1` と `T2 conforms to ExpressibleByIntegerLiteral` みたいなのがあった場合、`T2 == Int` みたいな制約が追加されたのち、もう一度解決できるか試す。

そこですべての型変数が解決すると型検査成功となる。

## `ConstraintSystem::tryTypeVariableBindings`

上に書いた通り、 `solveSimplified`でいい感じに決められた型変数、バインディングを `addConstraint` したのち `solveRec` をもう一度呼ぶ。


## `-debug-constraints` の出力を見てみる。

ここまでを踏まえて、`let a = 42` の出力はこんな感じ。

```
% swift -frontend -typecheck -debug-constraints study/s001.swift
---Constraint solving for the expression at [study/s001.swift:1:9 - line:1:9]---
---Initial constraints for the given expression---
(integer_literal_expr type='$T0' location=study/s001.swift:1:9 range=[study/s001.swift:1:9 - line:1:9] value=42)
Score: 0 0 0 0 0 0 0 0 0 0 0 0 0
Type Variables:
  #0 = $T0 [inout allowed]
  #1 = $T1 [inout allowed]

Active Constraints:

Inactive Constraints:
  $T0 literal conforms to ExpressibleByIntegerLiteral [[locator@0x7fcbd9885e00 [IntegerLiteral@study/s001.swift:1:9]]];
  $T0 conv $T1 [[locator@0x7fcbd9885e00 [IntegerLiteral@study/s001.swift:1:9]]];
($T0 literal=3 involves_type_vars bindings=(subtypes of) (default from ExpressibleByIntegerLiteral) Int)
Active bindings: $T0 := Int
(trying $T0 := Int
  ($T1 bindings=(supertypes of) Int)
  Active bindings: $T1 := Int
  (trying $T1 := Int
    (found solution 0 0 0 0 0 0 0 0 0 0 0 0 0)
  )
)
---Solution---
Fixed score: 0 0 0 0 0 0 0 0 0 0 0 0 0
Type variables:
  $T1 as Int
  $T0 as Int

Overload choices:

Constraint restrictions:

Disjunction choices:

Conformances:
  At locator@0x7fcbd9885e00 [IntegerLiteral@study/s001.swift:1:9]
(normal_conformance type=Int protocol=ExpressibleByIntegerLiteral lazy
  (normal_conformance type=Int protocol=_ExpressibleByBuiltinIntegerLiteral lazy))
(found solution 0 0 0 0 0 0 0 0 0 0 0 0 0)
---Type-checked expression---
(call_expr implicit type='Int' location=study/s001.swift:1:9 range=[study/s001.swift:1:9 - line:1:9] arg_labels=_builtinIntegerLiteral:
  (constructor_ref_call_expr implicit type='(_MaxBuiltinIntegerType) -> Int' location=study/s001.swift:1:9 range=[study/s001.swift:1:9 - line:1:9]
    (declref_expr implicit type='(Int.Type) -> (_MaxBuiltinIntegerType) -> Int' location=study/s001.swift:1:9 range=[study/s001.swift:1:9 - line:1:9] decl=Swift.(file).Int.init(_builtinIntegerLiteral:) function_ref=single)
    (type_expr implicit type='Int.Type' location=study/s001.swift:1:9 range=[study/s001.swift:1:9 - line:1:9] typerepr='Int'))
  (tuple_expr implicit type='(_builtinIntegerLiteral: Int2048)' location=study/s001.swift:1:9 range=[study/s001.swift:1:9 - line:1:9] names=_builtinIntegerLiteral
    (integer_literal_expr type='Int2048' location=study/s001.swift:1:9 range=[study/s001.swift:1:9 - line:1:9] value=42)))
```

なんとなく読めるが `Score` などよくわからないものもあったりするので次回以降でもう少し詳細にみる。
