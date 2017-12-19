---
layout: post
title:  Swiftの型システムを読む その17 - Constraint Propagation Algorithm
---

最近のこのシリーズは1つの話題をなんかうまいことまとめようとして時間をかけすぎて結局アウトプットできてない感あるので、1メソッド = 1記事とかでもいいのでもっと雑に書こうと思った。。ちゃんと書くのはQiitaでやる。

最近はNameLookup周りを読んでいるのだけど、いい感じにまとまらないので今回はいったん離れて、マイナーそうだけど`CSXXXX.cpp`と名前が付いているため`CSApply`や`CSSolver`などと合わせていつも視界に入ってくる`CSPropagate`を読んでみる。

## CSPropagateとは？

Propagate = 伝播らしい。日本語だと「制約伝播」とかいうのだろうか？
「Constraint Propagate」などとググってみてやっとなにをするものなのか理解できた。

HM型推論では制約の集合を単一化(Swiftの実装名だと**Solve**)して解(Solution)を見つけるフェーズがあるが、その際に**「不要な解の候補を削除する」ことを指して「Constraint Propagation 」**と呼ぶらしい。


## オプションを有効にしないと使われない

`ConstraintSystem::solve`から`ConstraintSystem::solveRec`が呼ばれる前に`propagateConstraints()`が呼ばれるが、見ての通り`TC.Context.LangOpts.EnableConstraintPropagation`が有効でないと実行されない。

```cpp
bool ConstraintSystem::solve(Expr *const expr,
                             SmallVectorImpl<Solution> &solutions,
                             FreeTypeVariableBinding allowFreeTypeVariables) {
  // (略)

  // If the experimental constraint propagation pass is enabled, run it.
  if (TC.Context.LangOpts.EnableConstraintPropagation)
    if (propagateConstraints())
      return true;

  // Solve the system.
  solveRec(solutions, allowFreeTypeVariables);
```

こんな感じ？

```
% swift -frontend -typecheck -propagate-constraints hoge.swift
```

## 具体的にはなにをする？

```
// Do a form of constraint propagation consisting of examining
// applicable function constraints and their associated disjunction of
// bind overload constraints. Disable bind overload constraints in the
// disjunction if they are inconsistent with the rest of the
// constraint system. By doing this we can eliminate a lot of the work
// that we'll perform in the constraint solver.
```

コメントを読むと、基本的にはオーバーロードの制約を整理するみたい。

[https://github.com/apple/swift/pull/10825](https://github.com/apple/swift/pull/10825)

まだ作業中みたいだけど、[このあたりの](https://github.com/rudkx/swift/blob/9e6dac47e956285c2b8e02d9b5ddaf6d45db6e2c/test/Sema/complex_expressions.swift)複雑なオーバーロードが解決できるようになるっぽい。


## まとめ

全然具体的なロジックは読んでないけど、なんとなくやってることはわかったので、続きはまたいつか。。

