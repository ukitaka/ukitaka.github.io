---
layout: post
title:  Swiftの型システムを読む その12 - LLDBによるデバッグ&オリジナルデバッグオプションとプリントデバッグ
---

「読む」のタイトルからも分かる通り当初はコードリーディングする(+デバッグオプションでできる範囲)だけにしようと思ってたけれど、やっぱり動かさないとわからないことが多くてだんだんビルドする機会が増えてきた気がする。

もちろん触れるようになるのは悪いことではないけどやっぱり時間がめっちゃかかる点が辛いですね。。

今回はlldbを使ったビルド方法についてメモ。
これできれば一回ビルドすれば済むのでめっちゃ楽。

## LLDBを使ったデバッグ

Swift部分のみDebugできるように`--debug-swift` オプションをつけてビルドをする。

```
$ ./utils/build-script -R --debug-swift
```

ビルドが完了すると`Ninja-ReleaseAssert+swift-DebugAssert` 以下に成果物ができている。

```
% ../build/Ninja-ReleaseAssert+swift-DebugAssert/swift-macosx-x86_64/bin/swift --version
Swift version 4.1-dev (LLVM fe49d8f2ca, Clang b227f55990, Swift aa5418dd82)
Target: x86_64-apple-darwin16.7.0
```


これを`lldb` 経由で起動する。


```
$ lldb -- ../build/Ninja-ReleaseAssert+swift-DebugAssert/swift-macosx-x86_64/bin/swift hoge.swift
(lldb) target create "../build/Ninja-ReleaseAssert+swift-DebugAssert/swift-macosx-x86_64/bin/swift"
Current executable set to '../build/Ninja-ReleaseAssert+swift-DebugAssert/swift-macosx-x86_64/bin/swift' (x86_64).
(lldb) settings set -- target.run-args  "hoge.swift"
```


`b ファイル名:行数`  でブレークポイントを設定できる。
たとえばCSApply.cppの`coerceToType`にブレークポイントを置いてみる。

