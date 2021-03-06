---
layout: post
title: Rustのクレート / モジュールについてメモ
---


+ クレート / モジュール は他の言語で言うところのライブラリ / パッケージのようなイメージ。
+ `lib.rs` もしくは `main.rs` をルートとして、`mod` キーワードで辿っていけるもののみコンパイルされる。
+ ディレクトリでモジュールを定義した場合は `mod.rs` でまた `mod` キーワードを使ってたどり着けるもののみコンパイルされる。 
+ `use` でインポート/エクスポート

## modキーワードについて

`mod` キーワードはモジュールを定義するために使われる。
`mod` キーワードを使ってモジュールを定義する方法は3種類ある。

1.  `mod モジュール名 { … }` で定義する。
2. `モジュール名.rs` を作って `lib.rs` に `mod モジュール名` と書いて定義する。つまりファイルからモジュールを定義する。
3. `モジュール名/mod.rs` を作って、`lib.rs` に `mod モジュール名` と書いて定義する。つまりディレクトリからモジュールを定義する。

## modの2つの用法

`mod hoge;` と書いた場合と `mod hoge { ... }` と書いた場合は意味が違うという点に注意。

前者は`hoge.rs` もしくは `hoge/mod.rs`  からモジュールを定義するためのもの。
後者はその場所にインラインでモジュールを定義するためのもの。
同じ名前で併用するともちろんエラーになる。

```rust
mod hoge;

// error[E0428]: a module named `hoge` has already been defined in this module
mod hoge { }
```

## module.rs と module/mod.rs の両方があった場合

こういう構成だった場合、`mod module;` はエラーになる。 

```
src/lib.rs
src/module.rs
src/module/mod.rs
```

```
error: file for module `module` found at both hoge.rs and module/mod.rs
```


## パスについて
`x::y::z`のようなものを**パス**と呼んで、モジュールやモジュールに含まれるものを指すときに使う。

絶対パスと相対パスがある。ただし後述の `use` と一緒に使うかどうかで解釈が変わる。(こう書くと少しややこしく感じるが、直感的ではある)

### useで使う場合

+ `::x::y` のように`::` から始まるものは絶対パス
+ `x::y` のように `::` を省略して書けて、この場合も**絶対パス**。
+ `self::x` や `super::x` のように `self` か `super` で始まるものは相対パス。

### それ以外で使う場合

+ `::x::y`のように `::`から始まるものは絶対パス
+ `x::y` のように `::` を最初に書かなかった場合は**相対パス**。
+ `self`, `super` から始まるものは同様に相対パス。

```
let x = ::module_x::StructX { };
```

↑ 例えばこうとか。


## externについて

`extern crate クレート名` で外部のクレートをリンクしろ、ということをかける。

```rust
extern crate crate_a;
```

別名もつけられる。

```rust
extern crate crate_a as 別名;
```

## useについて

+ インポートとしての `use` 
+ `lib.rs` や `mod.rs` で使うエクスポートとしての `pub use`
	+ (実はそれ以外でも使える…)

ドキュメントには上記のようにインポート / エクスポートの2つの役割があるみたく書いているけれど、やっていることは一つで`use`でモジュールの中身をそこにどんと置くようなイメージ。`pub` をつければそれの様子が外部からも見える。

たとえば

```rust
mod mystd {
    pub use std::*;
}
```

と書けば

```rust
let a = mystd::net::Shutdown::Read;
```

のように使える。(例です。この用途なら `use ~ as` でよいのかな？)

上にもちょろっと書いたけど、

`module::*` のようにワイルドカードでかけたり、複数まとめて `module::{a,b,c}` のようにもかける。

## 参考

- [クレートとモジュール](http://rust-lang-ja.github.io/the-rust-programming-language-ja/1.6/book/crates-and-modules.html)
- [Paths](https://doc.rust-lang.org/stable/reference/visibility-and-privacy.html)
- [Rustのモジュールの使い方](http://keens.github.io/blog/2017/01/15/rustnomoju_runokirikata/)
