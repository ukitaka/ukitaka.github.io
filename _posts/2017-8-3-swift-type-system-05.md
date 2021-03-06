---
layout: post
title: Swiftの型システムを読む その5
---


今回は、`-debug-constraints` の出力の`Contextual Type`というものについて調べたメモ。

## `Contextual Type` が出てくる時 / 出てこない時

まだ何かは全然わからないけれど、とりあえず出力される場合とそうでない場合があることが確認できたのでみてみる。

相変わらず ` let a = 42` を例に見ていく。


+ `let a: Int = 42` のときは`Contextual Type`あり

```
---Constraint solving for the expression at [study/s0007.swift:1:14 - line:1:14]---
---Initial constraints for the given expression---
(integer_literal_expr type='$T0' location=study/s0007.swift:1:14 range=[study/s0007.swift:1:14 - line:1:14] value=1)
Score: 0 0 0 0 0 0 0 0 0 0 0 0 0
Contextual Type: Int at [study/s0007.swift:1:8 - line:1:8]
Type Variables:
(略)
```


+ `let a = 42` のときは`Contextual Type`なし


```
---Constraint solving for the expression at [study/s001.swift:1:9 - line:1:9]---
---Initial constraints for the given expression---
(integer_literal_expr type='$T0' location=study/s001.swift:1:9 range=[study/s001.swift:1:9 - line:1:9] value=42)
Score: 0 0 0 0 0 0 0 0 0 0 0 0 0
Type Variables:
(略)
```


ある場合は、`42`の整数リテラルのASTについて`Contextual Type`がでているみたい。
なんとなく左辺で型を明示したことによって起こっていそうだということはわかる。


## ASTを見てみる

それぞれの場合でASTを確認して見ると

+ `let a: Int = 42` のとき

```
(source_file
  (top_level_code_decl
    (brace_stmt
      (pattern_binding_decl
        (pattern_typed
          (pattern_named 'a')
          (type_ident
            (component id='Int' bind=none)))
        (integer_literal_expr type='<null>' value=1))
))
  (var_decl "a" type='<null type>' let storage_kind=stored))
```

+ `let a = 42` のとき

```
(source_file
  (top_level_code_decl
    (brace_stmt
      (pattern_binding_decl
        (pattern_named 'a')
        (integer_literal_expr type='<null>' value=42))
))
  (var_decl "a" type='<null type>' let storage_kind=stored)
```


型を明示した場合、`pattern_binding_decl` が `pattern_typed` というパターンで構成されていることが確認できる。

## TypeChecker::typeCheckBinding を見てみる

`TypeChecker::typeCheckBinding` に型を明示的に書いた場合とそうでない場合の違いが見て取れた。

`pattern` (つまり左辺)が`TypedPattern` (つまり型が明記されている)場合は`contextualType` を設定して、

```cpp
    if (auto *typedPattern = dyn_cast<TypedPattern>(pattern)) {
      const Pattern *inner = typedPattern->getSemanticsProvidingPattern();
      if (isa<NamedPattern>(inner) || isa<AnyPattern>(inner))
        contextualType = typedPattern->getTypeLoc();
    }
```

`initializer` (つまり右辺の42)について型チェックを行う時に `contextualType` を渡している。

```cpp
  auto resultTy = typeCheckExpression(initializer, DC, contextualType,
                                      contextualPurpose, flags, &listener);
```


**つまり複数の`Expr` 等から構成されるASTにおいて、他の子ノードから別の子ノードの型が決まる場合に使われるのが `Contextual Type` ということらしい**

もう少し挙動を追ってみる。


## 渡された contextualTypeを追う
`TypeChecker::typeCheckExpression` にたどり着いたのち、`ConstraintSystem` に `setContextualType` される。

```cpp
cs.setContextualType(expr, convertType, convertTypePurpose);
```

この行で上書きされて消される。`ConvertTypeIsOnlyAHint` は `TypeChecker::typeCheckBinding`で設定されている。

```cpp
  if (options.contains(TypeCheckExprFlags::ConvertTypeIsOnlyAHint))
    convertType = TypeLoc();
```

この例ではその後 `getContextualType`で取り出して何かに使われることはなかった。どうやら `Array` `Dictionary` `Closure` などで使われるようなので少し見て見る。

## 例: `let a: [Double] = [1, 2, 3]`

`let a: [Double] = [1, 2, 3]` を例に見て見る。

ASTはこんな感じ。

```
(source_file
  (top_level_code_decl
    (brace_stmt
      (pattern_binding_decl
        (pattern_typed
          (pattern_named 'a')
          (type_array
            (type_ident
              (component id='Double' bind=none))))
        (array_expr type='<null>'
          (integer_literal_expr type='<null>' value=1)
          (integer_literal_expr type='<null>' value=2)
          (integer_literal_expr type='<null>' value=3)))
))
  (var_decl "a" type='<null type>' let storage_kind=stored))
```


`array_expr` の型チェック時に、`Contextual Type` が使われていることが確認できる。

```
(array_expr type='[Double]' location=study/s011.swift:1:19 range=[study/s011.swift:1:19 - line:1:25]
  (integer_literal_expr type='$T0' location=study/s011.swift:1:20 range=[study/s011.swift:1:20 - line:1:20] value=1)
  (integer_literal_expr type='$T1' location=study/s011.swift:1:22 range=[study/s011.swift:1:22 - line:1:22] value=2)
  (integer_literal_expr type='$T2' location=study/s011.swift:1:24 range=[study/s011.swift:1:24 - line:1:24] value=3))
Score: 0 0 0 0 0 0 0 0 0 0 0
Contextual Type: [Double] at [study/s011.swift:1:8 - line:1:15]
Type Variables:
  #0 = $T0 [inout allowed]
  #1 = $T1 [inout allowed] equivalent to $T0
  #2 = $T2 [inout allowed] equivalent to $T0
```


実際 `array_expr` の制約生成のタイミングで、

```cpp
auto contextualType = CS.getContextualType(expr);
```

```cpp
CS.addConstraint(ConstraintKind::LiteralConformsTo, contextualType,
                 arrayProto->getDeclaredType(),
                 locator);
```

となっている。

## まとめ

上にも書いたこれが大体正しそう。

> つまり複数の`Expr` 等から構成されるASTにおいて、他の子ノードから別の子ノードの型が決まる場合に使われるのが `Contextual Type` ということらしい

ただし、`let a: Int = 42` の場合は使われておらず、そうなると `Conversion` の制約に `Int` を誰が設定しているのか気になるので、次回調べる。

