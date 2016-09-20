---
layout: post
title: RxSwiftでnilをフィルタリングする
---

自分で書いても良いんですが、一応RxSwift Communityが提供している `RxOptional`というのがあるのでメモしておきます。


## 動機

nilをフィルタリングしたい。その際型もオプショナルじゃなくしたい。
たとえば`Observable<String?>` があったとして、`nil`をフィルタリングしつつ`Observable<String>`にしたい。
要は`Observable<E?>` → `Observable<E>` の変換を行いたい。


愚直に`filter`でやると

```swift
let o = Observable<String?>.just(nil) 
o.filter { $0 != nil } // nilは弾けたけどこの型は `Observable<String?>のまま
o.filter { $0 != nil }.map { $0! } // うーん 
```

なのでやるなら

```swift
public protocol OptionalType {
    associatedtype Wrapped
    func map<U>(@noescape f: (Wrapped) throws -> U) rethrows -> U?
}

extension Optional: OptionalType { }

public extension ObservableType where E: OptionalType {
    public func filterNil() -> Observable<E.Wrapped> { ... }
}
```

みたいなのを自分で書く必要がある。

## RxOptional

[https://github.com/RxSwiftCommunity/RxOptional](https://github.com/RxSwiftCommunity/RxOptional)

やっていることは↑の実装と同じ。ただし便利なオペレータが他にも幾つか提供されている。


+ filterNil
+ replaceNilWith
+ errorOnNil
+ catchOnNil
+ distinctUntilChanged

また、以下のような`Occupiable` というプロトコルを実装した型については他のオペレーターも使える。

```swift
public protocol Occupiable {
    var isEmpty: Bool { get }
    var isNotEmpty: Bool { get }
}
```

+  filterEmpty
+ errorOnEmpty
+  catchOnEmpty


RxOptionalがデフォルトで提供している`Occupiable`の実装は以下。

```swift
extension String: Occupiable { }
extension Array: Occupiable { }
extension Dictionary: Occupiable { }
extension Set: Occupiable { }
```

使い方はREADMEに書いてある＆コードも大した量じゃないので読んだほうが早いかも :innocent: 


