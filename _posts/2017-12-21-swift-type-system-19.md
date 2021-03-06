---
layout: post
title:  Swiftの型システムを読む その19 - Member制約とLookup
---

今回から何回か`X[.name] == Y`のような`Member`というタイプの制約の生成、単一化周りを見てみる。まずは用語などを整理をしてみる。

## Member
要はプロパティやメソッドなど。`.`(ドット)でアクセスする。

```swift
struct Dog {
     var name: String
     func bark() { print("わんわん") }
}
```

```swift
dog.name
dog.bark()
```

もうちょっというと`ValueDecl`のサブクラスはだいたいメンバーになれるので、例えばプロパティ、メソッド以外にもネストしたstruct/class/enumの定義や、typealiasなども含まれそう。

また、そもそもMemberを持てる型は大きく2種類。

+ Nominalな型
	+ つまりclass / struct / enum / protocol
+ ArchetypeType(制約付き)


## MemberLookup/NameLookup

与えられた名前からメンバーを探すことを`MemberLookup`という。
`Sema`内では`ConstraintSystem::performMemberLookup`がエントリーポイントで、幾つかの`Sema`関連クラスを通って`AST`の`NameLookup.cpp`の関数を呼び出している。

<img width="426" alt="スクリーンショット 2017-12-21 18.41.24.png (171.7 kB)" src="https://img.esa.io/uploads/production/attachments/2245/2017/12/21/2884/705e153b-9dfe-4edc-a218-bddaadee6989.png">


いつ`performMemberLookup`がされるかというと、主にMemberに関する制約生成~Simplify時だが、大きく分けると2つの関数から呼ばれる。

+ `ConstraintSystem::addValueMemberConstraint`
+ `ConstraintSystem::addUnresolvedValueMemberConstraint`

## UnresolvedなExprたち

いくつかの文脈で`Unresolved`という単語がでてくるので確認しておく。
基本的には意味どおり「まだ何を指しているかわからない状態」。

例えば以下のようなコードのparse直後のASTを見てみると

```swift
enum E { case a }
func f(_ e: E) { }
f(.a)
```

```
(call_expr type='<null>' arg_labels=_:
  (unresolved_decl_ref_expr type='<null>' name=f function_ref=unapplied)
  (paren_expr type='<null>'
    (unresolved_member_expr type='<null>' name='a' arg_labels='))))))
```


+ `unresolved_decl_ref_expr`
	+ 関数など、なにか他のところで定義されたものを指しているが、分からない状態。上で言うと`f`。
+ `unresolved_member_expr`
	+ そのメンバーが属するstruct/class/enumなどが省略された場合はこれになる。上で言うと`.a`

もし`.a`を省略せずに`E.a`と書いた場合にはまた別のASTになる。

```
(call_expr type='<null>' arg_labels=_:
  (unresolved_decl_ref_expr type='<null>' name=f function_ref=unapplied)
  (paren_expr type='<null>'
    (unresolved_dot_expr type='<null>' field 'a' function_ref=unapplied
      (unresolved_decl_ref_expr type='<null>' name=E function_ref=unapplied)))))))
```

+ `unresolved_dot_expr`
	+ そのメンバーが属するstruct/class/enumなどが明示的に書かれている場合には`.a`はこのASTになる。


## Unresolvedはいつ解決される？

+ `unresolved_dot_expr`の場合は制約生成時に`ConstraintGenerator::addValueMemberConstraint`で制約追加される。その際に`performMemberLookup`される。

+ `unresolved_member_expr`の場合は制約生成時に`ConstraintGenerator:: addUnresolvedValueMemberConstraint `で制約追加される。その際に`performMemberLookup`される。

+ `unresolved_decl_ref_expr`の場合は`Expr`の型チェックの`PreCheck`と呼ばれるフェーズで解決される。


## Qualified/Unqualified

`f`とかのように何かのメンバーでないものを指すときは「Unqualified」。
`E.a`のように何かのメンバーであるものを指すときは「Qualified」

AST内でのNameLookup時は区別される。


## まとめ

とりあえず用語をみたので、次はもう少し具体的に型推論周りなど見てみる。
