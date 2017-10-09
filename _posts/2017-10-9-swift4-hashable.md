---
layout: post
title:  Swift4.0でのTypeCheckerのバグを見つけたけどもう修正されてたメモ
---

Swiftの`Set`の型パラメータは `Element: Hashable` の制約が付いている。

```swift
struct Set<Element where Element : Hashable>
```


当然`Hashable` でないものを入れれば型検査のときに怒られるはずだけど、Swift4で怒られなくなってるものがあった。

```swift
struct NotHashable { }

Set([NotHashable(), NotHashable(), NotHashable()])
```

Swift3までは型検査に引っかかる。

```
hashable.swift:3:1: error: generic parameter 'Element' could not be inferred
Set([NotHashable(), NotHashable(), NotHashable()])
^
Swift.Set:139:15: note: 'Element' declared as parameter to type 'Set'
public struct Set<Element> : SetAlgebra, Hashable, Collection, ExpressibleByArrayLiteral where Element : Hashable {
              ^
hashable.swift:3:1: note: explicitly specify the generic arguments to fix this issue
Set([NotHashable(), NotHashable(), NotHashable()])
^
   <<#Element: Hashable#>>
```


Swift4では型検査通ってSILを吐くときにセグフォで落ちる。

```
% swiftc hashable.swift
0  swift                    0x0000000106263dba PrintStackTraceSignalHandler(void*) + 42
1  swift                    0x00000001062631f6 SignalHandler(int) + 662
2  libsystem_platform.dylib 0x00007fff781fbf5a _sigtramp + 26
3  libsystem_platform.dylib 0x0000000000000018 _sigtramp + 2279620824
4  swift                    0x000000010343ce0b swift::ASTVisitor<swift::Lowering::SILGenModule, void, void, void, void, void, void>::visit(swift::Decl*) + 427
5  swift                    0x000000010343bf6b swift::Lowering::SILGenModule::emitSourceFile(swift::SourceFile*, unsigned int) + 1115
6  swift                    0x000000010343d8f9 swift::SILModule::constructSIL(swift::ModuleDecl*, swift::SILOptions&, swift::FileUnit*, llvm::Optional<unsigned int>, bool) + 841
7  swift                    0x0000000102bd62c6 performCompile(swift::CompilerInstance&, swift::CompilerInvocation&, llvm::ArrayRef<char const*>, int&, swift::FrontendObserver*, swift::UnifiedStatsReporter*) + 13014
8  swift                    0x0000000102bd1784 swift::performFrontend(llvm::ArrayRef<char const*>, char const*, void*, swift::FrontendObserver*) + 7716
9  swift                    0x0000000102b866a8 main + 12248
10 libdyld.dylib            0x00007fff77f7b145 start + 1
11 libdyld.dylib            0x000000000000000f start + 2282245835
Stack dump:
0.	Program arguments: /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/swift -frontend -c -primary-file hashable.swift -target x86_64-apple-macosx10.9 -enable-objc-interop -sdk /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.13.sdk -color-diagnostics -module-name hashable -o /var/folders/1t/8tgg03jj07d5kng2cy_krf6w0000gn/T/hashable-307082.o
<unknown>:0: error: unable to execute command: Segmentation fault: 11
<unknown>:0: error: compile command failed due to signal 11 (use -v to see invocation)
```


案の定たくさん報告されてたw

