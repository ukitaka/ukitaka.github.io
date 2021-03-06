---
layout: post
title:  Rustのバージョン管理 rsvmの使い方 あるいは rustcのデバッグオプションの触り方メモ
---

`rustc` のデバッグ用のオプションを触りたかったのだけど、どうやらnightlyビルドじゃないとだめらしい。

```
$ rustc -Z help
warning: the option `Z` is unstable and should only be used on the nightly compiler, but it is currently accepted for backwards compatibility; this will soon change, see issue #31847 for more details
```

ので、どうやら`rsvm`というものがあるらしいので使ってみた。

## Homebrewで入れたRust toolchainを削除


```
brew unininstall rust
```


## rsvmを使ってnightlyのRustをインストール

+ [sdepold/rsvm](/Users/st20841/.rsvm/current/cargo/bin)

READMEの通りにインストール。

```
$ curl -L https://raw.github.com/sdepold/rsvm/master/install.sh | sh
```

`rsvm ls-remote` でインストール可能な一覧を見る。

```
$ rsvm ls-remote
0.10
0.11.0
0.12.0
1.0.0
1.0.0-alpha
1.0.0-alpha.2
1.0.0-beta
1.0.0-beta.2
1.0.0-beta.3
1.0.0-beta.4
1.0.0-beta.5
1.1.0
1.10.0
1.11.0
1.12.0
1.12.1
1.13.0
1.14.0
1.15.0
1.15.1
1.17.0
1.18.0
1.19.0
1.2.0
1.20.0
1.3.0
1.4.0
1.5.0
1.6.0
1.7.0
1.8.0
1.9.0
1.16.0
1.10.0-rc
beta
nightly
``` 

`nightly` があることが確認できたので、インストール。

```
$ rsvm install nightly
```


インストールされたことを`rsvm ls` で確認

```
$ rsvm ls
Installed versions:

  =>  nightly.20170921133940
```


`rsvm use` で使うバージョンを指定

```
$ rsvm use nightly.20170921133940
Activating rust nightly.20170921133940 ... done
```


ちゃんとnightlyになっていることが確認できる。

```
$ rustc --version
rustc 1.22.0-nightly (01c65cb15 2017-09-20)
```


## デバッグ用オプション動作確認

無事使えるようになった。

