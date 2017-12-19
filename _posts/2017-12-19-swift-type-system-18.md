---
layout: post
title:  Swiftの型システムを読む その18 - ConstraintLocator
---

`ConstraintLocator`について雑メモ。

## ConstraintLocator

`ConstraintLocator`の主な役割は`Expr`がいくつかの`SubExpr`から構成されている場合に「どの部分の`SubExpr`を指すか」を表すもの。
どの部分を指すかの情報は`ConstraintLocator::PathElement` を使って表す。
例えば `(3, (x, 3.14))` のようなタプルの`x`を指したい場合は`"tuple element #1" -> "tuple element #0"` のような感じ。

`PathElement`には`PathElementKind`で表された種類があって、上の例であれば`PathElementKind::TupleElement`となる。

`PathElementKind`は`PathElement::getKind`で取れる。

```cpp
PathElementKind getKind() const { ... }
```

また「何番目の」のような値を0~2つ持つことができ、それが`value1`, `value2`で表す。

```cpp
unsigned getValue() const { ... }

unsigned getValue2() const { ... }
```

`ConstraintLocator`が指している`Expr`は`anchor`という名前で取得できる。

```cpp
Expr *getAnchor() const { return anchor; }
```


## Constraint

`Constraint`は`ConstraintLocator`を持っていて、そこから`getAnchor()` を使ってどの`Expr`に対する`Constraint`かを表すことができる。

```cpp
ConstraintLocator *getLocator() const { return Locator; }
```

```cpp
constraint->getLocator()->getAnchor()
```

## ConstraintLocatorBuilder

```cpp
/// \brief A simple stack-only builder object that constructs a
/// constraint locator without allocating memory.
///
/// Use this object to build a path when passing components down the
/// stack, e.g., when recursively breaking apart types as in \c matchTypes().
```


> メモリをアロケートせずにConstraintLocatorを構築する、単純なスタック専用のBuilderオブジェクト。
>  このオブジェクトを使って、`matchTypes()` のように型を再帰的に分割する場合など、コンポーネントをスタックに渡すときにパスを構築する。

みたい。


## ConstraintSystem

`ConstraintSystem`は`ConstraintLocators`を持っている。

```cpp
llvm::FoldingSetVector<ConstraintLocator> ConstraintLocators;
```

`getConstraintLocator` 経由で`ConstraintLocator`で取ると`ConstraintLocators`にあればそれを取得、なければ作って`ConstraintLocators`に追加してからそれを返す。

```cpp
ConstraintLocator *ConstraintSystem::getConstraintLocator(
                     Expr *anchor,
                     ArrayRef<ConstraintLocator::PathElement> path,
                     unsigned summaryFlags) {
  assert(summaryFlags == ConstraintLocator::getSummaryFlagsForPath(path));

  // Check whether a locator with this anchor + path already exists.
  llvm::FoldingSetNodeID id;
  ConstraintLocator::Profile(id, anchor, path);
  void *insertPos = nullptr;
  auto locator = ConstraintLocators.FindNodeOrInsertPos(id, insertPos);
  if (locator)
    return locator;

  // Allocate a new locator and add it to the set.
  locator = ConstraintLocator::create(getAllocator(), anchor, path,
                                      summaryFlags);
  ConstraintLocators.InsertNode(locator, insertPos);
  return locator;
}
```


## まとめ

主な用途は`getAnchor()`して`Expr`を取り出すことっぽいので、そう覚えておくことにする。
