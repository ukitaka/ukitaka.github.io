---
layout: post
title: xcjobs-testfairyというgemを作った
---

旧TestFlightが使えなくなって少したちましたが、皆どこに乗り換えているのか気になるところです。
iOS8以降対応のアプリなら新TestFlightを素直に利用すればいいのでしょうが、iOS7も対応しているアプリを作るとなるとそうも行かないのではないでしょうか。


個人的にはボタンひとつで移行できた[TestFairy](http://testfairy.com/)をとりあえず使っている＆岸川先生の[xcjobs](https://github.com/kishikawakatsumi/xcjobs)を使ってビルドスクリプト書いているので
TestFairyへのアップロードもxcjobsで書きたい！と思ったのでgemを作ってみました。

+ [xcjobs-testfairy](https://github.com/ukitaka/xcjobs-testfairy)

本家のTestFlightの部分を参考にしながら作っただけなので、そんなすごいものではないですがよかったらお使いください。



あと、[fastlane](http://qiita.com/appwatcher/items/a3280ecdef7e4d9e5e24)も気になるところです。
