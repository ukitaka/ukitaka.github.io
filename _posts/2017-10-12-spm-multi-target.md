---
layout: post
title:  Swift Package Managerでマルチターゲットな構成を作る
---


※ 雑メモ注意です。公式ドキュメントをちゃんと読んで下さい。

+ Swift Package Managerのいいところ
    +  `swift build` / `swift test` だけでビルド・テストが走らせられる
    + メンドクサイ `.xcodeproj` の設定が要らない
+ ライブラリとして公開はしないけど、本や論文を実装してみたものとしてコードを公開してみることがある
+ ある程度コードを整理して使いまわしたいが、Swiftだと気軽にパッケージ切ったりできない…
    +  Swift以外を使うという手もあるが、ギョームでSwiftを書かなくなったのでSwift忘れないように書いている
+ SPMならディレクトリを分けてちょっとだけ設定すればOK！！


こんな流れでSPMでマルチターゲットな構成を作ってみた。
(Swiftは4.0)

## やりたいこと

```
Package.swift
Sources/ModuleA/A.swift
Sources/ModuleB/B.swift
Sources/Utils/Utils.swift
Tests/ModuleATests/ATests.swift
Tests/ModuleBTests/BTests.swift
Tests/UtilsTests/UtilsTests.swift
```

こんな構成にして

+ モジュールAとモジュールBの両方からUtilモジュールをimportして使いたい
+ それぞれにテストを書きたい

## やりかた

構成は上の通りで、`Sources`と`Tests` にディレクトリを切ってコードを整理する。そして `Sources` においたものは `.target` で、`Tests` においたものは`.testTarget` で `tagets` に設定する。

```swift
// swift-tools-version:4.0

import PackageDescription

let package = Package(
    name: "MyModules",
    products: [
      .library(name: "ModuleA", targets: ["ModuleA"]),
		.library(name: "ModuleB", targets: ["ModuleB"]),
      .library(name: "Utils", targets: ["Utils"])
    ],
    targets: [
      .target(name: "ModuleA", dependencies: ["Utils"]),
		.target(name: "ModuleB", dependencies: ["Utils"]),
      .target(name: "Utils", dependencies: []),
      .testTarget(name: "ModuleATests", dependencies: ["ModuleA"]),
      .testTarget(name: "ModuleBTests", dependencies: ["ModuleB"]),
      .testTarget(name: "UtilsTests", dependencies: ["Utils"]),
    ]
)
```


これでOK。ビルドしてみる。

```
$ swift build
```

テストも簡単。`.travis.yml` もこれだけでOK。

```
$ swift test
```

