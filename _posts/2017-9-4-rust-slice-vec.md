---
layout: post
title: RustのスライスとVecと型強制
---



[前回](https://blog.waft.me/2017/09/04/rust-slice-array/)の続き。

`Vec` についても `&[T; n]` -> `&[T]` の型強制と同じように `&Vec<T>` -> `&[T]` の型強制が使える。

```rust 
let v = vec![1, 2, 3];
let slice_v: &[i32] = &v;
```

前回とは違い、`Vec` に `Unsize` は実装されていないので、前回のルールは適用できない。
この挙動について、どう動いているのかを確認する。

## Derefトレイトと型強制

`Deref` は名前の通りデリファレンス時の挙動を設定するためのトレイト。
また`Deref` に関する型強制のルールがある。

>  Deref coercion: Expression &x of type &T to &*x of type &U if T derefs to U (i.e. T: Deref<Target=U>)
> 
> https://doc.rust-lang.org/nomicon/coercions.html

つまり `T: Deref<Target=U>` が設定されていれば `&T` を `&U` に型強制できる。

## VecのDerefの実装

`Vec` は上記の `Deref`トレイトを、
`Target` を `[T]` (つまりスライス) として実装している。

```rust
impl<T> ops::Deref for Vec<T> {
    type Target = [T];

    fn deref(&self) -> &[T] { ... }
}
```


よって上のルールにより、 `&Vec<T>` を `&[T]` に型強制できることがわかった。

