---
layout: post
title:  Swiftの型システムを読む その20 - UnresolvedDotExprの型推論(制約生成〜Simplify)
---

## 前回の復習
`E.a`のように呼び出し元の式(`BaseExpr`, `SubExpr`)の型がわかっていいる状態でのメンバーの参照がある場合の`.a`は`UnresolvedDotExpr`として扱われる。

##  制約生成
`ConstraintGenerator::visitUnresolvedDotExpr`で制約が生成される。

 `self.init`の場合かどうかで分かれていて、基本的にはそのまま`addMemberRefConstraints`が呼ばれる。

```cpp
if (CS.TC.getSelfForInitDelegationInConstructor(CS.DC, expr)) { 
... 
	return methodTy;
}
return addMemberRefConstraints(expr, expr->getBase(), expr->getName(),
                             expr->getFunctionRefKind());

```


```cpp
Type addMemberRefConstraints(Expr *expr, Expr *base, DeclName name,
                             FunctionRefKind functionRefKind)
```

`E.a` の場合、

+ `expr` は`UnresolvedDotExpr`, つまり`.a`を指す。
+ `base`は`UnresolvedDeclRefExpr`, つまり`E`を表す。
+ `name`は`a`を指す。

そこから、`ConstraintSystem::addValueMemberConstraint`が呼ばれる。
その際に`memberTy`、つまり`.a`の型が型変数で置かれる。
メソッドであれば関数型、プロパティであればそのプロパティの型。

```cpp
auto baseTy = CS.getType(base);
auto tv = CS.createTypeVariable(
            CS.getConstraintLocator(expr, ConstraintLocator::Member),
            TVO_CanBindToLValue |
            TVO_CanBindToInOut);
CS.addValueMemberConstraint(baseTy, name, tv, CurDC, functionRefKind,
    CS.getConstraintLocator(expr, ConstraintLocator::Member));
```

```cpp
void addValueMemberConstraint(Type baseTy, DeclName name, Type memberTy,
                                DeclContext *useDC,
                                FunctionRefKind functionRefKind,
                                ConstraintLocatorBuilder locator)
```


`ConstraintKind`としては`ValueMember`というkindになる。

## Simplify

いつも通り`addXXXConstraint`されたらできるだけSimplifyされてからConstraintSystemに記録される。Memberの制約の場合は`ConstraintSystem::simplifyMemberConstraint`がそれにあたる。
	

大まかな挙動としては
1. `ConstraintSystem::performMemberLookup`でメンバーを見つけようとする
2. 見つからなかったら`addUnresolvedConstraint`される。つまり`InactiveConstraints`と`ConstraintGraph`に入る。(見つからなくて成功する場合とは…?基本不正な呼び出しと考えて良い？)
3. 見つかったら候補がすべて`addOverloadSet`される。


## ConstraintSystem::addOverloadSet

```cpp
void ConstraintSystem::addOverloadSet(Type boundType,
                                      ArrayRef<OverloadChoice> choices,
                                      DeclContext *useDC,
                                      ConstraintLocator *locator,
                                      OverloadChoice *favoredChoice)
```

見つかった候補は個数に関わらずここに渡されてくる。

1. 候補が1つの場合はそのままオーバーロードが解決されたとして話が進められる。
2. 候補が複数の場合は`addDisjunctionConstraint`される。

`-debug-constraints`でみてみると確かに`Overload choices:`の項目に出ていることがわかる。

```
Overload choices:

locator@0x7fc2bf0278a8 [UnresolvedDot@unr2.swift:6:5 -> member] with unr2.(file).Dog.bark@unr2.swift:2:8 as Dog.bark: (String) -> ()
```


また複数の場合は`Disjunction`な制約が`addUnresolvedConstraint`され、あとでどこかで選ばれるのだろう。今度調べる。

```cpp
void addDisjunctionConstraint(ArrayRef<Constraint *> constraints,
                              ConstraintLocatorBuilder locator,
                              RememberChoice_t rememberChoice = ForgetChoice,
                              bool isFavored = false) {
  auto constraint =
    Constraint::createDisjunction(*this, constraints,
                                  getConstraintLocator(locator),
                                  rememberChoice);
  if (isFavored)
    constraint->setFavored();

  addUnsolvedConstraint(constraint);
}
```

## ConstraintSystem::resolveOverload

名前がちょっとややこしいが、これはオーバーロードがすでに選択されて「それを解として使うよ」みたいなときに呼ばれるメソッド。

```cpp
void ConstraintSystem::resolveOverload(ConstraintLocator *locator,
                                       Type boundType,
                                       OverloadChoice choice,
                                       DeclContext *useDC)
```

 `boundType`は型変数が入っている。

```
(lldb) po boundType->dump()
(type_variable_type id=0)
```

`choice`は名前の通り選択されたオーバーロードを表している。

```
(lldb) po choice.getDecl()->dump()
(func_decl "bark(_:)" interface type='(Dog) -> (String) -> String' ...
```

```
(lldb) po choice.getKind()
Decl
```


ここから`getTypeOfMemberReference`や`getTypeOfReference`が呼び出されて`refType`が求められる。これらの中身は単体でネタにできるくらいデカイのでまた次回以降で解説する。


```
(lldb) po refType->dump()
(function_type escaping
  (input=paren_type
    (struct_type decl=Swift.(file).String))
  (output=struct_type decl=Swift.(file).String))
```

`refType`が求まると、それについて改めて`Bind`で`addConstraint`される。

```cpp
 addConstraint(ConstraintKind::Bind, boundType, refType, locator);
```


## まとめ

1. メンバーの型が型変数`T`で置かれる
2. resolveされて`T := A -> B` みたいにBindされる

制約生成までででかくなってしまったので今回はここまで。
