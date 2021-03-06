---
layout: post
title:  Swiftの型システムを読む その7 - 多相な関数について
---

TaPLの23章「全称型」を最初に読んだ時、「あ〜はいはい、ジェネリクスでしょ！知ってるよ！」みたいな雑な理解をしていたが、よくよく読むとSwiftでのジェネリクスは23章のSystemFで扱っているような(冠頭に限らない)全称量化は許されていなくて、22章のlet多相をさらに制限したようなものであることがやっとわかってきた。

let多相について、Swiftにも`let` というキーワードがあるがそれには多相性はなく(後述)ややこしいので、この記事では TaPLの表記に合わせて「ML式多相」と呼ぶことにする。

あと型パラメータをもつstruct / class / enumは次の記事で別で考えるので、ここでは関数のみを扱っていることに注意。

また、この記事ではサブタイピングや有界量化などは考えない。

## Swiftでの関数の多相性について
「ML式多相をさらに制限したようなもの」と表現した通り、できることは少ない。
もともとMLにおけるlet多相には、健全性のために「右辺は値でなければいけない」という制限(値制限と呼ばれる)があるが、Swiftではさらに制限されて「funcキーワードで定義された関数」のみが多相性を持てる。

> 多相性という言葉は、プログラムの一部分を、異なる文脈では異なる型として用いることを可能にする各種言語機能を指す。
> 
> (22.7 let多相より)

TaPLでのlet束縛、逐次実行の表記に合わせて書くと、`id` が多相性を持っていることがわかる。

```
let id = λa. a in 
	id(true); 
	id(123)
```