```
$ rustc -Z help

Available debug options:

    -Z                      verbose -- in general, enable more debug printouts
    -Z            span-free-formats -- when debug-printing compiler state, do not include spans
    -Z             identify-regions -- make unnamed regions display as '# (where # is some non-ident unique id)
    -Z             emit-end-regions -- emit EndRegion as part of MIR; enable transforms that solely process EndRegion
    -Z                 borrowck-mir -- implicitly treat functions as if they have `#[rustc_mir_borrowck]` attribute
    -Z                  time-passes -- measure time of each rustc pass
    -Z             count-llvm-insns -- count where LLVM instrs originate
    -Z             time-llvm-passes -- measure time of each LLVM pass
    -Z                  input-stats -- gather statistics about the input
    -Z                  trans-stats -- gather trans statistics
    -Z                 asm-comments -- generate comments into the assembly (may change behavior)
    -Z                    no-verify -- skip LLVM verification
    -Z               borrowck-stats -- gather borrowck statistics
    -Z              no-landing-pads -- omit landing pads for unwinding
    -Z                   debug-llvm -- enable debug output from LLVM
    -Z                   meta-stats -- gather metadata statistics
    -Z              print-link-args -- print the arguments passed to the linker
    -Z            print-llvm-passes -- prints the llvm optimization passes being run
    -Z                     ast-json -- print the AST as JSON and halt
    -Z            ast-json-noexpand -- print the pre-expansion AST as JSON and halt
    -Z                           ls -- list the symbols defined by a library crate
    -Z                save-analysis -- write syntax and type analysis (in JSON format) information, in addition to normal output
    -Z         print-move-fragments -- print out move-fragment data for every fn
    -Z        flowgraph-print-loans -- include loan analysis data in --unpretty flowgraph output
    -Z        flowgraph-print-moves -- include move analysis data in --unpretty flowgraph output
    -Z      flowgraph-print-assigns -- include assignment analysis data in --unpretty flowgraph output
    -Z          flowgraph-print-all -- include all dataflow analysis data in --unpretty flowgraph output
    -Z           print-region-graph -- prints region inference graph. Use with RUST_REGION_GRAPH=help for more info
    -Z                   parse-only -- parse only; do not compile, assemble, or link
    -Z                     no-trans -- run all passes except translation; no output
    -Z             treat-err-as-bug -- treat all errors that occur as bugs
    -Z   continue-parse-after-error -- attempt to recover from parse errors (experimental)
    -Z              incremental=val -- enable incremental compilation (experimental)
    -Z               incremental-cc -- enable cross-crate incremental compilation (even more experimental)
    -Z             incremental-info -- print high-level information about incremental reuse (or the lack thereof)
    -Z        incremental-dump-hash -- dump hash information in textual format to stdout
    -Z               dump-dep-graph -- dump the dependency graph to $RUST_DEP_GRAPH (default: /tmp/dep_graph.gv)
    -Z              query-dep-graph -- enable queries of the dependency graph for regression testing
    -Z              profile-queries -- trace and profile the queries of the incremental compilation framework
    -Z     profile-queries-and-keys -- trace and profile the queries and keys of the incremental compilation framework
    -Z                  no-analysis -- parse and expand the source, but run no analysis
    -Z            extra-plugins=val -- load extra plugins
    -Z             unstable-options -- adds unstable command line options to rustc interface
    -Z    force-overflow-checks=val -- force overflow checks on or off
    -Z                 trace-macros -- for every macro invocation, print its name and arguments
    -Z                 debug-macros -- emit line numbers debug info inside macros
    -Z enable-nonzeroing-move-hints -- force nonzeroing move optimization on
    -Z            keep-hygiene-data -- don't clear the hygiene data after analysis
    -Z                     keep-ast -- keep the AST after lowering it to HIR
    -Z                show-span=val -- show spans for compiler debugging (expr|pat|ty)
    -Z             print-type-sizes -- print layout information for each type encountered
    -Z        print-trans-items=val -- print the result of the translation item collection pass
    -Z            mir-opt-level=val -- set the MIR optimization level (0-3, default: 1)
    -Z                 dump-mir=val -- dump MIR state at various points in translation
    -Z             dump-mir-dir=val -- the directory the MIR is dumped into
    -Z dump-mir-exclude-pass-number -- if set, exclude the pass number when dumping MIR (used in tests)
    -Z        mir-emit-validate=val -- emit Validate MIR statements, interpreted e.g. by miri (0: do not emit; 1: if function contains unsafe block, only validate arguments; 2: always emit full validation)
    -Z                   perf-stats -- print some performance-related statistics
    -Z                    hir-stats -- print some statistics about AST and HIR
    -Z                    mir-stats -- print some statistics about MIR
    -Z            always-encode-mir -- encode MIR of all functions into the crate metadata
    -Z       osx-rpath-install-name -- pass `-install_name @rpath/...` to the macOS linker
    -Z                sanitizer=val -- Use a sanitizer
    -Z            linker-flavor=val -- Linker flavor
    -Z                     fuel=val -- set the optimization fuel quota for a crate
    -Z               print-fuel=val -- make Rustc print the total optimization fuel used by a crate
    -Z   remap-path-prefix-from=val -- add a source pattern to the file path remapping config
    -Z     remap-path-prefix-to=val -- add a mapping target to the file path remapping config
    -Z   force-unstable-if-unmarked -- force all crates to be `rustc_private` unstable
    -Z             pre-link-arg=val -- a single extra argument to prepend the linker invocation (can be used several times)
    -Z            pre-link-args=val -- extra arguments to prepend to the linker invocation (space separated)
    -Z                      profile -- insert profiling code
    -Z              relro-level=val -- choose which RELRO level to use
    -Z                          nll -- run the non-lexical lifetimes MIR pass
    -Z             trans-time-graph -- generate a graphical HTML report of time spent in trans and LLVM
```