```
(lldb) b CSApply.cpp:5961
Breakpoint 1: where = swift`(anonymous namespace)::ExprRewriter::coerceToType(swift::Expr*, swift::Type, swift::constraints::ConstraintLocatorBuilder, llvm::Optional<swift::Pattern*>) + 53 at CSApply.cpp:5961, address = 0x000000010183dbd5
```

その状態で`run`で実行する。

```
(lldb) run
```


関係ないところで止まったりするので`continue` で進めつつ目的の箇所に到達する。

```
(lldb) continue
Process 54004 resuming
Process 54004 stopped
* thread #2, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
    frame #0: 0x000000010183dbd5 swift`(anonymous namespace)::ExprRewriter::coerceToType(this=0x00007fff5fbf1af8, expr=0x000000010a18b3e0, toType=Type @ 0x00007fff5fbf0768, locator=ConstraintLocatorBuilder @ 0x00007fff5fbf13b8, typeFromPattern=Optional<swift::Pattern *> @ 0x00007fff5fbf13a8) at CSApply.cpp:5961
   5958	Expr *ExprRewriter::coerceToType(Expr *expr, Type toType,
   5959	                                 ConstraintLocatorBuilder locator,
   5960	                                 Optional<Pattern*> typeFromPattern) {
-> 5961	  auto &tc = cs.getTypeChecker();
   5962
   5963	  // The type we're converting from.
   5964	  Type fromType = cs.getType(expr);
Target 0: (swift) stopped.
```

たとえばスタックトレースを出したければ`bt` を使う。これで誰がこの関数を呼んだか確認できる。

```
(lldb) bt
* thread #2, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
  * frame #0: 0x000000010183dbd5 swift`(anonymous namespace)::ExprRewriter::coerceToType(this=0x00007fff5fbf1af8, expr=0x000000010a18b3e0, toType=Type @ 0x00007fff5fbf0768, locator=ConstraintLocatorBuilder @ 0x00007fff5fbf13b8, typeFromPattern=Optional<swift::Pattern *> @ 0x00007fff5fbf13a8) at CSApply.cpp:5961
    frame #1: 0x00000001018448c1 swift`(anonymous namespace)::ExprRewriter::buildMemberRef(this=0x00007fff5fbf1af8, base=0x000000010a18b3e0, openedFullType=Type @ 0x00007fff5fbf12f8, dotLoc=SourceLoc @ 0x00007fff5fbf12f0, member=0x000000010a18b920, memberLoc=(LocationInfo = 0x0000000109e0faf8, NumArgumentLabels = 0), openedType=Type @ 0x00007fff5fbf12e8, locator=ConstraintLocatorBuilder @ 0x00007fff5fbf1ad8, memberLocator=ConstraintLocatorBuilder @ 0x00007fff5fbf1ab8, Implicit=true, functionRefKind=SingleApply, semantics=Ordinary, isDynamic=false) at CSApply.cpp:934
    frame #2: 0x00000001018433e1 swift`swift::TypeChecker::callWitness(this=0x00007fff5fbf6c88, base=0x000000010a18b3e0, dc=0x000000010a190930, protocol=0x000000010a14e3a0, conformance=ProtocolConformanceRef @ 0x00007fff5fbf1918, name=DeclName @ 0x00007fff5fbf1910, arguments=MutableArrayRef<swift::Expr *> @ 0x00007fff5fbf2d90, brokenProtocolDiag=(ID = builtin_integer_literal_broken_proto)) at CSApply.cpp:7906
    frame #3: 0x00000001018676d8 swift`(anonymous namespace)::ExprRewriter::convertLiteral(this=0x00007fff5fbf3860, literal=0x000000010a18b390, type=Type @ 0x00007fff5fbf3190, openedType=Type @ 0x00007fff5fbf3188, protocol=0x000000010a14e030, literalType=(anonymous namespace)::ExprRewriter::TypeOrName @ 0x00007fff5fbf3180, literalFuncName=DeclName @ 0x00007fff5fbf3178, builtinProtocol=0x000000010a14e3a0, builtinLiteralType=(anonymous namespace)::ExprRewriter::TypeOrName @ 0x00007fff5fbf3170, builtinLiteralFuncName=DeclName @ 0x00007fff5fbf3168, isBuiltinArgType=0x0000000000000000, brokenProtocolDiag=(ID = integer_literal_broken_proto), brokenBuiltinProtocolDiag=(ID = builtin_integer_literal_broken_proto))(swift::Type), swift::Diag<>, swift::Diag<>) at CSApply.cpp:6595
    frame #4: 0x0000000101868e47 swift`(anonymous namespace)::ExprRewriter::handleIntegerLiteralExpr(this=0x00007fff5fbf3860, expr=0x000000010a1909e8) at CSApply.cpp:1776
    frame #5: 0x000000010185c15d swift`(anonymous namespace)::ExprRewriter::visitIntegerLiteralExpr(this=0x00007fff5fbf3860, expr=0x000000010a1909e8) at CSApply.cpp:1819
    frame #6: 0x000000010185b254 
... 
```

Xcodeでもおなじみの`po` コマンドを使って変数を確認したり関数を呼び出したりもできる。AST系のクラスにはだいたい`dump()` か`print()` が生えているのでそれらを使えばOK。

```
(lldb) po expr->dump()
(type_expr implicit type='Int.Type' location=hoge.swift:1:9 range=[hoge.swift:1:9 - line:1:9] typerepr='Int')
```


そのほかのコマンドは`help` と打てばでてくる。

### (参考) lldbのGUIモード

もっとかっこよく表示したいというあなたのために、`gui`というコマンドが用意されていて、インタラクティブに使える。

```
(lldb) gui
```

<img width="1904" alt="スクリーンショット 2017-11-29 17.38.58.png (814.0 kB)" src="https://img.esa.io/uploads/production/attachments/2245/2017/11/29/2884/73149724-3c7e-4764-b4d5-fee275561477.png">



## プリントデバッグ

上記ができればほぼ問題ないし、わざわざコードに変更を入れて長いビルド時間を待つ必要もないが、一応プリントデバッグの術を書いておく。

上に書いた`dump`や`print`を使っていくが、工夫せずにコード中にプリントデバッグ用のコードを書くと、stdlibのビルド中にもプリントされてしまい使い物にならない。

ここでは`-debug-ukitaka` のようなオプションを追加して、それが有効なときだけプリントするようにしてみる。

まず、`swift/include/swift/Basic/LangOptions.h` に有効・無効を管理するboolを追加する。

```cpp
/// ukitaka debug flg
bool EnableUkitakaDebug = false;
```

`include/swift/Option/FrontendOptions.td` に追加したいオプションを追加する。

```cpp
def debug_ukitaka : Flag<["-"], "debug-ukitaka">,
   HelpText<"Debug option for ukitaka">;
```

`lib/Frontend/CompilerInvocation.cpp` でオプションを拾って最初のboolに格納する。

```cpp
Opts.EnableUkitakaDebug |= Args.hasArg(OPT_debug_ukitaka);
```

あとは適当にフラグを見て`dump`なり`print`なりをする。
`LangOpt`は`ASTContext`にぶら下がっていて、だいたい`getASTContext`のような関数で取得できる。

```cpp
// こんな感じ
if(getASTContext().LangOpts.EnableUkitakaDebug) {
    expr->dump();
}
```

## まとめ

やはりデバッガが使えると捗りますね。。。大人しく最初から使ってればよかった。

