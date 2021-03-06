---
layout: post
title:  Swiftの型システムを読む その13 - SwiftにおけるSubtype関係と型強制意味論
---

やっとSwiftのSubtypingについて少し理解が進んだのでメモ。
最初、OptionalやExistentialへの変換は「なんかいい感じの変換が行われているんだろうなー」くらいの認識で、Subtypingとは関係ないものだと思っていた。

```swift
protocol Animal { }
struct Dog: Animal { }

let dog: Dog = Dog()

// OK: to Optional
let dogOpt: Dog? = dog

// OK: to Existential
let animal: Animal = dog
```

しかし`TypeChecker::isSubtypeOf` という関数を使って調べてみたところ、実際はどちらもSwiftのsubtype関係(<)を満たすことがわかった。。

```cpp
TEST(Sema, Optional) {
    // 略
    auto *aStruct = C.makeNominal<StructDecl>("A");
    Type aTy = aStruct->swift::TypeDecl::getDeclaredInterfaceType();
    Type optATy = OptionalType::get(aTy);

    EXPECT_TRUE(TC->isSubtypeOf(aTy, optATy, &DC));
}

TEST(Sema, Protocol) {
    // 略
   auto *animalProtocol = C.makeProtocol(&DC, "Animal");

    ProtocolType *animalTy = ProtocolType::get(animalProtocol, Type(), C.Ctx);
   Type ty = animalProtocol->swift::NominalTypeDecl::getDeclaredType();
    auto *dogStruct = C.makeNominal<StructDecl>("Dog");
    Type dogTy = dogStruct->getDeclaredInterfaceType();

    auto conformance = C.Ctx.getConformance(dogTy, animalProtocol, SourceLoc(), dogStruct, ProtocolConformanceState::Complete);

    dogStruct->registerProtocolConformance(conformance);

    EXPECT_TRUE(TC->conformsToProtocol(dogTy, animalProtocol, &DC, ConformanceCheckFlags::InExpression));
    EXPECT_TRUE(TC->isSubtypeOf(dogTy, ty, &DC));
}
```

