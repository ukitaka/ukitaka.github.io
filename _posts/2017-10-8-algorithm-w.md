---
layout: post
title:  Algorithm W Step By Stepを読んだ & 実装した
---

Algorithm WをSwiftで実装してみた。

+ [GitHub - ukitaka/AlgorithmW.swift](https://github.com/ukitaka/AlgorithmW.swift)

少し前に作った [ukitaka/TypeSystem](https://github.com/ukitaka/TypeSystem) とほぼ同じだけど、前回のはlet多相を無視して作ったので今回はちゃんと実装。

## Algorithm W

+ [A theory of type polymorphism in programming - ScienceDirect](http://www.sciencedirect.com/science/article/pii/0022000078900144)
+ [Algorithm W Step by Step](http://catamorph.de/documents/AlgorithmW.pdf)


Algorithm Wはlet多相をもった型システムにおける型推論アルゴリズム。
TaPLの22章で紹介されているものと違うのは、明示的に型注釈をつけることを許していなくて、完全な型無しラムダ項に主要型を割り当てる。

## let多相の動作確認

```
let id = λ x . x in ((id id) (id 1))
```

こんなコード相当のテストケースで多相性を確認する。
要は`(id id)` の左の`id` は `(X -> X) -> (X -> X)` という型になるが、`(id 1)`の`id`は `Int -> Int` となって、最終的にこの式全体は `Int` になることを確認。

```swift
func testTypeInference3() {
    let env = TypeEnvironment()
    let x = TypeVariable("x")
    let termX = Term.variable(x)
    let id = TypeVariable("id")
    let termId = Term.variable(id)
    let one = Term.literal(Literal.integer(1))
    let term = Term.let(id, .abstraction(x, termX),
            Term.application(Term.application(termId, termId),  Term.application(termId, one)))
    let inferredType = Inference.typeInference(env: env, term: term)
    XCTAssertEqual(inferredType, .integer)
}
```

よさそう。

## 多相性を実現している部分

まず[この部分](https://github.com/ukitaka/AlgorithmW.swift/blob/f387c77a91f1042844e0f47d52b36feb7b3dbd18/Sources/AlgorithmW/Inference.swift#L40-L42)で `termBind` の型を `generalize` する。
つまり`∀` をつける。

```
id : X -> X
```

をこうして、型環境に突っ込んでおく。

```
id : ∀ X . X -> X 
```


その後 `id` が出現して型再構築を行うときに、[この部部分](https://github.com/ukitaka/AlgorithmW.swift/blob/f387c77a91f1042844e0f47d52b36feb7b3dbd18/Sources/AlgorithmW/Inference.swift#L18-L19) で、フレッシュな型変数に置き換える。`TypeScheme` の `instantiate` がまさにその実装

```swift
extension TypeScheme {
    func instantiate() -> Type {
        let freshTypeVariables = (0..<variables.count).map { _ in TypeVariable() }.map(Type.typeVar)
        let subst = Substitution(Dictionary(keys: variables, values: freshTypeVariables))
        return type.apply(subst)
    }
}
```

## おまけ

これを作っている途中にSwiftコミッターのCodaFi氏のAlgorithm.swiftも見つけた。Playgroundで遊ぶならこっちの方がよさそう。

+ [CodaFi/AlgorithmW.swift](https://gist.github.com/CodaFi/ca35a0c22fbd96eca505b5df45f2509e)

AlgorithmWに加えて、AlgorithmMという推論アルゴリズムを実装している。ちゃんと読んでないけど、AlgorithmWがtop-downなアプローチなのに対して、AlgorithmMがbottom-upなアプローチだとか書いてある。

+ [Generalizing Hindley-Milner Type Inference Algorithms](https://pdfs.semanticscholar.org/8983/233b3dff2c5b94efb31235f62bddc22dc899.pdf)

証明はこっち

+ [Proofs about a Folklore Let-Polymorphic Type Inference Algorithm](https://ropas.snu.ac.kr/~kwang/paper/98-toplas-leyi.pdf)


