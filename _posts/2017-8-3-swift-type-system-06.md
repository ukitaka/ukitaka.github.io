---
layout: post
title: Swiftの型システムを読む その6
---


今回は`let a: Int = 42` の場合に、`ConstraintKind::Conversion` の型が `Int` に決まる仕組みを見ていく。

## どういうこと？

`let a = 42` の場合

```
Type Variables:
  #0 = $T0 [inout allowed]
  #1 = $T1 [inout allowed]

Active Constraints:

Inactive Constraints:
  $T0 literal conforms to ExpressibleByIntegerLiteral [[locator@0x7f81d000ae00 [IntegerLiteral@study/s001.swift:1:9]]];
  $T0 conv $T1 [[locator@0x7f81d000ae00 [IntegerLiteral@study/s001.swift:1:9]]];
```

`let a: Int = 42` の場合

```
Type Variables:
  #0 = $T0 [inout allowed]

Active Constraints:

Inactive Constraints:
  $T0 literal conforms to ExpressibleByIntegerLiteral [[locator@0x7fd5f8807200 [IntegerLiteral@study/s0007.swift:1:14]]];
  $T0 conv Int [[locator@0x7fd5f8807200 [IntegerLiteral@study/s0007.swift:1:14]]];
```


`conv` の部分が(おそらく左辺に明示した`Int`によって) `Int`になっていることがみてとれる。

## `ConstraintKind::Conversion` を`addConstraint` しているところを確認する

`TypeChecker::typeCheckBinding` の中の `BindingListener::buildConstraints` 。

```cpp
    bool builtConstraints(ConstraintSystem &cs, Expr *expr) override {
		...
      cs.addConstraint(ConstraintKind::Conversion, cs.getType(expr),
                       InitTypeAndInOut.getPointer(), Locator,
                       /*isFavored*/true);
		...
    }
```


ここでsecondに渡されている `InitTypeAndInOut.getPointer()` が`Int` になっている。

`InitTypeAndInOut` は名前の通り、型とinoutかどうかを表すもの。
これに型をセットしている部分をみると

```cpp
InitTypeAndInOut.setPointer(cs.generateConstraints(pattern, Locator));
```

どうやら `pattern` 、つまり`=` の左辺の `generateConstraitns` の結果の型で決まるらしい。

```cpp
Type ConstraintSystem::generateConstraints(Pattern *pattern,
                                           ConstraintLocatorBuilder locator) {
  ConstraintGenerator cg(*this);
  return cg.getTypeForPattern(pattern, locator);
}
```


`ConstraintGenerator::getTypeForPattern` をみると、

```cpp
      case PatternKind::Typed: {
        auto typedPattern = cast<TypedPattern>(pattern);
        // FIXME: Need a better locator for a pattern as a base.
        Type openedType = CS.openUnboundGenericType(typedPattern->getType(), //そのまま返す
                                                    locator);
        return openedType;
      }
```

`openUnboundGenericType` とかやってるが、パースの段階で`Int` と具体的にわかっているので特に何もおこらずそのまま返される。

## まとめ

`let a: Int = 42` の場合の `ConstraintKind::Conversion`の型は `Contextual Type` によって決まるわけではなく、普通に `TypedPattern` というAST に設定された型を元に決まる。
