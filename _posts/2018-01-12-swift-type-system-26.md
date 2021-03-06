---
layout: post
title:   Swiftの型システムを読む その26 - type(of:)の実装と@_semantics
---

たまたま`type(of:)` の実装を読んでたら少し知見があったのでメモ。

## type(of:)とは？

> You can use the type(of:) function to find the dynamic type of a value, particularly when the dynamic type is different from the static type. The static type of a value is the known, compile-time type of the value. The dynamic type of a value is the value’s actual type at run-time, which can be nested inside its concrete type.

 [https://developer.apple.com/documentation/swift/2885064-type](https://developer.apple.com/documentation/swift/2885064-type)

静的な型ではなく、実行時の`Dynamic type`(のメタタイプ)を返す。
例えば以下の`b`は静的には`Any`型だが、実行時には `Int`型である。

```swift
let a: Int = 1
let b: Any = a
type(of: b) // Int
```

(正確にはExistentialの中身がInt型)

サブクラスなども同様で、`animal`の静的な型は`Animal`だが、実行時の型は`Dog`である。

```swift
class Animal { }
class Dog: Animal { }
let dog = Dog()
let animal: Animal = dog
type(of: animal) // Dog
```

## 実装をみてみる

[stdlib/core/Builtin.swift](https://github.com/apple/swift/blob/a1e3c768869c8f03d6902c27476946bc8bc3d3db/stdlib/public/core/Builtin.swift#L728-L738)に実装がある…あるのだが、みてみると「この実装は使われていない」と書いてある。どういうことだろう？と思って調べてみたのが今回の記事。

```swift
@_inlineable // FIXME(sil-serialize-all)
@_transparent
@_semantics("typechecker.type(of:)")
public func type<T, Metatype>(of value: T) -> Metatype {
  // This implementation is never used, since calls to `Swift.type(of:)` are
  // resolved as a special case by the type checker.
  Builtin.staticReport(_trueAfterDiagnostics(), true._value,
    ("internal consistency error: 'type(of:)' operation failed to resolve"
     as StaticString).utf8Start._rawValue)
  Builtin.unreachable()
}
```


## @_semantics

```swift
@_semantics("typechecker.type(of:)")
```

`@_semantics`は名前の通り特別なセマンティクスが与えられている関数に付けられるアトリビュート。簡単に言えば**コンパイル時に実際の実装が挿入される**。
`typechecker.type(of:)` という名前でgrepしてみると、どうやら`lib/Sema/TypeChecker.cpp`で拾っているようだ。

```cpp
DeclTypeCheckingSemantics
TypeChecker::getDeclTypeCheckingSemantics(ValueDecl *decl) {
  // Check for a @_semantics attribute.
  if (auto semantics = decl->getAttrs().getAttribute<SemanticsAttr>()) {
    if (semantics->Value.equals("typechecker.type(of:)"))
      return DeclTypeCheckingSemantics::TypeOf;
    if (semantics->Value.equals("typechecker.withoutActuallyEscaping(_:do:)"))
      return DeclTypeCheckingSemantics::WithoutActuallyEscaping;
    if (semantics->Value.equals("typechecker._openExistential(_:do:)"))
      return DeclTypeCheckingSemantics::OpenExistential;
  }
  return DeclTypeCheckingSemantics::Normal;
}
```

`ConstraintSystem::resolveOverload`時にこの特別扱いに当てはまるかをチェックして、当てはまれば`DeclTypeCheckingSemantics`の種類に応じた処理を行う。 
`type(of:)`の場合は`DeclTypeCheckingSemantics::TypeOf`になる。

## 型チェックをみる
改めて特別扱いされてはいるものの、基本的には普通の関数と同じように型チェックされる。シグネチャをもう一度見てみると、`T`と`Metatype`の2つの型パラメータをとる。

```swift
public func type<T, Metatype>(of value: T) -> Metatype
```

通常のopenのフローには乗らないものの、いつも通り「型パラメータにフレッシュな型変数を割り当てる」ことによって多相性を実現する。

```cpp
auto input = CS.createTypeVariable(
  CS.getConstraintLocator(locator, ConstraintLocator::FunctionArgument),
  TVO_CanBindToInOut);
auto output = CS.createTypeVariable(
  CS.getConstraintLocator(locator, ConstraintLocator::FunctionResult),
  TVO_CanBindToInOut);
```

そして`Metatype`が`T`の`DynamicType`であるという制約が追加される。

```cpp
CS.addConstraint(ConstraintKind::DynamicTypeOf, output, input,
    CS.getConstraintLocator(locator, ConstraintLocator::RvalueAdjustment));
```

## ASTの書き換えをみてみる

`ExprRewriter::finishApply` の中で`@_semantics`付きの関数の書き換えを行なっている。ここで実装としての ASTが挿入される。

`type(of:)`の場合は`DynamicTypeExpr`が挿入される。

```cpp
auto replacement = new (tc.Context)
  DynamicTypeExpr(apply->getFn()->getLoc(),
                  apply->getArg()->getStartLoc(),
                  arg,
                  apply->getArg()->getEndLoc(),
                  Type());
```

`-dump-ast`などで出力した場合は`metatype_expr`という名前で出てくる。

## SILを見てみる

### Animal -> Dogの例

`value_metatype`はその名の通りvalueのDynamic Typeをとる命令。

```
%15 = value_metatype $@thick Animal.Type, %14
```


### Any -> Intの例

```
%12 = existential_metatype $@thick Any.Type, %8
```

Existentialの場合は`existential_metatype`によってDynamic Typeが取得される。Existentialは(実行時のメモリ上の表現では)、値・メタタイプ・protocol witness tableの3つ組なのでそこからメタタイプを取り出す。


## Swift3とSwift4の違い

Swift3では`type(of:)`は**Parse時に**特別扱いされていたが、Swift4の
[このコミット](https://github.com/apple/swift/commit/1889fde2284916e2c368c9c7cc87906adae9155b)でTypeCheck時の特別扱いに変わった。

> Resolve `type(of:)` by overload resolution rather than parse hackery.
> `type(of:)` has behavior whose type isn't directly representable in Swift's type system, since it produces both concrete and existential metatypes. In Swift 3 we put in a parser hack to turn `type(of: <expr>)` into a DynamicTypeExpr, but this effectively made `type(of:)` a reserved name. It's a bit more principled to put `Swift.type(of:)` on the same level as other declarations, even with its special-case type system behavior, and we can do this by special-casing the type system we produce during overload resolution if `Swift.type(of:)` shows up in an overload set. This also lays groundwork for handling other declarations we want to ostensibly behave like normal declarations but with otherwise inexpressible types, viz. `withoutActuallyEscaping` from SE-0110.


## まとめ

+ `@_semantics`の使われ方がわかった。
+ `ConstraintKind::DynamicTypeOf`の使われ方がわかった。

IUOと`type(of:)`を組み合わせた時の挙動で面白そうなのをDiscordで見かけたので、時間があるときに調べてみる。
