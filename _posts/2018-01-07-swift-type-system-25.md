---
layout: post
title:   Swiftの型システムを読む その25 - PotentialBindingの決定
---

前回は`PotentialBinding`の周辺実装をみたので、今回はSolveアルゴリズムの根幹をなす `ConstraintSystem::getPotentialBindings` を読んでみる。

ファイルは`lib/Sema/CSBindings.cpp`

## ConstraintSystem::getPotentialBindings 概要
```cpp
/// \brief Retrieve the set of potential type bindings for the given
/// representative type variable, along with flags indicating whether
/// those types should be opened.
ConstraintSystem::PotentialBindings
ConstraintSystem::getPotentialBindings(TypeVariableType *typeVar) { ... }
```

型変数を受け取って`PotentialBindings`を返す。受け取る型変数は代表元である必要があるっぽい。

`result`に対してPotentialBindingを追加したりゴニョゴニョしたりして、それをreturnする。

```cpp
PotentialBindings result(typeVar);

// resultに対してなにか処理 ...

return result;
```

## ConstraintGraphから集めたConstraintに対する操作

まず、ConstraintGraphからその型変数を持つConstraintを集めてくる。

```cpp
getConstraintGraph().gatherConstraints(typeVar, constraints, ConstraintGraph::GatheringKind::EquivalenceClass);
```

そこで`Kind`に応じて前処理的と共通の処理を行う。

```cpp
for (auto constraint : constraints) {
  // 前処理。ここで必要ければスキップすることもある。
  switch (constraint->getKind()) {
    ...
  }
  // 共通処理
}
```

メインの目的は「PotentialBindingを追加すること」で、主に2つの方法で追加される。

+ `ConstraintKind::LiteralConformsTo`の場合はそのリテラルを表すprotocolを元にDefaultType(例えば、`ExpressibleByIntegerLiteral`なら`Int`)を得て、それをPotentialBindingとして登録する
+ `Subtype`や`Conversion`の場合は、その型自身へのPotentialBindingを登録する。

### ConstraintKind::LiteralConformsToの場合

重要な部分を抽出するとこんな感じ。

```cpp
case ConstraintKind::LiteralConformsTo: {
  // ここでDeaultTypeを取得
  auto defaultType = tc.getDefaultType(constraint->getProtocol(), DC);

  // literalProtocolsに追加(あとで整理するときに使う)
  literalProtocols.insert(constraint->getProtocol());

  // ここでresult記録
  result.foundLiteralBinding(constraint->getProtocol());
  result.addPotentialBinding({defaultType, AllowedBindingKind::Subtypes,
                        constraint->getKind(),
                        constraint->getProtocol()});
```

`getDefaultType`は`ExpressibleByIntegerLiteral`は`Int`など、リテラルがデフォルトで取りうる型を決める関数。

```cpp
// ExpressibleByIntegerLiteral -> IntegerLiteralType
else if (protocol == getProtocol(SourceLoc(), KnownProtocolKind::ExpressibleByIntegerLiteral)) {
    type = &IntLiteralType;
    name = "IntegerLiteralType";
  }
```

`IntegerLiteralType`などの名前はSwift側で`typealias`が貼られている。

```cpp
/// The default type for an otherwise-unconstrained integer literal.
public typealias IntegerLiteralType = Int
```

### ConstraintKind::Subtypeなどの場合

簡単に言うと「型Tのサブクラス」という制約があったら「TへのPotentialBinding」を試す。
Constraintのどっちが型変数かどうかで`Subtypes`か`Supertypes`のどちらで使えるか決まる。

```cpp
// 1つめが型変数の場合。
// T < MyType みたいな。場合はsubtypeに使えるbindingとして記録
if (first->getAs<TypeVariableType>() == typeVar) {
    // Upper bound for this type variable.
    type = second;
    kind = AllowedBindingKind::Subtypes;
}
// 2つめが型変数の場合。
// MyType < T みたいな。場合はsupertypeに使えるbindingとして記録
else if (second->getAs<TypeVariableType>() == typeVar) {
    // Lower bound for this type variable.
    type = first;
    kind = AllowedBindingKind::Supertypes;
}
```

```cpp
 result.addPotentialBinding({type, kind, constraint->getKind()}, /*allowJoinMeet=*/!adjustedIUO);
```

Lvalue, InOut, Optionalなどの細かい調整もここで行われる。

## PotentialBindingsの整理

ConstraintGraphから集めたConstraintを一通り見終わったら、`result`の整理を行う。主に4つの処理が行われる。詳細略。

+ `LiteralProtocol`は`Defaulted`なProtocolより優先される。`LiteralProtocol`がある場合は`DefaultedProtocol`は削除される。
+ 上のfor文で記録した`defaultableConstraints`がまだ追加されていなければPotentialBindingとして追加する
+ `addOptionalSupertypeBindings`フラグが立っている場合はTへのBindingをT?へのBindingへ書き換える処理がされる。
+ 最後に、ここまでで`FullyBound`でない場合はBindingをすべて消す。


## まとめ

メインの戦略は「リテラルをデフォルトの型にする」「サブタイプなどをその型自身にする」などみたい。
