---
layout: post
title: Rustのスライスと配列と型強制
---

Rust では `[i32; 3]` は配列を表し、`[i32]` はスライスを表す。
シグネチャが違うことからわかるように、配列とスライスは全く別の型であることに注意。

## スライスに関する疑問
改めて、配列とスライスは別の型であることに注意する。

当然 `[i32]` が期待されているところで `[i32; 3]` を使おうとしても型チェックに引っかかる。

```rust
let a: [i32; 3] = [1, 2, 3];

// error[E0308]: mismatched types
let slice_a: [i32] = a;
```

にもかかわらず参照を取ると、エラーにならない。。

```
let a: [i32; 3] = [1, 2, 3];

// OK <- !?
let slice_a: &[i32] = &a;
```

なぜだろう…？？？ というのが今回のモチベーション。

## 型強制
> Rustの型強制は、式の型と必要な型が異なるときに、自動的に変換を挟むものである。
> http://qnighy.hatenablog.com/entry/2017/06/05/070000

Scalaでいうところのimplicit conversionのようなものみたい。

Rustには[4つの型強制ルール](https://doc.rust-lang.org/nomicon/coercions.html)があって、それに従う。`Deref` を利用して必要に応じて自前で型強制を定義できるっぽい。

そして、参照の場合( `&[i32; 3]` -> `&[i32]` ) が動作したのは型強制のおかげらしい。

この挙動を実現してる `Unsize` トレイトとその型強制についての実装をみてみる。

##  `Unsize` トレイトの実装

+ `Unsize`は、サイズがダイナミックに決まる型を表すトレイト。
+ 例えば `[i32; 3]` は静的にサイズが決まるが、それを動的に決まる型として扱いたいときのために `Unsize<[i32]>` を実装している。
+ このようにすべての`[T; N]` について、`Unsize<[T]>` がコンパイラによって自動的に実装されている。
+ 自動的にしか実装できない。

## `Unsize` トレイトによる型強制

[4つの型強制ルール](https://doc.rust-lang.org/nomicon/coercions.html)に `Unsize` トレイトに関するルールがある。


+ `&T,` `*mut T`, `Box`, `Rc`  などを**ポインタ型**と呼ぶ。
+ ポインタ型 `Ptr` について、 `T: Unsize<U>` が実装されていれば `Ptr<T>` は `Ptr<U>`に型強制される。


## 今回のケースについて

最初の例に戻って具体的にどう適用されたかを丁寧に書いてみる。

+ `[i32; 3]` について、コンパイラが自動で `Unsize<[i32]>` の実装を用意する。
+ 参照 `&[i32; 3]` について、`Unsize<[i32]>` が実装されているので、`&[i32; 3]` は `&[i32]` に型強制される。


## 参考
- [What is the difference between Slice and Array?](https://stackoverflow.com/questions/30794235/what-is-the-difference-between-slice-and-array)
- [Rust reference - Coercions](https://doc.rust-lang.org/nomicon/coercions.html)