さらには`CSSimplify.cpp`に決定的な記述を見つけた。 [参考](https://github.com/apple/swift/blob/master/lib/Sema/CSSimplify.cpp#L4382-L4383)

```cpp
// for $< in { <, <c, <oc }:
//   T $< U ===> T $< U?
```

もう「サブタイピング」「暗黙変換」などがごっちゃになってわからなくなってきて改めてSubtypingについて調べているうちに、

+ Subtypeの実装には`coercive`と`inclusive`な2種類あってSwiftは`coercive`(強制的)な実装を採用しているようだ
+ TaPLによるとSwiftのsubtypingは「型強制意味論」を採用しているようだ

ということがわかってきて、それについて調べたらいろいろ辻褄があってスッキリした。


## coerciveなsubtypingの実装について

[Subtypingのwikipedia](https://en.m.wikipedia.org/wiki/Subtyping) や[このPDF](http://cgi.cse.unsw.edu.au/~cs3161/07s2/lectures/Week11/Subtyping_paper.pdf)から引用すると

+ Coersiveなsubtypingにおいて、部分型付けはsubtypeからsupertypeへの暗黙の型変換によって定義される。これを型強制(Type coercion)と呼ぶ。
+ すべてのsubtype関係`S <: T`について型強制`S -> T`が提供される
+ 意味解析時に型強制が自動で挿入される。


## 型推論(制約生成~simplify~solve)までの動き

主に`ConstraintKind::Subtype`もしくは`ConstraintKind::Conversion`で`ConstraintSystem::addConstraint`されたとき、

+ subtype関係に基づいて型再構築、型検査が行われる。
+ subtype関係によって制約が満たされる場合は`ConstraintSystem`の`ConstraintRestrictions` に記録される。

```cpp
// どの型とどの型がどのSubtype関係のルールを使って制約を満たすか、のような意味。
SmallVector<std::tuple<Type, Type, ConversionRestrictionKind>, 32>
      ConstraintRestrictions;
```

`ConversionRestrictionKind`がどんなsubtype関係のルール(つまりどんな型強制を使って)制約を満たしたかを表している。

```cpp
/// Specifies a restriction on the kind of conversion that should be
/// performed between the types in a constraint.
///
/// It's common for there to be multiple potential conversions that can
/// apply between two types, e.g., given class types A and B, there might be
/// a superclass conversion from A to B or there might be a user-defined
/// conversion from A to B. The solver may need to explore both paths.
enum class ConversionRestrictionKind {
  /// Tuple-to-tuple conversion.
  TupleToTuple,
  /// Scalar-to-tuple conversion.
  ScalarToTuple,
  /// Deep equality comparison.
  DeepEquality,
  /// Subclass-to-superclass conversion.
  Superclass,
  /// Class metatype to AnyObject conversion.
  ClassMetatypeToAnyObject,
  /// Existential metatype to AnyObject conversion.
  ExistentialMetatypeToAnyObject,
  /// Protocol value metatype to Protocol class conversion.
  ProtocolMetatypeToProtocolClass,
  /// Inout-to-pointer conversion.
  InoutToPointer,
  /// Array-to-pointer conversion.
  ArrayToPointer,
  /// String-to-pointer conversion.
  StringToPointer,
  /// Pointer-to-pointer conversion.
  PointerToPointer,
  /// Lvalue-to-rvalue conversion.
  LValueToRValue,
  /// Value to existential value conversion, or existential erasure.
  Existential,
  /// Metatype to existential metatype conversion.
  MetatypeToExistentialMetatype,
  /// Existential metatype to metatype conversion.
  ExistentialMetatypeToMetatype,
  /// T -> U? value to optional conversion (or to implicitly unwrapped optional).
  ValueToOptional,
  /// T? -> U? optional to optional conversion (or unchecked to unchecked).
  OptionalToOptional,
  /// Implicit forces of implicitly unwrapped optionals to their presumed values
  ForceUnchecked,
  /// Implicit upcast conversion of array types.
  ArrayUpcast,
  /// Implicit upcast conversion of dictionary types, which includes
  /// bridging.
  DictionaryUpcast,
  /// Implicit upcast conversion of set types, which includes bridging.
  SetUpcast,
  /// T:Hashable -> AnyHashable conversion.
  HashableToAnyHashable,
  /// Implicit conversion from a CF type to its toll-free-bridged Objective-C
  /// class type.
  CFTollFreeBridgeToObjC,
  /// Implicit conversion from an Objective-C class type to its
  /// toll-free-bridged CF type.
  ObjCTollFreeBridgeToCF
};
```

上記がすべて。(他にも`Fix`というものがあるけど...)
SuperClassの場合も例外なくこの仕組みに乗っていることがわかる。
逆に`Optional<Dog>` から`Optional<Animal>` は `OptionalToOptional`として定義されていて、ジェネリックな型一般に使えないこともここから見て取れる。

## 型推論(Solutionのapply~AST書き換え)の動き

制約ソルバーによって`Solution` が見つかると、それを元のASTに適用していく。そのとき上で記録した `ConstraintRestrictions` を元に型強制を適用していく。

`lib/Sema/CSApply.cpp`の`ExprRewriter` というクラスによって書き換えが行われ、
サブタイピングが働ける場所(たとえば`let a: Animal = Dog()` のようにばBindingDeclなど)であれば`ExprRewriter::coerceToType`という関数が呼び出され、そこから各`coerceXXXX`へと渡される。

例えば`Existential`というルールに基づく型強制、つまり存在型へのpackについて見てみると、`ErasureExpr`への変換(いわゆる型消し、存在汎化)がされているのがわかる。

```cpp
Expr *ExprRewriter::coerceExistential(Expr *expr, Type toType,
                                      ConstraintLocatorBuilder locator) {
  // ... 略
  // `ErasureExpr`への変換が行われている。
  return cs.cacheType(new (ctx) ErasureExpr(expr, toType, conformances));
}

```

## まとめ

長くなってしまったけれど、ざっくりまとめると以下。

1. Swiftのサブタイピングはcoerciveなサブタイピングで意味論にも影響する
2. どのサブタイピングルールが使われたかを記録して、AST書き換え時に使う


特に`A`と`Optional<A>`のsubtype関係やprotocolにconformしている型と存在型とのsubtype関係などは(サンプル数少ないけど)ほかの言語では見たことがないので、Swiftの型システムの特徴的な部分と言えそう。

そしてやっと全体の作り、理論との繋がりが見えたので詳細に読んでいくことができそう。

## 参考文献

- TaPL 15章 部分型付けに対する型強制意味論
- [Z Luo, S Soloviev, T Xue - Information and Computation, 2013 - Elsevier](http://www.sciencedirect.com/science/article/pii/S0890540112001757)
