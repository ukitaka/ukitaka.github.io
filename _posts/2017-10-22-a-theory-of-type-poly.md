---
layout: post
title:  A Theory of Type Polymorphism (Robin Milner 1978)を 実装した その1
---

[A Theory of Type Polymorphism](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.67.5276&rep=rep1&type=pdf) は前回読んだ[Algorithm W Step by Step](http://catamorph.de/documents/AlgorithmW.pdf) の元の、AlgorithmWについての論文。

一応元の論文も読んでおくかと思ってちょろっと眺めつつ実装してみたものの、実装するだけならStep by Stepの方だけで良かったかもしれない…

実装は[GitHub - ukitaka/TypeSystems.swift](https://github.com/ukitaka/TypeSystems.swift) にあります。

## Algorithm W

Step by Stepの方とは違い、`if` と不動点`fix` が構文に組み込まれている。

```swift
public indirect enum Exp {
    case `var`(VarName)
    case literal(Literal)
    case `if`(Exp, Exp, Exp)
    case abs(VarName, Exp)
    case app(Exp, Exp)
    case `let`(VarName, Exp, Exp)
    case fix(VarName, Exp)
}
```


Step by Stepの方でTypeEnvという名前で扱っていたもの(つまり、変数名と型の対応)が、(typed) prefixという形で扱われている。

```swift
public struct TypedPrefix {
    public enum TypedMember {
        case `let`(VarName, Type)
        case fix(VarName, Type)
        case abs(VarName, Type)
    }

    let members: [TypedMember]
}
```

単一化アルゴリズムは特に変わりない。論文中ではAlgorithm Uという名前で扱われていて、[J. A. Robinson: A Machine-Oriented Logic Based on the Resolution Principle](https://web.stanford.edu/class/linguist289/robinson65.pdf) という論文で初めて単一化が形式的に定義されたらしいということがわかった。

```swift
    /// Unification
    ///
    /// See: "J. A. Robinson: A Machine-Oriented Logic Based on the Resolution Principle"
    /// https://web.stanford.edu/class/linguist289/robinson65.pdf
    public static func mostGeneralUnifier(_ type1: Type, _ type2: Type) -> Substitution {
        switch (type1, type2) {
        case let (.typeVar(varName), _):
            return Substitution(varName: varName, type: type2)
        case let (_, .typeVar(varName)):
            return Substitution(varName: varName, type: type1)
        case let (.func(arg1, ret1), .func(arg2, ret2)):
            let s1 = mostGeneralUnifier(arg1, arg2)
            let s2 = mostGeneralUnifier(s1.apply(to: ret1), s1.apply(to: ret2))
            return s1 ∪ s2
        case (.int, .int):
            return Substitution()
        case (.bool, .bool):
            return Substitution()
        default:
            fatalError("error")
        }
    }

```

多相性の実現方法も前回見たとおり、`let` の場合は新しい型変数を割りあてることで実現している。

```swift
case let .var(x):
    if let member = p[x], p.isActive(member: .abs(x)) || p.isActive(member: .fix(x)) {
        return (Substitution.empty(), TypedExp.var(x, member.type))
    } else if let member = p[x], p.isActive(member: .let(x)) {
        return (Substitution.empty(), TypedExp.var(x, member.type.instantiate())) // ここ
     }
```


## 多相性の確認

```
let id = λx.x in ((id id) (id 1))
```

このプログラムを型推論すると`Int`になり、各部分は以下のようになることが確認できる。

```
let id:(X1 → X1) = (λx:X1. x:X1):(X1 → X1) in ((id:((Int → Int) → (Int → Int)) id:(Int → Int)):(Int → Int) (id:(Int → Int) 1:Int):Int):Int:Int
```


論文ではよりよい実装としてAlgorithmJが紹介されているのでその実装もしてみる。(つづく)
