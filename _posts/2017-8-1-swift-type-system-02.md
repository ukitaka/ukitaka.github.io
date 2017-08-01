---
layout: post
title: Swiftの型システムを読む その2
---

今回は主にAST周りについて、コードリーディング時に知っておくべきことをまとめる。

## swiftのASTについて
主要なものが4つ。

+ Expr (式)
+ Stmt (文)
+ Decl (class, structなどの宣言)
+ Pattern (パターン)

他にもいくつかあるけれど、とりあえずこの4つを押さえておけば良さそう。


### Exprのクラス図

すべてExprのサブクラス。(Exprとの継承関係は省略)
多いのであまり意味ない。。

![https://img.esa.io/uploads/production/attachments/2245/2017/08/01/2884/938ef908-b862-4822-8300-8bb089c0761a.png](https://img.esa.io/uploads/production/attachments/2245/2017/08/01/2884/938ef908-b862-4822-8300-8bb089c0761a.png)

### Stmtのクラス図

![https://img.esa.io/uploads/production/attachments/2245/2017/08/01/2884/781ae70e-5232-4a11-a174-c487fd82572a.png](https://img.esa.io/uploads/production/attachments/2245/2017/08/01/2884/781ae70e-5232-4a11-a174-c487fd82572a.png)

### Declのクラス図

すべてDeclのサブクラス。(Declとの継承関係は省略)

![https://img.esa.io/uploads/production/attachments/2245/2017/08/01/2884/aa567f83-f37d-4e7d-aa05-127577bfcf3d.png](https://img.esa.io/uploads/production/attachments/2245/2017/08/01/2884/aa567f83-f37d-4e7d-aa05-127577bfcf3d.png)

### Patternのクラス図

![https://img.esa.io/uploads/production/attachments/2245/2017/08/01/2884/4c2b33e9-fa12-4c78-bb4d-668fda984901.png](https://img.esa.io/uploads/production/attachments/2245/2017/08/01/2884/4c2b33e9-fa12-4c78-bb4d-668fda984901.png)

## 実際にASTをみてみる

試しに`let a = 42` について [その1](https://blog.waft.me/2017/08/01/swift-type-system-01/)で見た通り、`-dump-parse`を使ってASTを見てみる。

```swift
let a = 42
```

```
$ swift -frontend -dump-parse s001.swift
```

```
(source_file
  (top_level_code_decl
    (brace_stmt
      (pattern_binding_decl
        (pattern_named 'a')
        (integer_literal_expr type='<null>' value=42))
))
  (var_decl "a" type='<null type>' let storage_kind=stored))
```


シンプルだけど `Expr` / `Stmt` / `Decl` / `Pattern` が綺麗に全部出てくる良い例ですね(？)

なんとなく以下のようなことがわかる。

+ トップレベルは brace `{ }` があることになってるらしい。
+ `a = 42` は どうやら`pattern binding decl` らしい。
+ `42`は整数リテラルとしてパースされているけど、まだ具体的な型がわかっていない。

公式ドキュメントに [Swiftの文法まとめ](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/zzSummaryOfTheGrammar.html)があるので、必要に応じてこれも参考にすると良さそう。

## ASTの走査とVisitorの挙動

swiftの実装はC++なのでパターンマッチなんて便利なものはなく、ASTの走査にはVisitorパターンが使われている。
だいたい最初にXcodeでコードジャンプしながら読んでいると`visit`からどこに飛ぶのかちょっとわかりづらいのと、型システムだけでなく全体的に使われているので、あえてメモ。

まず、visitorは `ASTVisitor`というクラスを継承する。
そうすると`Decl`などそれぞれに対する `visit` が使えるようになる。

```cpp
DeclRetTy visit(Decl *D, Args... AA) {
	...
}

ExprRetTy visit(Expr *E, Args... AA) {
  ...
}

StmtRetTy visit(Stmt *S, Args... AA) {
  ...
}

PatternRetTy visit(Pattern *P, Args... AA) {
  ...
}
```

各visitは`Kind` (≒ サブクラス)に応じた`visitXXXX`を呼び出す。
たとえば `VarDecl` であれば `visitVarDecl` を呼び出す、など。 
ここがマクロによって作られているのでちょっと読みづらい。

```cpp
  DeclRetTy visit(Decl *D, Args... AA) {
    switch (D->getKind()) {
#define DECL(CLASS, PARENT) \
    case DeclKind::CLASS: \
      return static_cast<ImplClass*>(this) \
        ->visit##CLASS##Decl(static_cast<CLASS##Decl*>(D), \
                             ::std::forward<Args>(AA)...);
#include "swift/AST/DeclNodes.def"
    }
    llvm_unreachable("Not reachable, all cases handled");
  }
```

なのでもしコードリーディング中に `visit(decl)` のようなコードを見かけたら`visitXXXX` に飛ぶ、というところだけ覚えておけば良さそう。

