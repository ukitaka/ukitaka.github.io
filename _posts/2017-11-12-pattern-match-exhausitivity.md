---
layout: post
title:  A generic algorithm for checking exhaustivity of pattern matching を読んだ
---

Swiftコンパイラのコードリーディングをしていてふと見つけたパターンマッチの網羅チェックに関する論文。

- [A generic algorithm for checking exhaustivity of pattern matching](https://infoscience.epfl.ch/record/225497)

論文ではScalaのDottyというコンパイラに導入してissueやパフォーマンスの改善に繋がったと紹介されていて、下のp-rによってSwiftにも採用されている。(swift4からかな？)

- [Redo Exhaustiveness Analysis by CodaFi · Pull Request #8908 · apple/swift · GitHub](https://github.com/apple/swift/pull/8908)

short paperなら二段組で4ページなのでさくっと読めた。

## 論文要点まとめ

落合先生のフォーマットを使ってみる。

### どんなもの？

パターンマッチの網羅性をチェックするために「Space」(≒ 型やパターンが取りうる値の集合)と呼ばれる概念を導入し、網羅性をsubspace関係 (≒ 集合の包含関係)によって形式化する。

### 先行研究に比べてどこがすごい？

特定の言語や型システムから切り離されており、一般的でシンプルなアルゴリズム。また特定の型システムに適用する場合にもコアのロジックはそのままにカスタマイズできる。

### 技術や手法のキモはどこ？

space, subspace relation(≾)の導入と「`パターンp1, p2, … が型Tを網羅している ⇔ 𝒯(T)  ≾  𝒫(p1) | 𝒫(p2) | …`」という形式化。

subspace relation を space subtraction(⊖) によって定義したところ？

### どうやって有効だと検証した？

フォーマルな証明はされていない。
Scalaのdottyに採用してissueが改善した。ソースコードが1/3程度になった。ほとんどのケースでパフォーマンスが改善した。

### 議論はある？

ある形のsubspace関係を扱うために、subspace関係をsubtraction(⊖) によって定義している。
またSwiftでいう、caseにwhereが付いている場合などはこのアルゴリズムでは扱うのは難しい。

### 次に読むべき論文は？

警告の表示方法についての論文がReferenceにもSwiftのコード中にも書いてあるのでこれを読むべきかも？

- [Warnings for pattern matching](http://moscova.inria.fr/~maranget/papers/warn/index.html)


## Design & Algorithm メモ

基本的なアイディアは型とパターン(case)を値のSpaceだと考える。

+ 型のSpaceは、その型を持つ値の集合で、`𝒯(T)` は型`T`のSpaceを表す。
+ パターンのSpaceは、そのパターンによってカバーされる値の集合で、`𝒫(p)` はパターン`p`のSpaceを表す。


例えば、Swiftっぽいコードで考えると

```
𝒯(Bool) = { true, false }
𝒯((Bool, Int)) = { (true, 0), (true, 1) .... }
𝒫(case .some(_)) = { .some(true), .some(false) } 
𝒫(case (true, _)) = { (true, false), (true, true) }
```

これにsubspace関係(≾) を導入する。

`s1 ≾ s2`  ⇔ `s1 ⊖ s2 = 0`

`0` は空Spaceで `⊖` はsubtraction。直観的には集合の差分。
細かい規則は論文中のFigure 1, 2を参照。

これを使って網羅性は以下のように形式化される。

`パターンp1, p2, … が型Tを網羅している ⇔ 𝒯(T)  ≾  𝒫(p1) | 𝒫(p2) | …` 

直観的にはパターンが網羅している値の和集合が、型Tに属する値の集合のスーパーセットになっている、みたいなイメージ。

subspace関係を調べることで網羅チェックが行える。

## まとめ
かなり省略してメモしたのであんまり参考にならないと思うし、Short Paperは4ページなので自分で読んだほうが良いかも。。。
次はSwiftのコンパイラの実装を読んでみるか、実装してみる。

