---
layout: post
title:  Swiftの型システムを読む その11 - Typeクラス周辺のクラス図
---

(もはや記事のナンバリングの意味はあんまりなくなってきた気がする。。)

`lib/Sema`や依存する`lib/AST` を読むにあたっての障壁の1つとして`Type`関連のクラス多すぎ問題がある。

ざっと書いただけでも

+ `TypeRepr`
+ `Type`
+ `TypeLoc`
+ `TypeBase` やその派生クラスたち
+ `CanType` で表現されるCanonicalなType
+ 型には現れないけど `InterfaceType` とか `DeclaredType` とかなにを指しているのかイマイチよくわからないものたち
    + `Type* getXXXXType`だけでも数十種類ある

ここをクリアすべくType周辺のクラス図を作ってみた。
各クラスの解説は少しだけにして、詳細な解説はいつかやる。

## TypeBase / Type / CanType

基本的には型を表すクラスは`Type`ではなく `TypeBase` を継承して作られている。

じゃあ`Type`クラスはなに？というと`TypeBase` のポインタを持ったシンプルな値型。TypeBaseからの暗黙変換？(C++的になんと呼ぶかわからない) が定義されているので、コードを追っていると突然`TypeBase`から`Type`になっていることがある。

`CanType` は「Canonical Type」の略で、シンタックスシュガー(`A?` -> `Optional<A>`) やtypealiasなどを解決し、その結果をTypeのキャッシュをしたりする。

Canonicalizeされた結果が同じなら同じポインタを返すので、そのままハッシュのキーとして使えるとか。
`Sema` 内でもよく `type->getCanonicalType()` を使ってからなにかチェックを行っていることが多いのでかなり重要なものと思われる。

![CanType.svg (4.9 kB)](https://img.esa.io/uploads/production/attachments/2245/2017/11/23/2884/35d52c92-1cb3-4096-9e11-5ec6a2b90301.svg)

## NominalType

一番馴染みがありそうな型たち。以前の記事でprotocolは型じゃないのでは？みたいに書いたけど`ProtocolType` がありますね。。。ただ`ProtocolType`だと `isExistentialType` でtrueが返るので存在型を表しているのでしょう
かね。。

![NominalType.svg (10.0 kB)](https://img.esa.io/uploads/production/attachments/2245/2017/11/23/2884/b5926812-7222-47c8-a7d5-dfcfdaa82612.svg)

## BoundGenericType

型パラメータを持っていて、その型パラメータがすでに具体的になっている状態を「Bound」を呼ぶらしい。

![BoundGenericType.svg (8.6 kB)](https://img.esa.io/uploads/production/attachments/2245/2017/11/23/2884/13a980c1-b1fe-4916-9c5e-067ff468e57f.svg)

## FunctionType

関数を表す型で、型パラメータを持つかどうかで別れている。

![AnyFunction.svg (6.8 kB)](https://img.esa.io/uploads/production/attachments/2245/2017/11/23/2884/dab1e021-10b4-4cc5-a8ff-d7641acd9e55.svg)

## SyntaxSugarType

名前の通り `[Int]`, `Int?` , `Int!` らへんを指すみたい。上に書いた`getCanonicalType` のタイミングでデシュガーされる。

![SyntaxSugar.svg (8.5 kB)](https://img.esa.io/uploads/production/attachments/2245/2017/11/23/2884/63621ab6-f41b-431f-a0d4-8ad69dff6288.svg)


## MetatypeType

`String.Type` とか `MyProtocol.Protocol` とかを指す型のよう。
`Existential`かどうかで別れているんですね。

![MetatypeType.svg (6.9 kB)](https://img.esa.io/uploads/production/attachments/2245/2017/11/23/2884/2d2107ca-a5d5-416c-897e-4286fe7cd0f5.svg)


## SubstitutableType

これもきちんと調べてないけど、`Substitutable` という名前から推測すると、おそらく型推論後にApplyされる型なのだろうか...？
でも一番それっぽい`TypeVariableType` は`SubstitutableType` ではないのでわからない。。

![SubstitutableType.svg (6.9 kB)](https://img.esa.io/uploads/production/attachments/2245/2017/11/23/2884/16fa623a-851d-4303-9c96-1c71d61d4a17.svg)

## BuiltinType

![BuiltinType.svg (19.2 kB)](https://img.esa.io/uploads/production/attachments/2245/2017/11/23/2884/93bfbd6a-eb89-48e8-96ea-367196e599e5.svg)

## その他 TypeBase派生クラス

![OtherTypes.svg (18.9 kB)](https://img.esa.io/uploads/production/attachments/2245/2017/11/23/2884/e9ad600f-9fec-441d-a241-5c6dd1d12b0e.svg)


## まとめ

とりあえず`TypeBase` とその派生クラスたちをざっと見た。
少し気になったのは`throws` とかは型にあらわれていないけどどこで管理しているのだろう？と思った。
 (`inout` は`InOutType` として現れているのに)

`TypeLoc`や`TypeRepr`周りもそのうち書くかも。


