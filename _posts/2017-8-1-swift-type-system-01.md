---
layout: post
title: Swiftの型システムを読む その1
---

今年はTaPLを割と読み進められていて、[型推論をSwiftで実装](https://github.com/ukitaka/TypeSystem)したり、[存在型についてSwiftで考えてみたり](http://qiita.com/ukitaka/items/a993b5d7ed5ae84b1b52) している。
そろそろ実際の言語の型システムも読めそうな気持ちになってきたので、なんか読んでみようと思い、一番よく使うSwiftか一番好きなScalaか一番興味があって型システムもイケてそうなRustあたりで迷って、結局Swiftを読んでいる。
これは読みながら書いたメモ。

## 必要な事前知識など
ある程度TaPLを読み進めてしまったので、どこがわからなかったか忘れてしまったけれど、最低限 Hindley-Milnerのような制約ベースの型推論がどんなものかわかっていれば問題ないと思う(たぶん…)

例えば`let a = 1` の型推論の基本的な流れはこんな感じ。
ソースコードでも出てくるような単語は英語も併記。

1. 型が明記されてないところを型変数(TypeVar) T1などで埋める
```
let a: T1 = 1: T2
```

2. 制約(Constraint)を作る
```
T1 == T2 
T2 == Int (1が整数リテラルなので)
```

3. 制約を単一化(Unify, Simplify)する
要は連立方程式を解く(Solve)。
```
T1 = Int
T2 = Int
```

(これはイメージで、Swiftの言語機能にはサブタイピングやGenericsなどもあり実際の挙動はもう少し複雑。)


## ドキュメントについて

型システム周りのドキュメントは一応あって、[docs/TypeChecker.rst](https://github.com/apple/swift/blob/master/docs/TypeChecker.rst) にある。
まずはざっと読んで見ると良いかもしれない。

## Swift開発環境構築について
 [Swiftコンパイラ開発環境構築](http://qiita.com/rintaro/items/2047a9b88d9249459d9a)を参考。
コード読むだけならビルドする必要はないが、やっぱりXcodeがあると便利なので、チェックアウトとXcodeのプロジェクトを生成するところまで。

```
$ mkdir swift-lang
$ cd swift-lang
$ git clone git@github.com:apple/swift.git
$ cd swift
$ utils/update-checkout --clone
$ utils/build-script -x --skip-build
```

たくさんファイルがあって邪魔なので、必要なければXcodeから`swiftAST` と `swiftSema` 以外のフォルダはRemove referencesしてもたぶん大丈夫。いまのところ型システムを読むだけならばそれで困っていない。

## 覚えておくべきコマンド 3つ
Swiftの型システムを読む上で知っておくべきコマンドが3つある

1. `swift -frontend -dump-parse ファイル名.swift`
2. `swift -frontend -typecheck -debug-constraints ファイル名.swift`
3. `swift -frontend -dump-ast ファイル名.swift`

1, 2, 3はコンパイラの処理順通り。
2が一番大事で、制約生成やソルバーの挙動をdumpする。基本的にはこれを使えばOK。

1と3はいずれもASTを表示するものであるが、フェーズが違う。
1はソースファイルからシンプルにパースされたものがdumpされるが、3は型の再構築や型チェックが行われた状態。`42`などのリテラルなども `Int.init(_builtinIntegerLiteral: 42)` のような呼び出しに置き換えられている。

基本は2を使って挙動を確認しつつ、型付けされる前のASTを確認したければ1を、型チェック後のASTを確認したければ3を使う。

`-debug-constraints` の出力の読み方はまた今度書く(たぶん)

