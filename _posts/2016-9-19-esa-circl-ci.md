---
layout: post
title: Jekyll Nowにesa.ioからPOSTする
---

## esa.io → github

esaはGithubのWebhookに対応していて、特定カテゴリ以下の記事をgithubにpushするみたいなことができます。

[参考- GitHub Webhook (β) で GitHubのリポジトリに md ファイルを push できるようになりました](https://docs.esa.io/posts/176)


しかし、pushされるmdの形式が以下のようで、Jekyllの形式とあっていません。

```markdown:35.html.md
---
title: "2016-09-19-esa.md"
category:
tags:
created_at: 2016-09-18 23:40:08 +0900
updated_at: 2016-09-18 23:40:08 +0900
published: true
number: 36
--- 

/*  ↑ はesaが勝手に追加した部分 */
/*  ↓ はesaは自分で書いた部分 */

---
layout: post
title: Jekyll Nowにesa.ioからPOSTする
---

本文本文本文本文本文本文本文本文
```

## esa.io → github → CircleCI → github → blog

+ esaからPOSTしたら、`_draft` ディレクトリ以下に置く
+ CircleCIを使ってpushされたときに `_draft` 以下の`.md`を変換して `_posts` ディレクトリに移す
+ CircleCIからpushする 

みたいにしました。
細かい設定は `circle.yml` や `_script/build.rb`などを参考にしてください。

+ [circle.yml](https://github.com/ukitaka/ukitaka.github.io/blob/master/circle.yml)
+ [build.rb](https://github.com/ukitaka/ukitaka.github.io/blob/master/_scripts/build.rb)

(めっちゃ雑なコードですが)

## 小ネタ

CircleCIのビルドをスキップするにはコミットメッセージに `[skip ci]` とか `[ci skip]` とか書くといいっぽいです。

[https://circleci.com/docs/skip-a-build/](https://circleci.com/docs/skip-a-build/)

いまの設定だと設定ファイルの変更をしただけとかでもCI動いてしまうので、必要ないときはスキップするようにしてます。


あと、esaと違って自動リンクにはしてくれないのでちゃんと`[リンク名](URL)`の形式でリンクを書くようにしましょう。