+ [SR-5932 Segmentation fault: 11 when creating a Set of optionals - Swift](https://bugs.swift.org/browse/SR-5932)
+ [SR-5836 Compilation crash in Dictionary extension - Swift](https://bugs.swift.org/browse/SR-5836)
+ [SR-6058 Segmentation fault: 11 when emitting SIL for custom collection function - Swift](https://bugs.swift.org/browse/SR-6058)
+ [SR-5934 Segmentation fault: 11 when creating a Set from an array of arrays - Swift](https://bugs.swift.org/browse/SR-5934)

ちなみに`Set<NotHashable>` と明示的に書くと型検査でエラーになるし、メッセージも期待通り。
各Elementから型推論する場合のみ起こるみたい。

```
ashable.swift:3:1: error: type 'NotHashable' does not conform to protocol 'Hashable'
```


今日(2017/10/7)時点ですでに修正済みになってたのでどんな修正がされたのか見てみる。


## 修正内容を見る

プルリクとしてはこの2つ

+ [Sema: Fix a failure to emit a diagnostic by slavapestov · Pull Request #12149 · apple/swift · GitHub](https://github.com/apple/swift/pull/12149)
+ [Add another test case for SR-5932 by slavapestov · Pull Request #12194 · apple/swift · GitHub](https://github.com/apple/swift/pull/12194)

メインの変更は`TypeChecker::diagnoseArgumentGenericRequirements` の関数のreturnの部分。

```cpp
-  return result != RequirementCheckResult::Success;
+  return result == RequirementCheckResult::Failure;
```

`RequirementCheckResult` は4つの列挙子があって、今回は `SubstitutionFailure` の場合が失敗として扱われてしまっていたことが問題みたい。(そもそも4パターンあるのにtrue/falseにするから…)

```cpp
enum class RequirementCheckResult {
  Success, Failure, UnsatisfiedDependency, SubstitutionFailure
};
```



## どう直ったか

`onstraintSystem::diagnoseFailureForExpr` という関数が型検査の途中で呼ばれる。

```
┗ TypeChecker::typeCheckExpression
  ┗ ConstraintSystem::applySolution
    ┗ ConstraintSystem::diagnoseFailureForExpr
```

この関数は省略して書くと、このような構成になっている。

```cpp
void ConstraintSystem::diagnoseFailureForExpr(Expr *expr) {
  // ... (略)

  FailureDiagnosis diagnosis(expr, *this);

  // Swift4ではここでtrueが返ってきていた
  // 本来は下のほうの処理でエラーになる
  if (diagnosis.diagnoseExprFailure())
    return;

	// ... (略)

  // 本来はここでエラーがでる
  if (diagnosis.diagnoseConstraintFailure())
    return;

	// ... (略)
}
```

`diagnosis.diagnoseExprFailure()` は上のプルリクで修正された`TypeChecker::diagnoseArgumentGenericRequirements`  を呼び出している。

```
┗ FailureDiagnosis::diagnoseExprFailure
  ┗ FailureDiagnosis::visit
    ┗ FailureDiagnosis::visitApplyExpr
      ┗ TypeChecker::diagnoseArgumentGenericRequirements (ここ)
        ┗ TypeChecker::checkGenericArguments
          ┗ TypeChecker::conformsToProtocol 
            ┗ FailureDiagnosis::diagnoseConstraintFailure 
```

本来はエラーメッセージは以下の場所で出力される。

```
ConstraintSystem::diagnoseFailureForExpr
  ┗ ConstraintSystem::diagnoseConstraintFailure
    ┗ ConstraintSystem ::diagnoseGeneralConversionFailure
      ┗ TypeChecker::conformsToProtocol
        ┗ diagnoseConformanceFailure (ここで出力)
```

```cpp
TC.diagnose(ComplainLoc, diag::type_does_not_conform,
          T, Proto->getDeclaredType());
```


`CSDiag` はまだほとんど読んでいないので、また時間があるときに読む(予定)


## 付録

AST

```
(source_file
  (struct_decl "NotHashable" interface type='NotHashable.Type' access=internal @_fixed_layout
    (constructor_decl implicit "init()" interface type='(NotHashable.Type) -> () -> NotHashable' access=internal designated
      (parameter_list
        (parameter "self" type='inout NotHashable' interface type='inout NotHashable' mutable))
      (parameter_list)
      (brace_stmt
        (return_stmt implicit))))
  (top_level_code_decl
    (brace_stmt
      (call_expr type='<null>' nothrow arg_labels=_:
        (type_expr type='Set<<<unresolvedtype>>>.Type' location=hashable.swift:2:1 range=[hashable.swift:2:1 - line:2:1] typerepr='Set')
        (paren_expr type='([NotHashable])' location=hashable.swift:2:5 range=[hashable.swift:2:4 - line:2:50]
          (array_expr type='[NotHashable]' location=hashable.swift:2:5 range=[hashable.swift:2:5 - line:2:49]
            (call_expr type='NotHashable' location=hashable.swift:2:6 range=[hashable.swift:2:6 - line:2:18] arg_labels=
              (constructor_ref_call_expr type='() -> NotHashable' location=hashable.swift:2:6 range=[hashable.swift:2:6 - line:2:6]
                (declref_expr implicit type='(NotHashable.Type) -> () -> NotHashable' location=hashable.swift:2:6 range=[hashable.swift:2:6 - line:2:6] decl=hashable.(file).NotHashable.init()@hashable.swift:1:8 function_ref=single)
                (type_expr type='NotHashable.Type' location=hashable.swift:2:6 range=[hashable.swift:2:6 - line:2:6] typerepr='NotHashable'))
              (tuple_expr type='()' location=hashable.swift:2:17 range=[hashable.swift:2:17 - line:2:18]))
            (call_expr type='NotHashable' location=hashable.swift:2:21 range=[hashable.swift:2:21 - line:2:33] arg_labels=
              (constructor_ref_call_expr type='() -> NotHashable' location=hashable.swift:2:21 range=[hashable.swift:2:21 - line:2:21]
                (declref_expr implicit type='(NotHashable.Type) -> () -> NotHashable' location=hashable.swift:2:21 range=[hashable.swift:2:21 - line:2:21] decl=hashable.(file).NotHashable.init()@hashable.swift:1:8 function_ref=single)
                (type_expr type='NotHashable.Type' location=hashable.swift:2:21 range=[hashable.swift:2:21 - line:2:21] typerepr='NotHashable'))
              (tuple_expr type='()' location=hashable.swift:2:32 range=[hashable.swift:2:32 - line:2:33]))
            (call_expr type='NotHashable' location=hashable.swift:2:36 range=[hashable.swift:2:36 - line:2:48] arg_labels=
              (constructor_ref_call_expr type='() -> NotHashable' location=hashable.swift:2:36 range=[hashable.swift:2:36 - line:2:36]
                (declref_expr implicit type='(NotHashable.Type) -> () -> NotHashable' location=hashable.swift:2:36 range=[hashable.swift:2:36 - line:2:36] decl=hashable.(file).NotHashable.init()@hashable.swift:1:8 function_ref=single)
                (type_expr type='NotHashable.Type' location=hashable.swift:2:36 range=[hashable.swift:2:36 - line:2:36] typerepr='NotHashable'))
              (tuple_expr type='()' location=hashable.swift:2:47 range=[hashable.swift:2:47 - line:2:48]))))))))
```
