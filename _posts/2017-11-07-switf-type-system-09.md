---
layout: post
title:  Swiftの型システムを読む その9 - unittestを使って動作確認する
---

ここまで主にswiftフロントエンドのデバッグオプションとコードリーディングだけで進めてきたが、やはりコードを動かしてみたい時がある。

今回は`swiftSema` の単体テストを動かせる環境を作って、直接内部のコードを触る準備をする。

## Swiftコンパイラのテスト環境について
rintaro先生のこちらの記事がわかりやすい。 [Swiftコンパイラのテスト環境 - Qiita](https://qiita.com/rintaro/items/2f84776cf1629150b312)

基本的には

1. `unittest/Sema` を作成＆設定して、Semaの単体テストを書いていく。
2. `utils/build-script -Rt` でビルドをする
3. `../build/Ninja-ReleaseAssert/swift-macosx-x86_64/unittests/Sema/SwiftSemaTests` のようなテスト実行用のバイナリができるのでそれを実行する

という流れのよう。実際は2の時点で単体テストも実行されているのだが、全部のテストが実行されてしまって見辛いので、あえて3で確認している。
`lit` とか `cmake` を使うともうちょっと簡単にかけるのかもしれないけれど、あまり調べていない。

## 単体テストの追加
Semaの単体テストはないので、自分で追加してみる。

まずはディレクトリを作成する。

```
unittests
├── AST
├── Basic
├── CMakeLists.txt
├── Driver
├── IDE
├── Parse
├── Reflection
├── Sema // 追加
├── SourceKit
├── SwiftDemangle
├── Syntax
└── runtime
```

`unittests/CMakeList.txt` を編集して`Sema` を追加する。

```
include(AddSwiftUnittests)

if(SWIFT_INCLUDE_TOOLS)
  # We can't link C++ unit tests unless we build the tools.

  add_subdirectory(AST)
  add_subdirectory(Basic)
  add_subdirectory(Driver)
  add_subdirectory(IDE)
  add_subdirectory(Parse)
  add_subdirectory(SwiftDemangle)
  add_subdirectory(Syntax)
  add_subdirectory(Sema) // 追加

  if(SWIFT_BUILD_SDK_OVERLAY)
    # Runtime tests depend on symbols in StdlibUnittest.
    #
    # FIXME: cross-compile runtime unittests.
    add_subdirectory(runtime)
    add_subdirectory(Reflection)
  endif()

  if(SWIFT_BUILD_SOURCEKIT)
    add_subdirectory(SourceKit)
  endif()
endif()
```

`unittest/Sema/CMakeLists.txt` を以下のように設定する。
`include_directories` でインクルードパスを追加しているのは、`lib/Sema` 以下にヘッダーファイルがいくつかおいてあるため。

```
add_swift_unittest(SwiftSemaTests
  SubTypeTests.cpp
)

include_directories(BEFORE
  ${SWIFT_SOURCE_DIR}/lib/Sema
)

target_link_libraries(SwiftSemaTests
   swiftAST
   swiftParse
   swiftSema
)
```


`unittest/Sema/SubTypeTests.cpp` をこんな感じで作ってみる。

```cpp
#include "swift/AST/ASTContext.h"
#include "swift/AST/Decl.h"
#include "swift/AST/DeclContext.h"
#include "swift/AST/Types.h"
#include "TypeChecker.h" // ちゃんと使える
#include "gtest/gtest.h"

using namespace swift;

TEST(Sema, Sample) {
  EXPECT_TRUE(true);
}
```


これで準備はOK。

## ビルド＆テスト実行

```
$ ./utils/build-script -Rt
```

`-t` をつけないとテスト用のターゲットのビルド自体が行われなかったので、`-t` をつける。


この状態で実行すると、ビルドディレクトリに単体テスト用のバイナリができている。
差分ビルドが効けば手元のMacBook Pro(3.1GHzデュアルコアIntel Core i5)でも30秒かからずにビルドが終わる。

```
$ ls -la build/Ninja-ReleaseAssert/swift-macosx-x86_64/unittests/Sema/
total 41352
drwxr-xr-x   5 ukitaka  staff       160 11  7 00:04 ./
drwxr-xr-x  15 ukitaka  staff       480 11  6 23:44 ../
drwxr-xr-x   3 ukitaka  staff        96 11  6 22:13 CMakeFiles/
-rwxr-xr-x   1 ukitaka  staff  21170064 11  7 00:04 SwiftSemaTests*
-rw-r--r--   1 ukitaka  staff       957 11  6 22:13 cmake_install.cmake
```

実行！

```
$ build/Ninja-ReleaseAssert/swift-macosx-x86_64/unittests/Sema/SwiftSemaTests
```

```
[==========] Running 1 test from 1 test case.
[----------] Global test environment set-up.
[----------] 1 test from Sema
[ RUN      ] Sema.Sample
[       OK ] Sema.Sample (0 ms)
[----------] 1 test from Sema (0 ms total)

[----------] Global test environment tear-down
[==========] 1 test from 1 test case ran. (1 ms total)
[  PASSED  ] 1 test.
```

良さそう。

## 追記: 単体テストのみビルドし直す方法

出力をみてたら見つけたので追記。

```
$ cmake --build ../build/Ninja-ReleaseAssert/swift-macosx-x86_64 -- -j8 SwiftUnitTests
```


## まとめ

とりあえず動かす準備ができたので、次以降でいくつかの関数を触ってみる。
