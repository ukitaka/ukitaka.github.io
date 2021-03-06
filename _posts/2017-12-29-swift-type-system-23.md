---
layout: post
title:   Swiftの型システムを読む その23 - ConstraintSystemにおけるConstraintGraphの使われ方
---

前回はConstraintGraph単体の実装を見たので、今回はConstraintSystemでの使われ方を見てみる。

## ConstraintGraphの使われ方 概要

1. 制約生成のタイミングでsimplifyしても解けなかった制約、つまり型変数を含む制約をグラフに登録しておく。
2. solve時にはBindもしくはDisjunctionから一つ選んで制約を追加 -> グラフを縮約の流れを続ける

## ConstraintGraph::addConstraint

一番よく使われるのは`ConstraintSystem::addUnsolvedConstraint` によってグラフに追加されるパターン。制約生成時に`ConstraintSystem`で`addConstraint`された場合まずできるだけsimplifyしてみて、それで解けた場合は`ConstraintRestrictions`等に記録して終わりみたいな流れなのだが、型変数を含んでいる場合は当然解けないので一旦`addUnsolvedConstraint` 経由でグラフに登録する。

```cpp
/// \brief Add a newly-generated constraint that is known not to be solvable
/// right now.
void addUnsolvedConstraint(Constraint *constraint) {
  // We couldn't solve this constraint; add it to the pile.
  InactiveConstraints.push_back(constraint);

  // Add this constraint to the constraint graph.
  CG.addConstraint(constraint);

  // Record this as a newly-generated constraint.
  if (solverState)
    solverState->addGeneratedConstraint(constraint);
}
```

またSolve時`solveSimplify`において`Disjunction`を試してく際にもグラフに追加される。

```cpp
// Put the disjunction constraint back in its place.
InactiveConstraints.insert(afterDisjunction, disjunction);
CG.addConstraint(disjunction);
```


## ConstraintGraph::removeConstraint

`solve` 時の `simplify`で`simplifyConstraint`が成功すると、グラフから取り除かれる。主な用途はそれくらい。

```cpp
bool ConstraintSystem::simplify(bool ContinueAfterFailures) {
    switch (simplifyConstraint(*constraint)) {
    case SolutionKind::Error:
        // 略
    case SolutionKind::Solved:
      // Remove the constraint from the constraint graph.
      CG.removeConstraint(constraint);

    case SolutionKind::Unsolved:
        // 略
    }
  }
}
```

## ConstraintGraph::optimize / mergeNodes / contractEdges

Solve時にはいろいろ制約を試しながら進めることになるが、制約追加後にグラフの縮約を行う。
具体的には`solveRec`で`optimize`が呼ばれている。

```cpp
// Contract the edges of the constraint graph.
CG.optimize();
```

縮約後に改めてcomponentを`computeConnectedComponents`で計算してみて、1つであればそのまま解きにいく。

```cpp
unsigned numComponents = CG.computeConnectedComponents(typeVars, components);
```

```cpp
// If we don't have more than one component, just solve the whole
// system.
if (numComponents < 2) {
  SmallVector<Constraint *, 8> disjunctions;
  collectDisjunctions(disjunctions);

  return solveSimplified(selectDisjunction(disjunctions), solutions,
                         allowFreeTypeVariables);
}
```

componentが2つ以上ある場合はそれぞれのcomponentについてSolutionを見つけ、すべての組み合わせを試して全体の解を見つける。

## まとめ

1. 制約生成のタイミングでsimplifyしても解けなかった制約、つまり型変数を含む制約をグラフに登録しておく。
2. solve時にはBindもしくはDisjunctionから一つ選んで制約を追加 -> グラフを縮約の流れを続ける

いまいちOrphanedConstraintsの使われ方を追いきれなかったので、もう少しコードリーディングを進めてから`solve` / `solveRec` / `solveSimplify`を読む際に追ってみる。
