---
layout: post
title: SwiftでのMethod Swizzlingについて
---

## SwiftでのMethod Swizzlingについての基本事項

+ メソッドの実装を入れ替えられる
+ swiftで実装を入れ替えるには以下の二つの条件を満たす必要がある
    + `NSObject`のサブクラス、もしくは`@objc`属性のついたクラス
    + メソッドが`dynamic`であること

```
class MyClass : NSObject {
        dynamic func hoge() { print("hoge") }
            dynamic func fuga() { print("fuga") }
}
```

```
let myClass = MyClass()

// 入れ替え
let fromMethod = class_getInstanceMethod(MyClass.self, "hoge")
let toMethod = class_getInstanceMethod(MyClass.self, "fuga")
method_exchangeImplementations(fromMethod, toMethod)

myClass.hoge() // fuga
```

`struct`や`enum`にもメソッドが定義できますが、`NSObject`のサブクラスにも`dynamic`にもできないので対象外です。

## 疑問1. superを呼び出すとどうなる？ サブクラスの挙動はどうなる？

B isa A として `hoge`と`fuga`を入れ替える

```
class A : NSObject {
    dynamic func hoge() { println("A : hoge()"); }
    dynamic func fuga() { println("A : fuga()"); }
}

class B : A {
   override dynamic func hoge() {
       print("B : hoge() ");
       super.hoge()
   }

   override dynamic func fuga() {
       print("B : fuga() ")
       super.fuga()
   }
}

// A の メソッドを入れ替える
switchInstanceMethod(A.self, "hoge", A.self, "fuga")

// B の メソッドを入れ替える
switchInstanceMethod(B.self, "hoge", B.self, "fuga")

// 表示
let b = B()
b.hoge()
```

| B | A |   b.hoge() の 表示  |
|----|---|---|--------|
| - | - | B : hoge() A : hoge()        |
| 入れ替え | - | B : fuga() A : fuga() |
| - |入れ替え | B : hoge() A : fuga()  |
| 入れ替え |入れ替え | B : fuga() A : hoge() |


### 結論

各型ごとの入れ替え


## 疑問2. どのタイミングで入れ替えるべき？

[ishkawa/ISRefreshControl](https://github.com/ishkawa/ISRefreshControl/blob/master/ISRefreshControl/UITableView%2BISRefreshControl.m) での実装であるように`load`で入れ替えるのが定石だった。

```
+ (void)load
{
    @autoreleasepool {
        if (![UIRefreshControl class]) {
            ISSwizzleInstanceMethod([self class], @selector(initWithCoder:), @selector(_initWithCoder:));
        }
    }
}
```

`load`や`initialize`の挙動は[\[NSObject load\] と \[NSObject initialize\] の違い](http://akisute.com/2011/08/nsobject-load-nsobject-initialize.html)が参考になる。

しかし、`swift 1.2`から`load`の**オーバライドが禁止された**ため、[NSHisperのこの記事](http://nshipster.com/swift-objc-runtime/)では、以下の2パターンを推奨している。

+ `initialize`で`dispatch_once`で囲った中で入れ替える(NSObjectのサブクラスである必要あり)
+ `application(_:didFinishLaunchingWithOptions:)`で入れ替える


## 疑問3. 元に戻せる？

戻せます。

```
let a = A()
switchInstanceMethod(A.self, "hoge", A.self, "fuga")
a.hoge() // A : fuga()
switchInstanceMethod(A.self, "hoge", A.self, "fuga")
a.hoge() // A : hoge()
switchInstanceMethod(A.self, "hoge", A.self, "fuga")
a.hoge() // A : fuga()
```

が呼ぶたびに挙動が変わるのはちょっと怖いので普通は実行時に何度も切り替えることはしないと思います。


## 疑問4. 入れ替わっているか判定できる？

ぱっと見た感じ少なくともそういったAPIはなさそうです。自分で管理するしか？

## 疑問5. 入れ替え先がなかった場合は？いつ落ちる？

入れ替え先のセレクタがなくても元の挙動になるだけで落ちなかったです。

```
let a = A()
switchInstanceMethod(A.self, "hoge", A.self, "piyo")
a.hoge() // A : hoge()
```

## 疑問6. 別のクラスと入れ替えたりもできる？

可能でした。

```
class A : NSObject {
    dynamic func hoge() { println("A : hoge()"); }
}

class C : NSObject {
    dynamic func hoge() { println("C : hoge()"); }
}

switchInstanceMethod(A.self, "hoge", C.self, "hoge")
let a = A()
let c = C()

a.hoge() // C : hoge()
c.hoge() // A : hoge()
```

## 疑問7. クラスメソッドとインスタンスメソッドを入れ替えたりもできる？

可能でした。

```
class A : NSObject {
    dynamic func hoge() { println("A : hoge()"); }
    dynamic func fuga() { println("A : fuga()"); }
    dynamic class func piyo() { println("A : piyo()"); }
}

let fromMethod = class_getInstanceMethod(A.self, "hoge")
let toMethod   = class_getClassMethod(A.self, "piyo")
method_exchangeImplementations(fromMethod, toMethod)

let a = A()
a.hoge() // A : piyo()
```

## 疑問8. 引数の数や型が違うメソッドは入れ替えられるの？

入れ替え自体では落ちないが普通の呼び出しでは基本的には落ちる。

```
class A : NSObject {
    
    dynamic func hoge() { println("A : hoge()"); }
    
    dynamic func fuga(comment: String) {
        println("A : fuga(\(comment))");
    }
    
    dynamic func fugaReturnString(comment: String) -> String {
        println("A : fugaReturnString(\(comment)) -> \(comment)");
        return comment
    }
    
    dynamic func piyo(num: Int) {
        println("A : piyo(\(num))");
    }
    
}
```

+ 引数の数が違う場合

```
switchInstanceMethod(A.self, "hoge", A.self, "fuga:")
// a.hoge() は落ちる
```

+ 返り値の有無が違う場合
	+ 入れ替えが行えるが、なにも返ってこない。	

```
switchInstanceMethod(A.self, "fugaReturnString:", A.self, "fuga:")
let ret = a.fuga("これはコメント") // A : fugaReturnString(これはコメント) -> これはコメント
println(ret) // ()
```

+ 引数の型が違う場合
	
	+ 入れ替えられるが、(おそらくcastできない場合) 落ちる

```
switchInstanceMethod(A.self, "fuga:", A.self, "piyo:")
a.fuga("comment") // A : piyo(140579043279440)
//a.piyo(123)     これは落ちる
```
	
## その他参考記事

[What are the Dangers of Method Swizzling in Objective C?](http://stackoverflow.com/questions/5339276/what-are-the-dangers-of-method-swizzling-in-objective-c)
