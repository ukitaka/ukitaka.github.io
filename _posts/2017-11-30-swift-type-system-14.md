---
layout: post
title:  Swiftの型システムを読む その14 例外を投げる関数と投げない関数のSubtype関係
---

前回の続きというか、前回の記事で網羅できていなかったものを1件見つけたのでメモ。

Swiftでは`throw`を持たない関数は持つ関数として振る舞えるので、なんとなくsubtype関係がありそうなことがわかる。

```cpp
let f1: (Int) -> (Int) = { n in n }
let f2: (Int) throw -> (Int) = f1 // OK
```

しかし`ConversionRestrictionKind` にはFunction間のSubtyingルールが見当たらない。再掲。

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

今回はこれについて調べたメモ。

## TypeChecker::isSubtypeOfでsubtype関係を確認

例によってunittestを書いて確認したところ、やはりSubtype関係がありそう。

```cpp
TEST(Sema, ThrowFunc) {
    auto InputType = C.makeNominal<StructDecl>("A")->getDeclaredType();
    auto ResultType = C.makeNominal<StructDecl>("B")->getDeclaredType();
    const AnyFunctionType::ExtInfo InfoForThrowFunc = AnyFunctionType::ExtInfo(AnyFunctionType::Representation::Swift, true);
    const AnyFunctionType::ExtInfo InfoForNonThrowFunc = AnyFunctionType::ExtInfo(AnyFunctionType::Representation::Swift, false);

    auto NonThrowFuncType = FunctionType::get(InputType, ResultType, InfoForNonThrowFunc);
    auto ThrowFuncType = FunctionType::get(InputType, ResultType, InfoForThrowFunc);
    EXPECT_FALSE(TC->isSubtypeOf(ThrowFuncType, NonThrowFuncType, &DC));
    EXPECT_TRUE(TC->isSubtypeOf(NonThrowFuncType, ThrowFuncType, &DC));
}
```

## Constraint restrictionの確認

前回の記事で、型強制を挿入する目印として`restriction` がある、のように説明したが、今回は`-debug-constraints`で確認をしてみると`costraint restriction`が空であることが確認できる。 

```
---Constraint solving for the expression at [func.swift:2:31 - line:2:31]---
  (overload set choice binding $T0 := (Int) -> Int)
(increasing score due to function conversion)
---Initial constraints for the given expression---
(declref_expr type='(Int) -> Int' location=func.swift:2:31 range=[func.swift:2:31 - line:2:31] decl=func.(file).f1@func.swift:1:5 direct_to_storage function_ref=unapplied)
Score: 0 0 0 0 1 0 0 0 0 0 0 0 0
Contextual Type: (Int) throws -> Int at [func.swift:2:9 - line:2:25]
Type Variables:
  #0 = $T0 [lvalue allowed] [inout allowed] as (Int) -> Int

Active Constraints:

Inactive Constraints:
Resolved overloads:
  selected overload set choice f1: $T0 == (Int) -> Int

(found solution 0 0 0 0 1 0 0 0 0 0 0 0 0)
---Solution---
Fixed score: 0 0 0 0 1 0 0 0 0 0 0 0 0
Type variables:
  $T0 as (Int) -> Int

Overload choices:
  locator@0x7fba1f8c3200 [DeclRef@func.swift:2:31] with func.(file).f1@func.swift:1:5 as f1: (Int) -> Int


Constraint restrictions: // ここ

Disjunction choices:
---Type-checked expression---
(declref_expr type='(Int) -> Int' location=func.swift:2:31 range=[func.swift:2:31 - line:2:31] decl=func.(file).f1@func.swift:1:5 direct_to_storage function_ref=unapplied)
```

## 型チェック済みのASTの確認

`-dump-ast`で型チェック済みのASTを確認してみると`function_conversion_expr`が挿入されていることがわかる。

