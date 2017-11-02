---
layout: post
title:  IntelliJでRustを書いているときにモジュールが更新された場合でも補完が効くようにする
---

小ネタメモ。

どうやら`cargo metadata` というコマンドを使ってインデックス情報を作っているらしいので、これを叩いてあげればOKっぽい。

```
cargo metadata --verbose --format-version 1 --all-features
```


