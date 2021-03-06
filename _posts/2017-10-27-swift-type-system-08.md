---
layout: post
title:  Swiftの型システムを読む その8 - 存在型とprotocol
---

以前  [型システムの理論からみるSwiftの存在型(Existential Type) - Qiita](https://qiita.com/ukitaka/items/a993b5d7ed5ae84b1b52) を投稿したのだけど、このときは「protocol型の変数は存在型だよ」とかさらっと書いただけで、しかも説明のためにnominalなものとそうでないものを混ぜて書いてしまってなんだか微妙な記事になってしまった気がする。。

今回はいくつかギモンだった部分が解決してスッキリしたので、もうちょっとSwiftコンパイラの動きと合わせて書いてみる。

## Swiftの存在型をややこしくしていている3つのルール
 
Swiftのprotocolは存在型であることを意識させないような言語設計になっているので、使いやすくはあるけど理解しようと思うと少しややこしい。

存在型であることを意識させないようにするための仕組みが3つある。

1. `protocol Animal { }` について、`let animal: Animal` のように型のポジションに出て来るprotocolは、 `{ ∃X, X: Animal }` という存在型の略記である。
2. ` let animal: Animal = Dog()` のような場合は`Dog` 型から `{ ∃X, X: Animal }` への暗黙変換が型チェック時に挿入される
3. 存在型へのメソッド呼び出し等の際には暗黙で`Opening existential` が型チェック時に挿入される。


## そもそもProtocolは型なのか？
上のQiitaでも引用したとおり、swiftのドキュメントにはこう書かれている。

>  In Swift, we consider protocols to be types. A value of protocol type has an existential type, meaning that we don't know the concrete type until run-time (and even then it varies), but we know that the type conforms to the given protocol.


「Protocolは型として考えられる」「Protocol型の値は存在型を持つ」ややこしい。。ちょっと整理していく。

例えば `Animal` プロトコルがあったとする。

```swift
protocol Animal { }
```

例えばそれに準拠する `Dog` があったとして、`Animal` 型の変数に代入できるように見える。

```swift
let animal: Animal = Dog()
```

じゃあ「やっぱり`Animal` は型じゃん！」となりそうだけど**実はそうではなく、Animalという名前の型は型システム上は存在しない。**

存在するのは `{∃X, X: Animal }` という型である。

**つまりSwiftの文法上の型の位置に現れるprotocol名は存在型の略記で あり、`Animal` 型は型システム上は存在しない。**


## 存在型への暗黙変換と”サブタイピング”

```swift
let animal: Animal = Dog()
```

`Animal` は存在型 `{ ∃X, X: Animal}`であり、`Dog` は普通の型であり一見違うように見えるが、これは**存在型への暗黙変換**により型チェックが通る。

`-dump-ast` で出力を見てみると、確かに `erasure_expr` , `impliit` の文字が見える。 

```
(erasure_expr implicit type='Animal' location=proto.swift:4:22 range=[proto.swift:4:22 - line:4:26]
          (normal_conformance type=Dog protocol=Animal)
          (call_expr type='Dog' location=proto.swift:4:22 range=[proto.swift:4:22 - line:4:26] nothrow arg_labels=
            (constructor_ref_call_expr type='() -> Dog' location=proto.swift:4:22 range=[proto.swift:4:22 - line:4:22] nothrow
              (declref_expr implicit type='(Dog.Type) -> () -> Dog' location=proto.swift:4:22 range=[proto.swift:4:22 - line:4:22] decl=proto.(file).Dog.init()@proto.swift:2:8 function_ref=single)
              (type_expr type='Dog.Type' location=proto.swift:4:22 range=[proto.swift:4:22 - line:4:22] typerepr='Dog'))
            (tuple_expr type='()' location=proto.swift:4:25 range=[proto.swift:4:25 - line:4:26]))))
```

`Dog: Animal`,  `let animal: Animal` という記法と暗黙的な変換が合わさって一見サブタイピングのような動きをするのがややこしいが、これは述語論理的にはこれは存在汎化にあたる規則なので~~サブタイピングとは関係ないはず~~。サブタイピングでした。 [追記しました。 ](https://blog.waft.me/2017/11/29/swift-type-system-13/)


## 存在型のメソッド呼び出しと"Opening existential"

例えば型パラメータを1つ取る`struct Hoge<T>` があったする。

```swift
struct Hoge<T> { }
```

`Hoge<String>` 型の項に対して、`String` 型のメソッドを呼び出すことは当然できない。

```swift
let hoge: Hoge<String> = ...
hoge.isEmpty // NG
```

`Optional<String>` であれば `?` を使えば呼び出せるがこれは特殊ケース。

```swift
let hoge: Optional<String> = ...
hoge?.isEmpty // OK
```

`{ ∃X, X: Animal }` は中の`X` こそ`Animal`プロトコルに準拠しているものの、この存在型自体は `Animal` プロトコルに準拠していない。のでメソッドが呼び出せるはずがない。**….が、呼び出せる！** 

```swift
let animal: Animal = ...
animal.bark() // OK
```

これがややこしいポイントその3である。
`.bark()` の呼び出し部分のASTを見てみると `open_existential_expr` が挿入されていることがわかる。`implicit` 。

```
(open_existential_expr implicit type='()' location=proto.swift:13:3 range=[proto.swift:13:1 - line:13:8]
        (opaque_value_expr implicit type='Animal' location=proto.swift:13:1 range=[proto.swift:13:1 - line:13:1] @ 0x7fef860f07b0)
        (declref_expr type='Animal' location=proto.swift:13:1 range=[proto.swift:13:1 - line:13:1] decl=proto.(file).a@proto.swift:11:5 direct_to_storage function_ref=unapplied)
        (call_expr type='()' location=proto.swift:13:3 range=[proto.swift:13:1 - line:13:8] nothrow arg_labels=
          (dot_syntax_call_expr type='() -> ()' location=proto.swift:13:3 range=[proto.swift:13:1 - line:13:3] nothrow
            (declref_expr type='(Animal) -> () -> ()' location=proto.swift:13:3 range=[proto.swift:13:3 - line:13:3] decl=proto.(file).Animal.bark()@proto.swift:2:8 [with Animal[abstract:Animal]] function_ref=single)
            (opaque_value_expr implicit type='Animal' location=proto.swift:13:1 range=[proto.swift:13:1 - line:13:1] @ 0x7fef860f07b0))
          (tuple_expr type='()' location=proto.swift:13:7 range=[proto.swift:13:7 - line:13:8]))))))
```

これはOpening existential の記法を使うと、以下のようになる。

```swift
// これは以下のシンタックスシュガー
animal.bark()

// aはA型。 AはAnimalプロトコルに準拠
let a = animal openas A
a.bark()
```

## まとめ
今回は型の位置に現れるプロトコルについてのルールと暗黙変換のルールをいくつかみた。制約の位置(有界量化)のルールやプロトコル同士の関係 `protocol Animal: Creature` などはまだあまり調べてないので今度調べる。


