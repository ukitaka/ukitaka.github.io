---
layout: post
title:   Swiftの型システムを読む その24 - PotentialBinding周辺の用語・関数
---

Swiftの型推論では、型変数に適切なBindingを決めて試すことを繰り返す
ことで制約の単一化を行う。今回はその際に使われる`PotentialBinding`やそれに関連する関数を見ていく。ファイルとしては`CSBindings.cpp`。

## PotentialBinding / PotentialBindings

型変数が取りうる具体的な型へのBindingを`PotentialBinding`という構造体で表している。

```cpp
/// A potential binding from the type variable to a particular type,
/// along with information that can be used to construct related
/// bindings, e.g., the supertypes of a given type.
struct PotentialBinding {
  /// The type to which the type variable can be bound.
  Type BindingType;
  ...
}
```

型変数が取りうるBindingは複数考えられるので、`PotentialBindings`という構造体に型変数と具体的な型の配列を持っている。

```cpp
struct PotentialBindings {
  TypeVariableType *TypeVar;

  /// The set of potential bindings.
  SmallVector<PotentialBinding, 4> Bindings;
  ...
}
```

また、その型変数を持っているConstraintを`BindingSource`という名前で`PotentialBining`に持っている。

```cpp
/// The kind of the constraint this binding came from.
ConstraintKind BindingSource;
```

## AllowedBindingKind

PotentialBindingには「そのBindingが適用できる条件」のようなものがある。それが`AllowedBindingKind`で表されている。

```cpp
/// The kind of bindings that are permitted.
enum class AllowedBindingKind : unsigned char {
  /// Only the exact type.
  Exact,
  /// Supertypes of the specified type.
  Supertypes,
  /// Subtypes of the specified type.
  Subtypes
};
```

それぞれ「完全に一致する型として」「スーパータイプとして」「サブタイプとして」使えることを表している。
これを`PotentialBinding`が`Kind`という形で持っている。

```cpp
 /// The kind of bindings permitted.
 AllowedBindingKind Kind;
```

## DefaultedProtocol

`Constarint`が以下のKindを持つ場合に`ProtocolDecl`が取得できる。

```cpp
ProtocolDecl *Constraint::getProtocol() const {
  assert((Kind == ConstraintKind::ConformsTo ||
          Kind == ConstraintKind::LiteralConformsTo ||
          Kind == ConstraintKind::SelfObjectOfProtocol)
          && "Not a conformance constraint");
  return Types.Second->castTo<ProtocolType>()->getDecl();
}
```

この`Constraint`が型変数を含み、それを`PotentialBinding`として採用する場合、`PotentialBinding`に`DefaultedProtocol`という名前で保持される。

```cpp
/// The defaulted protocol associated with this binding.
Optional<ProtocolDecl *> DefaultedProtocol;
```

## DefaultableConstraint

`ConstraintKind::Defaultable` を持つConstraintは例えば以下のようなときに使われる。

+ `return`文が省略されたClosure式のの返り値の型はデフォルトで`()` 
+ Arrayリテラルがヘテロな場合など、ArrayのElementの型はデフォルトで`Any`

この制約を元に`PotentialBinding`が作られる場合、`DefaultableBinding`に記録される。

```cpp
/// If this is a binding that comes from a \c Defaultable constraint,
/// the locator of that constraint.
ConstraintLocator *DefaultableBinding = nullptr;
```

この`PotentialBinding`が採用される場合、`ConstraintSystem`の`DefaultedConstraints`に記録される。

```cpp
addConstraint(ConstraintKind::Bind, typeVar, type,
              typeVar->getImpl().getLocator());

// If this was from a defaultable binding note that.
if (binding.isDefaultableBinding()) {
      DefaultedConstraints.push_back(binding.DefaultableBinding);
}
```


## ConstraintSystem::PotentialBindings::addPotentialBinding

```cpp
void ConstraintSystem::PotentialBindings::addPotentialBinding(
    PotentialBinding binding, bool allowJoinMeet) { ... }
```

基本的には`PotentialBindings`の`Bindings`に追加するのだが、同じ型変数への`Supertype`の`PotentialBinding`が追加された場合は結び(join)を計算してから追加する。

## ConstraintSystem::determineBestBindings

名前の通り`Best`な`PotentialBindings`を決める。`PotentialBinding`ではないことに注意、つまり「どの型変数に対してBindingを決めるのが良いのか」を決めている。

```cpp
Optional<ConstraintSystem::PotentialBindings>
ConstraintSystem::determineBestBindings() { ... }
```

決める方法は`BindingScore`というスコアで決めている。`ConstraintSystem`の`Score`とは別。

```cpp
typedef std::tuple<bool, bool, bool, bool, unsigned char, unsigned int>
    BindingScor
```

少しだけ中身を見ると、`Defaultable`なBinding以外のBindingがあるか、型変数の有無等によってスコアが決まっているみたい。

```cpp
static BindingScore formBindingScore(const PotentialBindings &b) {
  return std::make_tuple(!b.hasNonDefaultableBindings(),
                         b.FullyBound,
                         b.SubtypeOfExistentialType,
                         b.InvolvesTypeVariables,
                         static_cast<unsigned char>(b.LiteralBinding),
                         -(b.Bindings.size() - b.NumDefaultableBindings));
}
```

## ConstraintSystem::getPotentialBindings

上の`determineBestBindings`の中で使われる。ある型変数に対する`PotentialBindings`を取得する。詳細な実装は次回以降読む。

```cpp
/// \brief Retrieve the set of potential type bindings for the given
/// representative type variable, along with flags indicating whether
/// those types should be opened.
ConstraintSystem::PotentialBindings
ConstraintSystem::getPotentialBindings(TypeVariableType *typeVar) { ... }
```


ざっくりとは
+ ConstraintGraphから必要な制約を集めてくる
+ リテラルやDefaultableなものを元に`PotentialBindings`を作る。

## まとめ

とりあえずざっと読んでみてわからなかった用語などをまとめてみた。
`getPotentialBindings`が`CSBinding.cpp`のほとんどを占め、型推論の根幹でもありあそうなので、次回ちゃんと読む。


