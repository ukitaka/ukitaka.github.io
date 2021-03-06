---
layout: post
title:   Swiftの型システムを読む その22 - ConstraintGraphの理論と実装
---

ConstraintGraphは制約のSolveの際に型変数を管理するためのクラス。
名前の通りグラフなのだが、**ハイパーグラフ(hyper graph)**と呼ばれる一般的なグラフを少し拡張したような物になっている。

今回はハイパーグラフについて調べ、実装を読んでみる。

## ハイパーグラフ(hyper graph)とその周辺用語の整理

ハイパーグラフは一般的なグラフと同じようにノードとエッジから構成されるが、エッジに特徴がある。

一般的なグラフのエッジは2つのノード間を結ぶが、ハイパーグラフにおけるエッジは**ハイパーエッジ(hyper edge)**と呼ばれ、**N個のノードを結ぶ。**
(Nは1でもOK)

![1.png (83.7 kB)](https://img.esa.io/uploads/production/attachments/2245/2017/12/27/2884/0a6ccd33-6ad2-4307-a965-6e37fafd88d4.png)


形式的にはノードの集合と、エッジを構成するノードの集合の集合のペアからなる。

### 隣接(Adjacency)

隣接はあるノードからエッジで結ばれているノードのこと。
ハイパーグラフでも大体同じ意味だが、ConstraintGraphでは少し性質がことなる。(後述)

![2.png (57.1 kB)](https://img.esa.io/uploads/production/attachments/2245/2017/12/27/2884/7349c600-cf51-458e-a05f-7d396121ad1e.png)


### 連結成分(Connected Component) 

単にComponentとも。非連結グラフを構成する連結グラフのこと。
要はエッジによって結ばれたかたまり。ハイパーグラフでも同じ。

![3.png (46.6 kB)](https://img.esa.io/uploads/production/attachments/2245/2017/12/27/2884/e8c65f86-f4ea-4288-9970-7f5c29aec57a.png)

### 縮約(Contraction)

一般的なグラフにおいては、グラフからあるエッジをを取り除きそのエッジの両端を一つのノードにまとめることを縮約(Contraction)という。

![4.png (45.6 kB)](https://img.esa.io/uploads/production/attachments/2245/2017/12/27/2884/e47d9511-0511-4eae-ae7e-3b97f707c433.png)


ハイパーグラフにおいては正確な定義はわからないけど、エッジが結んでいるノードの一部を一つのノードのまとめることを言うみたい。

![5.png (69.7 kB)](https://img.esa.io/uploads/production/attachments/2245/2017/12/27/2884/06ce62d5-c6dc-4e7f-a708-1b8dafb8a617.png)


グラフについてはここまでわかればよさそう。
グラフ関係ないけど、同値類(Equivalence Class)とか代表元(Representative)とかもし知らなければ調べておくと読みやすいかも。

## ハイパーグラフとしてのConstraintGraph

`ConstraintGraph`ではノード・エッジが以下のものに対応している。

+ ノードは**型変数**
+ エッジは**その型変数を含むConstraintの集合**

![6.png (56.3 kB)](https://img.esa.io/uploads/production/attachments/2245/2017/12/27/2884/2e31b757-4b78-4eaa-bb6e-d0a47b40e728.png)


具体的に実装を見ていく。ノードについてはそのまま`ConstraintGraphNode`というクラスで実装されている。

```cpp
/// A single node in the constraint graph, which represents a type variable.
class ConstraintGraphNode { ... }
```

各ノードが持っている情報は大まかにこんな感じ。

```cpp
// このノードが表す型変数
TypeVariableType *TypeVar;

// この型変数をもつ制約。
// これがエッジを表す。
SmallVector<Constraint *, 2> Constraints;

// 隣接ノード
SmallVector<TypeVariableType *, 2> Adjacencies;

// 隣接ノードの情報
llvm::SmallDenseMap<TypeVariableType *, Adjacency, 2> AdjacencyInfo;

// 同値類
mutable SmallVector<TypeVariableType *, 2> EquivalenceClass;
```


また、`ConstraintGraph`におけるエッジは「あるNode(型変数)を中心として、その隣接の集まり」で表される。

`ConstraintGraph`自身は主に以下の2つの情報を管理している。

```cpp
// このグラフで扱っている型変数
SmallVector<TypeVariableType *, 4> TypeVariables;

// Orphanedは孤立したみたいな意味
// addConstraintされたものの、型変数を持たないときにここに入ってくる。
// (ただ...それはどんなとき？)
SmallVector<Constraint *, 4> OrphanedConstraints;
```


## ConstraintGraphに対する主な操作一覧

+ `ConstraintGraph::lookupNode`
+ `ConstraintGraph::addConstraint`
+ `ConstraintGraph::removeConstraint`
+ `ConstraintGraph::computeConnectedComponents`
+ `ConstraintGraph::optimize`
+ `ConstraintGraph::getOrphanedConstraints`
+ `ConstraintGraph::takeOrphanedConstraints`
+ `ConstraintGraph::setOrphanedConstraint`
+ `ConstraintGraph::mergeNodes`
+ `ConstraintGraph::bindTypeVariables`
+ `ConstraintGraph::gatherConstraints`

ここからいくつかピックアップして挙動をみてみる。

## lookupNode / addConstraint

`lookupNode`は名前の通り型変数を使って対象のノードを引いてくる。
その際に、その型変数がまだノードして存在しなければ追加してからそれを返す。
実際には型変数を表すクラス`TypeVariableType`の`impl`がノードを持っている形らしい。

```cpp
// Allocate the new node.
auto nodePtr = new ConstraintGraphNode(typeVar);
unsigned index = TypeVariables.size();
impl.setGraphNode(nodePtr);
impl.setGraphIndex(index);

// Record this type variable.
TypeVariables.push_back(typeVar);
```

また`[]`オペレータでもこれが使われる。
`addConstraint`によって`ConstraintGraph`に制約が追加されると、まず上の`lookupNode`が呼ばれてノードが用意される。

そしてそのノードに対して`Constraint`が追加される。
エッジが拡張されるようなイメージ。

```cpp
node.addConstraint(constraint);
```

両方とも型変数の場合には隣接ノードとしても登録される。

![7.png (70.4 kB)](https://img.esa.io/uploads/production/attachments/2245/2017/12/27/2884/86873cf0-c572-4435-ab53-6dcce0b30e6b.png)


## optimize / contractEdges / mergeEquivalenceClasses
ソルバーの中では`optimize`が呼ばれているだけなので実態が分かりづらいが、やっていることはシンプルでひたすら`contractEdges()`を呼んで可能な限り縮約しているだけ。

```cpp
void ConstraintGraph::optimize() {
  // Merge equivalence classes until a fixed point is reached.
  while (contractEdges()) {}
}
```

`contractEdges`では、Constraintが`Bind`系もしくは`Equal`の場合にそのConstraintを取り除いた後、Constraintの2つの型を同値類として登録する。

```cpp
removeEdge(constraint);
if (rep1 != rep2)
  CS.mergeEquivalenceClasses(rep1, rep2, /*updateWorkList*/ false);
```

`mergeEquivalenceClasses`は名前の通り2つの型変数をマージする。

![5.png (69.7 kB)](https://img.esa.io/uploads/production/attachments/2245/2017/12/27/2884/8958397d-d86c-4115-8c06-84ee96c444f0.png)

## bindTypeVariable

`solution`を1つ受け取る方の`applySolution`で使われる。
(`applySolution`は`solveSimplify`の過程で使われるものと、TypeCheckerで最終的に使われるものの2つがあって、前者)

```cpp
void ConstraintSystem::applySolution(const Solution &solution) { ... }
```

ここで`ConstarintSystem::assignFixedType`が呼ばれ、さらにそこから`bindTypeVariable`が呼ばれる。

```cpp
assignFixedType(binding.first, binding.second, /*updateState=*/false);
```

```cpp
// Notify the constraint graph.
CG.bindTypeVariable(typeVar, type);
```

そうするとNodeの`FixedBinding`というフラグがたつ。
(どう使われる？)

## まとめ

だいたいどんなものかはわかったので、実際の使われ方を次回見てみる。
