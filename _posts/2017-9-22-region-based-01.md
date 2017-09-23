---
layout: post
title:  Region-Based Memory Management in Cyclone を読んだ その1
---

[Region-Based Memory Management in Cyclone](https://www.cs.umd.edu/projects/cyclone/papers/cyclone-regions.pdf) を読んだのでそのメモ。

CycloneはRustの前身となった言語で、C言語をベースにリージョンと呼ばれる考え方を用いたメモリ管理を採用した言語。

> Cyclone is no longer supported; the core research project has finished and the developers have moved on  to other things. (Several of Cyclone's ideas have made their way into Rust.) 

[Cyclone公式](https://cyclone.thelanguage.org/)にこうあるように、後続のRustへ「ライフタイム」としてのアイディアが引き継がれている。[Region-Based Memory Management in Cyclone](https://www.cs.umd.edu/projects/cyclone/papers/cyclone-regions.pdf) は、Rustの参考論文ページにも貼ってある論文。

このメモでは

* Cycloneでのメモリ管理について学ぶ
* Rustにどのように応用されたか、Cycloneとの比較をする

(翻訳ではなくて自分の理解のまとめなので、ここに書いて有ることが論文に書いて有ることというわけではないので注意)

## Cycloneが目指したところ
一言で言うと **Dangling pointerのデリファレンスをコンパイル時に検出してエラーにする**ということを目指した。

Dangling pointerとは、ポインタの指す先のリソースが解放済み状態のポインタのこと。CあるいはC++のスマートポインタを使ってもDangling pointerが起こり、もちろんコンパイル時に検出はできず実行時のエラーになる。

そこで**「リージョン(region)」**という概念を型システムとしてC言語を拡張する形で導入し、**不正なポインタへのアクセスをコンパイラが検出できる**ようにしつつ、デフォルトのリージョンの決定方法あるいは**リージョン推論**によってプログラマによる明示的なリージョンを記述が最小限になるようにしている。

それまでの研究されていたリージョンによるメモリ	1管理に加えて、リージョンサブタイピング、Effect、ローカルリージョン推論、存在型との組み合わせあたりを実践しているらしい。

もちろん大前提としてメモリリークが起きないようなデザインにもなっている。

## Cycloneがやらなかったこと

Cycloneではリージョンによるメモリ管理を導入しつつも、**ヒープリージョンのメモリ管理にはGCを採用している**。このあたりはRustがいかにしてGCなしで実現しているかを別途解説したい。

## リージョン(region)とは？

Cycloneではメモリはリージョンと呼ばれる領域にわけられて利用される。すべてのオブジェクトはどこかのリージョンに属し、リージョンが解放されるときにそのリージョンに属するオブジェクトがまとめて解放される。

リージョンにはいくつか種類があるが、論文で説明されているリージョンと[Cycloneのドキュメント](https://cyclone.thelanguage.org/wiki/Introduction%20to%20Regions/)に書いてあるリージョンで微妙に名前が違って、例えば論文の「Dynamic region」はドキュメントでは「Lexical region」にあたる。ややこしいことにドキュメントでは「Dynamic region」は別の意味で使われている。

ここでは論文に名前を合わせる。

+ スタックリージョン(Stack region)
+ 動的リージョン(Dynamic region)
+ ヒープリージョン(Heap region)


ヒープリージョンは名前の通りメモリのヒープ領域の特別な部分を指すリージョンで、staticなデータが置かれたりや`malloc`  / `new` のアロケート先だったりする。Cycloneには `free` の仕組みがなく、ヒープリージョンに置かれたデータはGCによって解放される。

スタックリージョンはまさにいわゆる「スタック」に結びついたリージョンで、たとえば関数が呼びさされれば引数や関数内ローカル変数がそのリージョンにアロケートされ、呼び出しが終わればリージョンが解放される。サイズがコンパイル時に決まり、メモリのスタック上にアロケートされる。

動的リージョンは、`region r { ... }`  によって導入されるリージョンで、**コンパイル時にサイズが決まっていない**。実際にはヒープ上にアロケートされる。リージョン名`r` と`rnew` / `rmalloc` のようなリージョンを渡せるAPIを使ってメモリを動的にアロケートしたりできる。 `malloc` と同じ感覚でアロケートでき、さらにブロックを抜ければリージョンが解放されるのでリークも起きない。

動的リージョンはLinked listで実装され、足りなくなった場合は次のページを前のページの倍のサイズで作成する。リージョンハンドルは最後のページのアロケーションポイントへのポインタ。リージョンを解放するときは各ページを解放する。

(つづく)
