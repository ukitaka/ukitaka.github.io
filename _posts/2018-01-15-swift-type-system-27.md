---
layout: post
title:   Swiftの型システムを読む その27 - ImplicitlyUnwrappedOptionalとPotentialBindings
---

挙動ベースの軽めのメモ。
今回は以下のコードがどのように動いているかをみる。

```swift
let iuoInt: Int! = 1
let t =  type(of: iuoInt)
print(t) // Optional<Int>
```

## ImplicitlyUnwrappedOptional<T>のDynamic TypeはOptional<T>

`iuoInt`は静的には`ImplicitylyUnwrappedOptional<Int>`型だが、`type(of: iuoInt)`は`Optional<Int>`と表示される。

IUOは基本的にはSILGenの時点で(つまり型チェックが終わった段階で)ただのOptionalにされてしまうため、SILにIUOは登場しない。

```swift
let i: Int! = 3
```

```swift
sil_stage raw

import Builtin
import Swift
import SwiftShims

// i
sil_global hidden @_T04iuo31iSQySiGv : $Optional<Int>
```

つまり`ImplicitlyUnwrappedOptional<T>`のDynamic Typeは`Optional<T>`であり、この挙動自体は自然なものだと思っていた。

**が、実はSIL生成時でなく型チェック時にこの挙動が決められていたことに気づいた。**

## 型推論とPotentialBindingsを覗く

```swift
let iuoInt: Int! = 1
let _ =  type(of: iuoInt)
```

上記のコードを`-debug-constraints`でみてみると型推論の時点で`type(of: iuoInt)`の返り値が`Optional<Int>`になっていることがみてわかる。つまりこの式は静的に`Optional<Int>`である。

```
---Type-checked expression---
(metatype_expr type='Int?.Type' location=iuo.swift:2:10 range=[iuo.swift:2:10 - line:2:25]
  (optional_evaluation_expr implicit type='Int?' location=iuo.swift:2:19 range=[iuo.swift:2:19 - line:2:19]
    (inject_into_optional implicit type='Int?' location=iuo.swift:2:19 range=[iuo.swift:2:19 - line:2:19]
      (bind_optional_expr implicit type='Int' location=iuo.swift:2:19 range=[iuo.swift:2:19 - line:2:19] depth=0
        (declref_expr type='Int!' location=iuo.swift:2:19 range=[iuo.swift:2:19 - line:2:19] decl=iuo.(file).iuoInt@iuo.swift:1:5 direct_to_storage function_ref=unapplied)))))
```

PotentialBinding周辺の動きをみてみる。

```
($T1 bindings=(supertypes of) Int? (supertypes of) Int)
Active bindings: $T1 := Int? $T1 := Int
(trying $T1 := Int?
  ($T4 bindings=(supertypes of) Int?.Type)
  Active bindings: $T4 := Int?.Type
  (trying $T4 := Int?.Type
    (found solution 0 0 0 0 0 0 0 0 0 0 0 0 0)
  )
)
(trying $T1 := Int
  (increasing score due to force of an implicitly unwrapped optional)
  (solution is worse than the best solution)
)
```

1行目は`determineBestBindings`の中で`PotentialBindings`の数だけ出力される。今回は`T1`に対するbindingしかないのでそれが表示されている。

```
($T1 bindings=(supertypes of) Int? (supertypes of) Int)
```


2行目は`tryTypeVariableBindings`の中で使われている`PotentialBindings`が出力される。

```
Active bindings: $T1 := Int? $T1 := Int
```

`Int`と`Int?`の両方がためされ、`Int`の方は`SK_ForceUnchecked`というスコアがincreaseされるために選ばれずに`Int?`として型チェックが通る。

## PotentialBindingの実装を再び見る
`getPotentialBindings`で`T!`の場合に`T`と`T?`がPotentialBindingとして採用される流れを見てみる。

```cpp
// Don't deduce IUO types.
Type alternateType;
bool adjustedIUO = false;
if (kind == AllowedBindingKind::Supertypes &&
    constraint->getKind() >= ConstraintKind::Conversion &&
    constraint->getKind() <= ConstraintKind::OperatorArgumentConversion) {
  auto innerType = type->getWithoutSpecifierType();
  if (auto objectType =
          lookThroughImplicitlyUnwrappedOptionalType(innerType)) {
    type = OptionalType::get(objectType);
    alternateType = objectType;
    adjustedIUO = true;
  }
}
```

コメントに「IUOには推論しないよ」と書いてありますね〜。
代わりに`Optional<T>`が追加される。

```cpp
if (exactTypes.insert(type->getCanonicalType()).second)
  result.addPotentialBinding({type, kind, None},
                             /*allowJoinMeet=*/!adjustedIUO);
```

`allowJoinMeet`がfalseの場合(つまり、上でIUOの代わりに`Optional<T>`を採用した場合)、`T`自身も追加でPotentialBindingsに追加される。

```cpp
void ConstraintSystem::PotentialBindings::addPotentialBinding(PotentialBinding binding, bool allowJoinMeet) {
	// なにもしない
	// (略)
	// 追加
	Bindings.push_back(std::move(binding));
}
```

## まとめ

`T!`については`T?`と`T`がPotentialBindingとして存在して、制約がなければ`T?`が優先的に使用されるため、`type(of: iuoInt)`は静的に`Optional<Int>`となり、出力も`Optional<Int>`としてされる。

ちなみに面白いのがExistentialに入れた場合、パッケージ化は静的な型に基づいて行われるために存在型の証人型は`ImplicitlyUnwrappedOptional<Int>`となり、表示もそれになる。
なんだかなぁ。。

```swift
let iuoI: Int! = 1 
let any: Any = iuoI

print(type(of:iuoI)) // Optional<Int>
print(type(of:any))  // ImplicitlyUnwrappedOptional<Int>
```
