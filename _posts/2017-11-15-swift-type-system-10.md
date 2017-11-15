---
layout: post
title:  A generic algorithm for checking exhaustivity of pattern matching を読んだ
---

# Swiftの型システムを読む その10 - switch文の網羅チェック

[前回の記事](https://blog.waft.me/2017/11/12/pattern-match-exhausitivity/)でSpaceを使った網羅チェックのアルゴリズムを見たので、今回はswiftコンパイラに置けるその実装を見てみる。

なお、アルゴリズム・用語・記号の説明はほとんど省略するので、必要に応じて [元の論文](https://infoscience.epfl.ch/record/225497)や[前回の記事](https://blog.waft.me/2017/11/12/pattern-match-exhausitivity/)を参照。

Swiftのバージョンは4.0.2。


## Swiftのswitchと網羅チェック

`TypeCheckSwitchStmt.cpp` のコメントに網羅チェックと警告について参考にした論文が紹介されている。

```cpp
/// The SpaceEngine encapsulates an algorithm for computing the exhaustiveness
/// of a switch statement using an algebra of spaces described by Fengyun Liu
/// and an algorithm for computing warnings for pattern matching by
/// Luc Maranget.
///
/// The main algorithm centers around the computation of the difference and
/// the intersection of the "Spaces" given in each case, which reduces the
/// definition of exhaustiveness to checking if the difference of the space
/// 'S' of the user's written patterns and the space 'T' of the pattern
/// condition is empty.
```

	- [Liu, Fengyun 2016 - A generic algorithm for checking exhaustivity of pattern matching](https://infoscience.epfl.ch/record/225497)
	- [L Maranget - Journal of Functional Programming, 2007 - Warnings for pattern matching](http://moscova.inria.fr/~maranget/papers/warn/index.html)


## 網羅チェックはいつ行われるか？

switch文の型チェックの直後に行われる。たとえばcaseに全然関係ない型がある場合などは型チェック時に警告がでて、網羅チェックは限定的にしか行わない。なので基本的には

```swift
let a = 1
switch a {
case is String: // ここで"型チェック時に"警告
}
```

```cpp
TC.checkSwitchExhaustiveness(S, /*limitChecking*/hadError);
```

## spaceの実装

まずはspaceを構成する𝒪, 𝒯(T), 𝒦(K, s1, s2, ..., sn)あたりの実装がどのように行われているかをみてみる。

Spaceはそのまま `Space` というクラスによって実装されている。

```cpp
class Space final {　
    ...
}
```

swiftでの実装ではSpaceは`SpaceKind`によって分類されており、`BooleanConstant`が特別扱いされているのが特徴的。

```cpp
enum class SpaceKind : uint8_t {
    Empty           = 1 << 0, // 空Space 𝒪
    Type            = 1 << 1, // 𝒯(T)
    Constructor     = 1 << 2, // 𝒦(K, s1, s2, ..., sn) 
    Disjunct        = 1 << 3, // s1 | s2 | s3 
    BooleanConstant = 1 << 4,  // true or false
}
```


上記それぞれの`SpaceKind`に対して`Space`のコンストラクタがあって、例えば `𝒯(T)` にあたるものは以下のように定義されている。

```cpp
explicit Space(Type T, Identifier NameForPrinting)
    : Kind(SpaceKind::Type), TypeAndVal(T, false), Head(NameForPrinting),
    Spaces({}){}
```


## パターンのspace

パターンからSpaceへの射影 𝒫(p) は そのまま各caseに対応する`Pattern`を受けとって`Space`を返す`projectPattern` という関数になっている。 

```cpp
static Space projectPattern(TypeChecker &TC, const Pattern *item, bool &sawDowngradablePattern)
```

`PatternKind`で定義された13種類の`Pattern`に対してそれぞれの𝒫に関する規則が定義されている。いくつかピックアップしてみると

```
𝒫(_) = 𝒯(T)         // マッチさせたい型そのもの
𝒫(true) = 𝒯(true)   // trueはtrue
𝒫(false) = 𝒯(false) // falseはfalse
𝒫(e) = 0             // その他の式(例えば1, 2.0など) はempty
𝒫(_: T) = 0          // 型に関するパターンもempty
𝒫(is T) = 𝒯(T)       // isはその型

// enumのcaseはconstructorに
𝒫(.enumCase(a, b, c)) = 𝒦(.enumCase, s1, s2, ..., sn)
```


## 分解(Decompose)の実装

型Tをsubspaceのunionに分解する𝒟(T)と、分解可能かチェックする𝒟? (T)もそのまま`decompose`, `canDecompose` という関数になっている。

```cpp
static void decompose(TypeChecker &TC, Type tp,
                      SmallVectorImpl<Space> &arr) { ... }
static bool canDecompose(Type tp) { ... }
```

実装を見ると分かる通り、分解できるのは(swiftでは) `Bool`、 タプル、`enum` のみ。

```cpp
static bool canDecompose(Type tp) {
  return tp->is<TupleType>() || tp->isBool()
      || tp->getEnumOrBoundGenericEnum();
}
```

規則について整理すると、

```
𝒟?(Bool) = true
𝒟?((T1, T2, ...)) = true
𝒟?(EnumType) = true
𝒟?(x) = false
```

```
𝒟(Bool) = { true, false }
𝒟((T1, T2, ...)) = 𝒦("", 𝒯(T1), 𝒯(T2), ...)
𝒟(EnumType) = 𝒦(K1, ...) | 𝒦(K2, ...) |  …
```


## subtraction (⊖) の実装

Swiftの実装では `Space::minus` として実装されている。

```cpp
Space minus(const Space &other, TypeChecker &TC) const { ... }
```

最初のいくつかピックアップして規則を確認すると、ほぼそのまま実装されていることがわかる。

```
s ⊖ 0 = s
0 ⊖ s = 0
𝒯(T) ⊖ x = 𝒟(T) ⊖ x if 𝒟?(T)
x ⊖ (s1 | s2 | ···) = x ⊖ s1 ⊖ s2 ⊖ ...
```

特筆すべきところは特にないが、一応(Swift特有の)Boolはこんな感じ。

```cpp
PAIRCASE (SpaceKind::BooleanConstant, SpaceKind::BooleanConstant): {
  // The difference of boolean constants depends on their values.
  if (this->getBoolValue() == other.getBoolValue()) {
    return Space();
  } else {
    return *this;
  }
}
```

## intersection(⊓)の実装
こちらも `Space::intersect` として実装されている。

```cpp
Space intersect(const Space &other, TypeChecker &TC) const { ... }
```

```
s ⊓ 0 = 0
0 ⊓ s = 0
x ⊓ (s1 | s2 | ···) = (x ⊓ s1) | (x ⊓ s2) | ...
```

こちらもだいたいそのままなので略。

## subspace関係 (≼) の実装
論文通りならsubtractionを使って

```
s1 ≼ s2 if s1 ⊖ s2 = 0
```
のはずだが

```cpp
// An optimization that computes if the difference of this space and
// another space is empty.
bool isSubspace(const Space &other, TypeChecker &TC) const { ... }
```

とのことで各組み合わせについて愚直に実装してある。でも基本はsubstractionを使うのものと同じ(はず)

## 網羅チェックの実装
`SpaceEngine::checkExhaustiveness`がそれです。
引数の`limitedChecking` はすでに前段のSwitchStmtに対しての型チェックが失敗しているときにtrueが来る。

```cpp
void checkExhaustiveness(bool limitedChecking) { ... }
```


`SpaceEngine` が `SwitchStmt` の参照を持っていて、そこからマッチさせたい型や各case(パターン)などを取り出して使う。

```cpp
SwitchStmt *Switch;
```

```cpp
// マッチさせたい型
auto subjectType = Switch->getSubjectExpr()->getType();
```

```cpp
// 各caseを取り出して使う
for (auto *caseBlock : Switch->getCases()) {
    ...
}
```

ここがコアの部分です、。確かに`s1 ⊖ s2 = 0`をそのまま実装しているけど、結局`isSubspace`をつかってないのはなぜ。。。。

```cpp
auto uncovered = totalSpace.minus(coveredSpace, TC).simplify(TC);
if (uncovered.isEmpty()) {
  return;
}
```

## where句を持つcaseについて
これも理論通り網羅チェックには使えないことが明記されている。

```cpp
// 'where'-clauses on cases mean the case does not contribute to
// the exhaustiveness of the pattern.
if (caseItem.getGuardExpr())
  continue;
```


## その他の用語

いくつか論文中には出てこないものが実装されているので簡単にメモしておく。

- `isUseful` メソッドはこのSpaceが網羅チェックに有用かどうかを表す。空Spaceか、空Spaceのunionなどだとfalseが返る。warningに関する論文の方できちんと定義されているっぽい。

- `computeSize`でSpaceのサイズを返す。空Spaceはサイズ0, 𝒯(T)はサイズ1として、𝒦やunionの場合はそれらを構成するSpaceのサイズの合計になる。何に使われるかというとサイズがでかすぎる(= 複雑すぎる)場合にエラーにするようにしている。
- `canDowngrade` は`@_downgrade_exhaustivity_check`という[特定のcaseを網羅チェックから外す](https://github.com/apple/swift/blob/b115ae528679ddc953fb2f33ce4f1fb56e8f2502/test/stmt/nonexhaustive_switch_stmt_editor.swift#L8-L29)のに使えるアトリビューション用っぽいが、[Swift4では使えない](https://github.com/apple/swift/blob/815d82c9102b54c705b773b7d3e4653972fae713/lib/Sema/TypeCheckSwitchStmt.cpp#L1172-L1176))みたい。

## まとめ

Swiftには珍しく(?) きちんと参考論文が書いてあった&ほぼ論文通り実装されていたのと、1ファイルでほぼ閉じているのでとても読みやすかった。

あと、[dottyの方の実装](https://github.com/lampepfl/dotty/blob/master/compiler/src/dotty/tools/dotc/transform/patmat/Space.scala)をみたらメソッド名などがほとんど同じでこれを参考にして実装したっぽい？？

