---
layout: post
title: SwiftでのWebサーバー実装 2017年9月
---

2017年9月時点で見つけた良さそうなものの雑多メモ。
どんなアーキテクチャかと、`Server::Starter` への対応というネタをどこかでやりたいのでGraceful shutdownに対応しているかという観点を見ている。

## Kitura

IBMが作っているのでちゃんとしてそうというイメージ。今から本当にSSSやるなら、自分だったらこれを選ぶかも。

+ [GitHub - IBM-Swift/Kitura: A Swift web framework and HTTP server.](https://github.com/IBM-Swift/Kitura)
+ [GitHub - IBM-Swift/Kitura-net: Kitura networking](https://github.com/IBM-Swift/Kitura-net)
    + こっちがServer部分
    + **オレオレepollを実装している**
    + 基本的にはGCDによるPrefork(マルチスレッド)モデルっぽい
+ [GitHub - IBM-Swift/BlueSignals: Generic Cross Platform Signal Handler](https://github.com/IBM-Swift/BlueSignals)
    + シグナルを扱うライブラリ
+ Graceful Shutdown  サポートしてそう？
    + **と見せかけてしてなかった。**シグナルは `SIGPIPE` のみのサポート。
    + でもBlueSignalsあるしすぐできそう。近い将来に期待。


## Curassow

`Nest` というWSGI / Rack / PSGI 的なインターフェースをサポートしたWebサーバー。
正直あまり名前は聞かない。

+ [GitHub - kylef/Curassow: Swift HTTP server using the pre-fork worker model](https://github.com/kylef/Curassow)
+ [GitHub - nestproject/Nest: Swift Web Server Gateway Interface](https://github.com/nestproject/Nest)
+ Prefork (マルチスレッド)モデル
+ **Graceful Shutdown** サポートあり
    + 素晴らしい
    + `Server::Starter` 試すならこれか？

## Vapor

いまSwiftで一番人気があるのはこれなんだろうか？理由はよくわからないけど、フレームワークが使いやすいとかかな？

+ [GitHub - vapor/engine: 🚀 Non-blocking networking for Swift (HTTP and WebSockets).](https://github.com/vapor/engine)
+ [GitHub - vapor/vapor: 💧 A server-side Swift web framework.](https://github.com/vapor/vapor)
+ Prefork (マルチスレッド)モデル
	+ Non-blockingとは言ってるのは、メインスレッドでacceptしたのをワーカー(バックグラウンドスレッド)に渡して処理しているというだけ
+ Graceful Shutdownサポートなさそう

## Perfect-HTTPServer

老舗。だいぶ初期からSSSのフレームワークとして有名だった気がする。

+ [GitHub - PerfectlySoft/Perfect-HTTPServer: HTTP server for Perfect.](https://github.com/PerfectlySoft/Perfect-HTTPServer)
+ Prefork(マルチスレッド)モデル
+  Graceful Shutdownサポートなさそう
 
## Skelton

日本人の方が作ってる。アーキテクチャがとてもちゃんとしているが、現時点でのSwiftの非同期周りの弱さと、言語としての限界もちょっと見えてしまった。

+ [GitHub - noppoMan/Skelton: An asynchronous http server for Swift](https://github.com/noppoMan/Skelton)
+ イベント駆動 ・ WorkerProcess
    +  node.jsと同じアーキテクチャ
    + 非同期I/O
    + libuv
    + [Swiftに適したサーバーアーキテクチャを再考して実装までしてみる // Speaker Deck](https://speakerdeck.com/noppoman/swiftnishi-sitasabaakitekutiyawozai-kao-siteshi-zhuang-madesitemiru)
+ Graceful Shutdownサポートなさそう

## swift-server/http
+ [GitHub - swift-server/http: Repository for the development of cross-platform HTTP APIs](https://github.com/swift-server/http)
+ あくまで実装サンプル
+ [GitHub - swift-server/work-group: Work group steering the development and direction of the Swift Server APIs](https://github.com/swift-server/work-group)
    + SSSが気になるならここの動きは追うほうがよさそう

