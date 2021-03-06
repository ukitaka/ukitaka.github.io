---
layout: post
title:  Swiftの型システムを読む その16 - MetatypeとSubtyping
---

ジェネリクス周りのコードリーディングに苦戦しているのでちょっと話題を戻して再びサブタイピングについて。

## Metatypeとは？

Swiftでは`T.self`, `T.Type` など、型自体を値として扱うときにでてくるやつら。
`T.self`という式の型が`T.Type`であり、`T.Type`のことを「メタタイプ」と呼ぶ。

![図](https://img.esa.io/uploads/production/attachments/2245/2017/11/23/2884/2d2107ca-a5d5-416c-897e-4286fe7cd0f5.svg)

`MetatypeType`と`ExistentialMetatypeType`の2つがあり、

+ `P`がprotocolのとき、`P.Type`は`ExistentialMetatypeType`
+  `P`がprotocolのとき、`P.Protocol`は`MetatypeType`
+ それ以外の`T.Type`は`MetatypeType`

となる。また、`型.self`のような構文でメタタイプに型付けされる値を表現できるが、protocolとそれ以外で微妙にセマンティクスが違うことに注意。

+ `P`がprotocolのとき、`P.self`は`P.Protocol`型の値。
+ それ以外の`T.self`は`T.Type`型の値。

また「メタタイプのメタタイプの….」のようにメタタイプ自体もメタタイプを持つ。もう何を言ってるのかも自分でもわからなくなってきたけど、

```swift
protocol P { }
P.self // P.Protocol
P.Type // ExistentialMetatypeType
P.Protocol // MetatypeType
P.Type.Protocol // ExistentialMetatypeTypeのメタタイプ
P.Type.Type.Protocol // ExistentialMetatypeTypeのExistentialMetatypeTypeのメタタイプ
```

実装としては上のクラス図を頭に入れとけば概ね問題ないが、1点`AnyMetatypeType::getInstanceType`というメソッドで`T.Type`の`T`を得ることができるということだけ抑えておけばよさそう。

```cpp
meta->getInstanceType()
```

今回はこのややこしそうなメタタイプについてのサブタイピング規則を確認しておく。

## superclass関係に基づくサブタイピング規則

[lib/Sema/CSSimplify.cpp](https://github.com/apple/swift/blob/master/lib/Sema/CSSimplify.cpp)によると、classのメタタイプには以下のようなサブタイプ関係がある。

```cpp
// A.Type < B.Type if A < B and both A and B are classes.
```

つまりクラスのメタタイプはクラスのサブタイプ関係について共変である。
例えば以下のようにクラス`Animal`と`Dog`があったとすると、`Dog.Type < Animal.Type`が成立する。

```swift
class Animal { }
class Dog: Animal { }

// いずれもtrue
Dog() is Animal         // Dog < Animal
Dog.self is Animal.Type // Dog.Type < Animal.Type
```


## Protocolのサブタイプ関係に基づくサブタイピング規則

再び[lib/Sema/CSSimplify.cpp](https://github.com/apple/swift/blob/master/lib/Sema/CSSimplify.cpp)によると、protocolのメタタイプには以下のようなサブタイプ関係がある。

```cpp
// P.Type < Q.Type if P < Q, both P and Q are protocols, and P.Type and Q.Type are both existential metatypes.
```

「P < QのときにP.Type < Q.Type」なのであって、「P < QのときにP.Protocol < Q.Protocol」ではないことに注意。[参考](https://github.com/apple/swift/blob/master/test/Constraints/existential_metatypes.swift#L28-L29)

これを再現できる良い例が思い浮かばないけど、一応以下から確認できる。

```swift
func undefined<T>() -> T {
  fatalError()
}

func test() {
  let a: PP.Type = undefined()
  let b: P.Type = a // OK
}
```



## ConversionRestrictionKind::MetatypeToExistentialMetatypeに基づくサブタイピング規則

前回説明した`ConversionRestriction`(セマンティクスに影響するサブタイピング)にもメタタイプに関するものがある。

名前の通り`Metatype`と`ExistentialMetatype`間のサブタイピングで、規則はコメント中には書いてないけど、あえて書くならこんな感じ。

```
T.Type < P.Type if T: P
```

例としては以下のような場合。protocolのMetatypeとはサブタイプ関係はないので注意。

```swift
protocol Animal { }
struct Dog: Animal { }
Dog.self is Animal.Type // true
Dog.self is Animal.Protocol // false
```

これ地味にハマるところで、この例だとまだわかりやすいけど、例えばジェネリックな関数に渡すとぱっと見た感じは同じだが、挙動に違いが出る。

```swift
func isSubtypeOf<A, B>(_ a: A.Type, _ b: B.Type) -> Bool {
  return A.self is B.Type
}

Dog.self is Animal.Type // true
isSubtypeOf(Dog.self, Animal.self) // false
```

なぜかというと、関数の引数として渡している`Animal.self`の型は`Animal.Protocol`であり、その状態で`isSubtypeOf`の型パラメータがopenされると `B.Type`が`Animal.Protocol`を指すことになりfalseになる。

```
---Solution---
Fixed score: 0 0 0 0 0 0 0 0 0 0 0 0 0
Type variables:
  $T1 as Dog
  $T0 as (Dog.Type, Animal.Protocol) -> Bool // ここ
  $T2 as Animal	
```


## その他の規則

あとはマイナーなものしかないので参考程度にメモ。

```cpp
// Existential-metatype-to-superclass-metatype conversion.
if (type2->is<MetatypeType>()) {
  if (auto *meta1 = type1->getAs<ExistentialMetatypeType>()) {
    if (meta1->getInstanceType()->isClassExistentialType()) {
      conversionsOrFixes.push_back(
        ConversionRestrictionKind::ExistentialMetatypeToMetatype);
    }
  }
}
```

```cpp
// Metatype to object conversion.
//
// Class and protocol metatypes are interoperable with certain Objective-C
// runtime classes, but only when ObjC interop is enabled.
```


## サブタイプにならないメタタイプ

`A`は`Optional<A>`のサブタイプであることは前回紹介したが、これについては特に規則が用意されていないためメタタイプ同士にはサブタイプ関係はない。

```swift
A.self is Optional<A>.Type // false
```


## まとめ
一応サブタイプ関係があるものもあるけど、必ずしも共変ではない(A < Bであっても A.Type < B.Typeとは限らない)ことに注意。ややこしいですね。。
