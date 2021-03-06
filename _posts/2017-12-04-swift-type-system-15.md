---
layout: post
title:  Swiftの型システムを読む その15 - ArchetypeとGenericTypeParamTypeとDependentMemberType
---

サブタイピングとその意味論については前回までの記事で大まかな作りが把握できて、だいたい実装が読めるようになった….はず…。
以前 [多相な関数](https://blog.waft.me/2017/10/22/swift-type-system-07/)についてはざっくり「let多相のようなもの」だということは確認しているが、実装はほとんど読んでいなかったので次はジェネリクスについてのコードリーディングを進めていくことにした。

今回はGenericに関係する`TypeBase`のサブクラスについて、どこでどのように使われるかを調べた範囲でまとめる。まだ始めたばかりなので間違っていることもたくさんありそうなので注意。

Slavaの[The secret life of types in swift](https://medium.com/@slavapestov/the-secret-life-of-types-in-swift-ff83c3c000a5)という記事がとても参考になって、もう10回以上読み直しているのではないかくらい読んでいるのだが、4部構成のうちの2部で更新が止まってしまっていてしかもジェネリクスまわりの肝心な部分が4部に入っているので「早く更新してくれ〜〜〜」と思っているけど1年以上されてないことをみると望みは薄そう。

## 型パラメータに関する前提確認

Swiftにおいて、型パラメータを含みうるのは以下の2箇所。

1. struct/class/enum
```swift
struct Hoge<A> { }
class Hoge<A> { }
enum Hoge<A> { }
```
2. func
```swift
func hoge<A>(a: A) -> A { }
```


逆に、`VarDecl`(`let`など)には型パラメータを持たせることはできない。

```swift
// NG: できない
let id: <A> (A) -> A = { a in a }
```

また、`associated type`もある種型パラメータのような役割を果たす。

```swift
protocol MyProtocol {
  associatedtype T 
}
```


## Genericsに関係するType

読み始めて最初に思ったのが「ジェネリックなパラメータを指しているものがいくつかあって、それぞれ何をを指しているのかよくわからん」ということだった。

![GenTypes.svg (10.2 kB)](https://img.esa.io/uploads/production/attachments/2245/2017/12/04/2884/f3788e6b-9ef5-4c18-8625-28ccbf0b530b.svg)


それぞれについてわかった範囲でメモしておく。

## GenericTypeParamType

> Describes the type of a generic parameter.

名前の通り「型パラメータ」を表すType。struct/class/enum/funcなど型パラメータを含むDeclにおいて使われているよう。


例えば`struct Hoge<A>` の`A`は`GenericTypeParam` になる。

```
(lldb) po SD->getInterfaceType()->dump()
(metatype_type
  (bound_generic_struct_type decl=struct.(file).Hoge@/Users/ukitaka/Development/ios/tmp/struct.swift:1:8
    (generic_type_param_type depth=0 index=0 decl=struct.(file).Hoge.A@/Users/ukitaka/Development/ios/tmp/struct.swift:1:13)))
```


また`protocol P { associatedtype T }` の`T`も`GenericTypeParam`になる。

```
(lldb) po assocType->getInterfaceType()->dump()
(metatype_type
  (dependent_member_type assoc_type=proto.(file).Animal.T@/Users/ukitaka/Development/ios/tmp/proto.swift:2:18
    (base=generic_type_param_type depth=0 index=0 decl=proto.(file).Animal.Self)))
```

もう少しいうと静的に具体的な型に決まる型は`GenericTypeParamType`として扱われる、と言えそう？

## ArchetypeType

> An archetype is a type that represents a runtime type that is known to conform to some set of requirements. Archetypes are used to represent generic type parameters and their associated types, as well as the runtime type stored within an existential container.
> 

[https://github.com/apple/swift/blob/master/include/swift/AST/Types.h#L4220-L4226](https://github.com/apple/swift/blob/master/include/swift/AST/Types.h#L4220-L4226)

>   archetype
>    A placeholder for a generic parameter or an associated type within a
>    generic context. Sometimes known as a "rigid type variable" in formal
>    CS literature. Directly stores its conforming protocols and nested
>    archetypes, if any.

 
[https://github.com/apple/swift/blob/master/docs/Lexicon.rst](https://github.com/apple/swift/blob/master/docs/Lexicon.rst)

ざっくりいうと実行時に決まる型たち。struct/class/enum/func等、型パラメータを含みうる箇所**以外**で出てきた場合この型になる。
例えば`struct Hoge<A>` のが`A`型のプロパティを持っているとき、この`A`は`ArtchetypeType`になる。

```swift
struct Hoge<A> {
    let a: A // このA
}
```

```
(lldb) po PBD->getPattern(0)->getType()->dump()
(archetype_type name=A address=0x10e823a28
  0x10b884050 Module name=genst
    0x10e823010 FileUnit file="/Users/ukitaka/Development/ios/tmp/genst.swift"
      0x10e823450 StructDecl name=Hoge
)
```

また、実行時に決まる型として`Self`や`associated type`の型も挙げられ、これも`ArtchetypeType`になる。

```swift
protocol Hoge {
  associatedtype T // こっちはGTPT
  var s: Self { get } // Archetype
  var t: T { get } // こっちはArchetype
}
```

```
(lldb) po PBD->getPattern(0)->getType()->dump()
(archetype_type name=Self address=0x10c072bf0 conforms_to=proto_self.(file).Hoge@/Users/ukitaka/Development/ios/tmp/proto_self.swift:1:10
  0x10b03b450 Module name=proto_self
    0x10c072010 FileUnit file="/Users/ukitaka/Development/ios/tmp/proto_self.swift"
      0x10c072360 ProtocolDecl name=Hoge
)
```

```
(lldb) po PBD->getPattern(0)->getType()->dump()
(archetype_type name=Self.T address=0x10b04f598 parent=0x10b04f428 assoc_type=proto_self.(file).Hoge.T@/Users/ukitaka/Development/ios/tmp/proto_self.swift:2:18
  0x10a079450 Module name=proto_self
    0x10a086010 FileUnit file="/Users/ukitaka/Development/ios/tmp/proto_self.swift"
      0x10a086360 ProtocolDecl name=Hoge
)
```

例えばまだないけど、OpeningExistentialが入ったとして、`openas` で指定された型なども`ArchetypeType`になる。

```swift
let openedA = a openas A // AはArchetypeになる。
```


## DependentMemberType

> DependentMemberType -- the type of an associated type of a generic parameter conforming to a protocol

GenericTypeParamTypeと同様のポジションに現れる型パラメータのうち、`T.Element` のように型パラメータのメンバー型を参照しているときは`DependentMemberType`という型になる。

ArchetypeのポジションではそのままArchetype。

```swift
struct Hoge<T: Sequence> { // このTはGTPT
  let e: T.Element // ここはAT
   
  // ここのT.ElementはDMT
  func fuga<A>(e: T.Element) -> T.Element {
    return e
  }
}
```

````
(lldb) po FD->getInterfaceType()->dump()
(generic_function_type escaping
  (input=paren_type
    (bound_generic_struct_type decl=depen.(file).Hoge@/Users/ukitaka/Development/ios/tmp/depen.swift:1:8
      (generic_type_param_type depth=0 index=0 decl=depen.(file).Hoge.T@/Users/ukitaka/Development/ios/tmp/depen.swift:1:13)))
  (output=function_type escaping
    (input=tuple_type num_elements=0)
    (output=dependent_member_type assoc_type=Swift.(file).Sequence.Element
      (base=generic_type_param_type depth=0 index=0 decl=depen.(file).Hoge.T@/Users/ukitaka/Development/ios/tmp/depen.swift:1:13)))
  ( generic_sig=<T where T : Sequence>))
```

```
(archetype_type name=T.Element address=0x10b836fa0 parent=0x10a85daf8 assoc_type=Swift.(file).Sequence.Element
  0x10c07a450 Module name=depen
    0x10a85cc10 FileUnit file="/Users/ukitaka/Development/ios/tmp/depen.swift"
      0x10a85d080 StructDecl name=Hoge
)
```

## UnboundGenericType
> UnboundGenericType - Represents a generic type where the type arguments have not yet been resolved.

型パラメータが残った状態のジェネリックな型。
型パラメータを持つstructの`StructDecl`に対して、`getDeclaredType()`で型を取ると取れる。

```
(lldb) po SD->getDeclaredType()->dump()
(unbound_generic_type decl=genst.(file).Hoge@/Users/ukitaka/Development/ios/tmp/genst.swift:1:8)
```


ちなみに`getInterfaceType()`で取ると、先程まで見ていたとおりGenericTypeParamTypeで埋められたBoundなGeneric型として取れる。

```
(lldb) po SD->getInterfaceType()->dump()
(metatype_type
  (bound_generic_struct_type decl=genst.(file).Hoge@/Users/ukitaka/Development/ios/tmp/genst.swift:1:8
    (generic_type_param_type depth=0 index=0 decl=genst.(file).Hoge.A@/Users/ukitaka/Development/ios/tmp/genst.swift:1:13)))
```

このあたりで違いがあるのだろうけど、`getInterfaceType`と`getDeclaredType`の違いは説明できない。。また別の記事で書く。


## 参考文献

- [The secret life of types in swift](https://medium.com/@slavapestov/the-secret-life-of-types-in-swift-ff83c3c000a5)
	- 必読。というか、最初に読めばよかったのでこのシリーズを人に見せる用に書き直すときにはかならず紹介したい。

+ [Lexicon](https://github.com/apple/swift/blob/master/docs/Lexicon.rst)
	+ 用語集。網羅していないし解説もざっくりだけど、公式のドキュメントなので信頼はできる。


## まとめ

ジェネリクスの実装コードリーディング第一弾としてまずはType関連のくらすを紹介した。
まだまだ説明すべき概念がたくさんあるので、次回以降で説明する(予定)