```
(source_file
  (top_level_code_decl
    (brace_stmt
      (pattern_binding_decl
        (pattern_typed type='(Int) -> Int'
          (pattern_named type='(Int) -> Int' 'f1')
          (type_function
            (type_tuple
              (type_ident
                (component id='Int' bind=Swift.(file).Int)))
            (type_ident
              (component id='Int' bind=Swift.(file).Int))))
        (closure_expr type='(Int) -> Int' location=func.swift:1:24 range=[func.swift:1:24 - line:1:33] discriminator=0 single-expression
          (parameter_list
            (parameter "n" type='(Int)' interface type='(Int)'))
          (declref_expr type='Int' location=func.swift:1:31 range=[func.swift:1:31 - line:1:31] decl=func.(file).top-level code.explicit closure discriminator=0.n@func.swift:1:26 function_ref=unapplied)))
))
  (var_decl "f1" type='(Int) -> Int' interface type='(Int) -> Int' access=internal let storage_kind=stored)
  (top_level_code_decl
    (brace_stmt
      (pattern_binding_decl
        (pattern_typed type='(Int) throws -> Int'
          (pattern_named type='(Int) throws -> Int' 'f2')
          (type_function
            (type_tuple
              (type_ident
                (component id='Int' bind=Swift.(file).Int))) throws
            (type_ident
              (component id='Int' bind=Swift.(file).Int))))
        (function_conversion_expr implicit type='(Int) throws -> Int' location=func.swift:2:31 range=[func.swift:2:31 - line:2:31]
          (declref_expr type='(Int) -> Int' location=func.swift:2:31 range=[func.swift:2:31 - line:2:31] decl=func.(file).f1@func.swift:1:5 direct_to_storage function_ref=unapplied)))
))
  (var_decl "f2" type='(Int) throws -> Int' interface type='(Int) throws -> Int' access=internal let storage_kind=stored))
```

## 型強制の適用

`coerceToType`を読んで見る。
前半は`ConversionRestrictionKind`にもとづいて行われる型強制についての処理が書かれている。
その後にタプルや関数(つまり**構造的部分型**)に関する型強制が記述されている。`FunctionType` -> `FunctionType` の場合は無条件で`FunctionConversionExpr`が挿入されることがわかる。

```cpp
  // Coercions to function type.
  if (auto toFunc = toType->getAs<FunctionType>()) {
    // (略) ...
    // Coercion from one function type to another, this produces a
    // FunctionConversionExpr in its full generality.
    if (auto fromFunc = fromType->getAs<FunctionType>()) {
      // (略) ...
      return cs.cacheType(new (tc.Context)
                              FunctionConversionExpr(expr, toType));
    }
  }
```

つまり構造的部分型の場合はrestrictionが使われない、ということらしい。

- 参考 - [Swiftに息づくstructural types(構造的型) - Qiita](https://qiita.com/takasek/items/c15ef7ce5a00e65a4ad2)

## throwに関する型チェック(エラーがある場合)

逆に`throw`を持った関数は持たない関数としては振る舞えない。

```cpp
let f1: (Int) throw -> (Int) = { n in n }
let f2: (Int) -> (Int) = f1 // NG
```

```
func.swift:2:24: error: invalid conversion from throwing function of type '(Int) throws -> Int' to non-throwing function type '(Int) -> Int'
```

このチェックはおなじみ`matchTypes` からの `matchFunctionTypes`でこで`@autoclosure`や`throw`についてチェックされている。

```cpp
ConstraintSystem::SolutionKind
ConstraintSystem::matchFunctionTypes(FunctionType *func1, FunctionType *func2,
                                     ConstraintKind kind, TypeMatchOptions flags,
                                     ConstraintLocatorBuilder locator) {
  // An @autoclosure function type can be a subtype of a
  // non-@autoclosure function type.
  if (func1->isAutoClosure() != func2->isAutoClosure()) {
    // If the 2nd type is an autoclosure, then the first type needs wrapping in a
    // closure despite already being a function type.
    if (func2->isAutoClosure())
      return SolutionKind::Error;
    if (kind < ConstraintKind::Subtype)
      return SolutionKind::Error;
  }
  
  // A non-throwing function can be a subtype of a throwing function.
  if (func1->throws() != func2->throws()) {
    // Cannot drop 'throws'.
    if (func1->throws() || kind < ConstraintKind::Subtype)
      return SolutionKind::Error;
  }

  // A non-@noescape function type can be a subtype of a @noescape function
  // type.
  if (func1->isNoEscape() != func2->isNoEscape() &&
      (func1->isNoEscape() || kind < ConstraintKind::Subtype))
    return SolutionKind::Error;

  if (matchFunctionRepresentations(func1->getExtInfo().getRepresentation(),
                                   func2->getExtInfo().getRepresentation(),
                                   kind)) {
    return SolutionKind::Error;
  }
  // ... 略
```

## まとめ

構造的部分型はrestrictionが使われず、coerceToTypeで無条件に型強制される。