一方で[docs/TypeChecker.rst](https://github.com/apple/swift/blob/master/docs/TypeChecker.rst)を読むと、Swiftの型推論には

> Swift limits the scope of type inference to a single expression or statement

とあるように1つの式・文でしか型推論ができないという制約があるため、Swiftの`let` で同じようなことをしようとしても多相にはならない。

```swift
let a = 42 // この時点でaはIntに決まっている

printInt32(a) // NG
printInt(a) // OK
printInt64(a) // NG
```

また、そもそも型変数がsingle expression / statement で解決できない場合はコンパイル時にエラーになるので上記のような(型変数を含んだ)`id`   を作ることすらもできない。

```swift
let id = { a in a } // そもそもエラー
```

一方で `func` によって定義された関数であればML式多相のように、「型スキームのインスタンス化」のような方式で多相性を実現できる。

```swift
func id<A>(_ a: A) -> A { return a }

id(42)
id(true)
id("hello")
```

実際 [TypeChecker.rst - Polymorphic Types](https://github.com/apple/swift/blob/master/docs/TypeChecker.rst#polymorphic-types)を読むとまさにそのような説明が書いてある。

> When the constraint generator encounters a reference to a generic function, it immediately replaces each of the generic parameters within the function type with a fresh type variable, introduces constraints on that type variable to match the constraints listed in the generic function, and produces a monomorphic function type based on the newly-generated type variables. 


## 型推論について

上に書いた通り「 ジェネリックな関数が出現したら、その型パラメータにフレッシュな型変数を割り当て、制約を生成して、単相な関数を作る」ということらしいので、以下のコードを元にswiftの型推論をのぞいてみる。

```swift
func id<A>(_ a: A) -> A { return a }

let t = (id(true), id("hello"))
```

```
$ swift -frontend -typecheck -debug-constraints test_poly.swift
```

(長いので出力は省略)

```
(id(true: $T2): $T1, id("hello": $T5): $T4) : $T6
```

2つのid自体の型は一旦 `$T0`, `$T3` で置かれ、

```
---Constraint solving for the expression at [test_poly.swift:3:9 - line:3:31]---
  (overload set choice binding $T0 := ($T1) -> $T1)
  (overload set choice binding $T3 := ($T4) -> $T4)
``` 

右辺のタプルについてのASTの出力をみると、二つの `declrec_expr` について、別々の(フレッシュな)型変数 `$T1` と `$T4` が割り当てられてることが確認できる。

```
(declref_expr type='($T1) -> $T1'
...
(declref_expr type='($T4) -> $T4'
```

これをジェネリックな関数の型パラメータを型変数に置き換えることを `open` と呼ぶみたい？ `τ_0_0` がおそらく型パラメータ`A` 自体を指している。

```
Opened types:
  locator@0x7fb9990c4a00 [DeclRef@test_poly.swift:3:10] opens τ_0_0 -> $T1
  locator@0x7fb9990c4a18 [DeclRef@test_poly.swift:3:20] opens τ_0_0 -> $T4
```

あとは同じように制約ソルバによって単一化される。


```
($T2 literal=3 involves_type_vars bindings=(subtypes of) (default from ExpressibleByBooleanLiteral) Bool)
($T5 literal=3 involves_type_vars bindings=(subtypes of) (default from ExpressibleByStringLiteral) String)
($T6 involves_type_vars bindings=(supertypes of) ($T1, $T4))
Active bindings: $T6 := ($T1, $T4)
(trying $T6 := ($T1, $T4)
  ($T2 literal=3 involves_type_vars bindings=(subtypes of) (default from ExpressibleByBooleanLiteral) Bool)
  ($T5 literal=3 involves_type_vars bindings=(subtypes of) (default from ExpressibleByStringLiteral) String)
  Active bindings: $T2 := Bool
  (trying $T2 := Bool
    ($T1 bindings=(supertypes of) Bool)
    ($T5 literal=3 involves_type_vars bindings=(subtypes of) (default from ExpressibleByStringLiteral) String)
    Active bindings: $T1 := Bool
    (trying $T1 := Bool
      ($T5 literal=3 involves_type_vars bindings=(subtypes of) (default from ExpressibleByStringLiteral) String)
      Active bindings: $T5 := String
      (trying $T5 := String
        ($T4 bindings=(supertypes of) String)
        Active bindings: $T4 := String
        (trying $T4 := String
          (found solution 0 0 0 0 0 0 0 0 0 0 0 0 0)
        )
      )
    )
  )
)
```

```
Type variables:
  $T0 as (Bool) -> Bool
  $T1 as Bool
  $T5 as String
  $T2 as Bool
  $T6 as (Bool, String)
  $T3 as (String) -> String
  $T4 as String
```

型がつけられたASTは以下のようになる。

```
(tuple_expr type='(Bool, String)' location=test_poly.swift:3:9 range=[test_poly.swift:3:9 - line:3:31]
  (call_expr type='Bool' location=test_poly.swift:3:10 range=[test_poly.swift:3:10 - line:3:17] arg_labels=_:
    (declref_expr type='(Bool) -> Bool' location=test_poly.swift:3:10 range=[test_poly.swift:3:10 - line:3:10] decl=test_poly.(file).id@test_poly.swift:1:6 [with Bool] function_ref=single)
    (paren_expr type='(Bool)' location=test_poly.swift:3:13 range=[test_poly.swift:3:12 - line:3:17]
      (call_expr implicit type='Bool' location=test_poly.swift:3:13 range=[test_poly.swift:3:13 - line:3:13] arg_labels=_builtinBooleanLiteral:
        (constructor_ref_call_expr implicit type='(Int1) -> Bool' location=test_poly.swift:3:13 range=[test_poly.swift:3:13 - line:3:13]
          (declref_expr implicit type='(Bool.Type) -> (Int1) -> Bool' location=test_poly.swift:3:13 range=[test_poly.swift:3:13 - line:3:13] decl=Swift.(file).Bool.init(_builtinBooleanLiteral:) function_ref=single)
          (type_expr implicit type='Bool.Type' location=test_poly.swift:3:13 range=[test_poly.swift:3:13 - line:3:13] typerepr='Bool'))
        (tuple_expr implicit type='(_builtinBooleanLiteral: Builtin.Int1)' location=test_poly.swift:3:13 range=[test_poly.swift:3:13 - line:3:13] names=_builtinBooleanLiteral
          (boolean_literal_expr type='Builtin.Int1' location=test_poly.swift:3:13 range=[test_poly.swift:3:13 - line:3:13] value=true)))))
  (call_expr type='String' location=test_poly.swift:3:20 range=[test_poly.swift:3:20 - line:3:30] arg_labels=_:
    (declref_expr type='(String) -> String' location=test_poly.swift:3:20 range=[test_poly.swift:3:20 - line:3:20] decl=test_poly.(file).id@test_poly.swift:1:6 [with String] function_ref=single)
    (paren_expr type='(String)' location=test_poly.swift:3:23 range=[test_poly.swift:3:22 - line:3:30]
      (string_literal_expr type='String' location=test_poly.swift:3:23 range=[test_poly.swift:3:23 - line:3:23] encoding=utf8 value="hello" builtin_initializer=Swift.(file).String.init(_builtinStringLiteral:utf8CodeUnitCount:isASCII:) initializer=**NULL**))))
```

## まとめ

とりあえず関数についてのみSwiftにおけるType Polymorphismを確認できた。このシリーズいままでほぼコンパイラのコードを読んでただけなので、ようやく理論 → 実装のつながりを見つけられてちょっと感動した:D
