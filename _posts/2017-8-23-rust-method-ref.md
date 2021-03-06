---
layout: post
title: Rustのメソッド構文と参照について
---

論理学を勉強していると思ったらRustに入門していた、何を言っているのか(ry

参照とメソッド呼び出しを組み合わせた時にどう動いているのかなんかよくわからなかったので調べたメモ。

## 所有型、参照型、可変参照型

所有型は正式な言葉じゃないですが、参照型以外をここではそう呼ぶことにします。

```rust
struct A { };

let a = A { }; // 所有型
let ref_a = &A { }; // 参照型
let ref_mut_a = &mut A { }; // 可変参照型
```

それぞれ `A` という型に関係していますが、**別の型です**。
間違えて使おうとすると型チェックに引っかかり、 `mismatched types` というエラーになります。

```rust
fn f_a_own(a: A) { println!("f_a_own") }

fn f_a_ref(a: &A) { println!("f_a_ref") }

fn f_a_mut_ref(a: &mut A) { println!("f_a_mut_ref") }
```


```rust
f_a_own(A { }); // OK
f_a_ref(A { }); // NG
f_a_mut_ref(A { }); // NG
```


```rust
f_a_own(&A { }); // NG
f_a_ref(&A { }); // OK
f_a_mut_ref(&A { }); // NG
```


~~ただし1つだけサブタイプ関係があって、 `&mut A is-a &A` のようです。~~
(追記: 正確には違いました。後述の`Deref`の挙動により `&mut` は `&` として振舞えるようです。)

```rust
f_a_own(&mut A { }); // NG
f_a_ref(&mut A { }); // OK <- !!!
f_a_mut_ref(&mut A { }); // OK
``` 


## 参照とメソッド呼び出しの疑問

それぞれ別の型であることがわかったのですが、そうするとメソッド呼び出しについて少し疑問が出てきます。

struct `A`がメソッド `method1` を持っているとして、以下のように呼び出せます。

```rust
let a = A { };
a.method1();
```


ところが、別の型であるはずの参照型、可変参照型についても`method1` が呼び出せることがわかります。

```rust
(&A{ }).method1();
(&mut A{}).method1();
```

なんなら「参照の参照の……」でも`method1`は呼び出せます。

```rust
(&a).method1();
(&&a).method1();
(&&&a).method1();
(&&&&a).method1();
(&&&&&a).method1();
```


## どういう仕組みで動いているか

[Method lookup](https://github.com/rust-lang/rust/blob/1.19.0/src/librustc_typeck/check/method/README.md) に解説があるので簡単に引用します。
メソッド呼び出し構文

```rust
receiver.method(...)
```

は、以下と同じです。

```rust
ReceiverType::method(ADJ(receiver), ...)
```

メソッドの第一引数に自分自身を渡すのですが、その際に `ADJ` (ADJUSTMENTの略) でなにか処理を加えたあとに引数に渡していることがわかります。

この `ADJ` はなにをしているかというと、ドキュメントにも書いてありますが、デリファレンスをします。

**メソッドを呼び出す時、自動ででリファレンスされる** と考えると良いです。
ソースコードの該当箇所は[この辺](https://github.com/rust-lang/rust/blob/master/src/librustc_typeck/check/method/confirm.rs#L143)です。

古いですが、日本語ドキュメントのDerefのところでも同様の記述が確認できました。

> Deref はメソッド呼び出し時にも自動的に呼びだされます。
> http://rust-lang-ja.github.io/the-rust-programming-language-ja/1.6/book/deref-coercions.html

## Derefについて

`Deref`はデリファレンス( `*` )の挙動を設定するためのトレイトで、自分で設定することもできますが、参照型と可変参照型については以下のようになっています。([該当コード](https://github.com/rust-lang/rust/blob/master/src/libcore/ops/deref.rs#L72-L94))

```rust
#[stable(feature = "rust1", since = "1.0.0")]
impl<'a, T: ?Sized> Deref for &'a T {
    type Target = T;

    fn deref(&self) -> &T { *self }
}

#[stable(feature = "rust1", since = "1.0.0")]
impl<'a, T: ?Sized> Deref for &'a mut T {
    type Target = T;

    fn deref(&self) -> &T { *self }
}
```


要は `*self` をしていているので、もしさらに参照があるなら再帰的に `deref` が呼ばれて最終的に所有型にたどり着きます。


## まとめ
所有型と参照型は別の型だけど、メソッド呼び出し時に再帰的に参照が外されるので、参照型でも所有型のメソッドが利用できる。


