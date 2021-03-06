---
layout: post
title: Rustのpubについて
---

## 外部とは？

`pub`キーワードがつけられたものは外部から参照できる。
この場合の「外部」とは「モジュール外」のことで、基本的にはモジュールが境界となる。
ただしもちろんクレートもモジュールを含んでいるので、外部クレートのことも指す。 

## pubキーワードについて

pubの規則は2つのみ。

> ・ If an item is public, then it can be accessed externally from some module `m`  if you can access all the item's parent modules from m. You can also potentially be able to name the item through re-exports. See below.
> 
> ・If an item is private, it may be accessed by the current module and its descendant

https://doc.rust-lang.org/stable/reference/visibility-and-privacy.html

ここでいう`item` には モジュールも含まれるみたい。

たとえば2つ目の規則に従って、`submodule1` は以下から見える。

```rust
mod module {
    mod submodule1 { }

    // module内では見える。
     
    mod submodule2 {
        // submodule2もmoduleの内なので見える
        // subsubmodule2_1もmodule内なので見える。
        mod subsubmodule2_1 { }
    }
}

// ここでは見えない。
```


## structとフィールドについて

`struct` が境界を作らない点にも注意。例えば`struct` のフィールドがprivateであっても、その`struct`が定義されたモジュール内からは見える。(swiftになれてるとここはちょっと違うポイント。)	

```rust
mod module {
    // Aはprivate
    struct A {
        a: i32 // aもprivate
    }

    mod submodule {
        mod subsubmodule {
            fn func() {
                // ここはmodule内なのでAもaも見える
                let a = super::super::A {a: 123};
            }
        }
    }
}
``` 

その`struct` のメソッドだからといって特別扱いはなく、`impl` を書いた場所によってprivateなフィールドにアクセスできるか決まる。

```rust
mod module {
    pub struct A {
        a: i32 // aはprivate
    }

    impl A {
        fn say_hello(&self) {
            println!("hello! {}", self.a); // OK: aは見える
        }
    }
}

//ここにimplを書いてもaは見えない
impl module::A {
   ...
}
```

## enumとデータについて
`pub`を`enum`につけた場合は各データもpublicになる。

```rust
pub enum Status {
    On,
    Off
}
```


## privateな関数のテストについて

rustはテストを近くに書くのが文化らしい。良い。
なので、privateなitemが見える位置にテストケースを書いてしまえば良い。

```rust
mod module {
    fn my_impl() { }

    #[cfg(test)]
    mod test {
        #[test]
        fn test_my_impl() {
        }
    }
}
```


## pubの範囲を指定する

pubにも範囲を指定することができる。

+ `pub(in パス)`: 指定したモジュールに公開
+ `pub(crate)` : 現在のクレート内
+ `pub(super)`: = `pub(in super)`
+ `pub(self)` : = `pub(in self)`


## pub use で公開するものしないものを整理する

サンプルより。

```rust
// ここは強大なので`implemetation`が可視
// 外には`implementation`を公開せず、`api`のみを公開。
pub use self::implementation::api;

mod implementation {
    pub mod api {
        pub fn f() {}
    }
}
```


## 参考

- [Visibility and Privacy - The Rust Reference](https://doc.rust-lang.org/stable/reference/visibility-and-privacy.html)
