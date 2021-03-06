---
layout: post
title: lldbで逆アセンブル結果を見る
---

[GDB Linux X86-64 の呼出規約(calling Convention)を Gdb で確認する](http://th0x4c.github.io/blog/2013/04/10/gdb-calling-convention/)で紹介されているようなことをlldb (+ Clang)でやる場合のメモ。
サンプルプログラムは上記の記事と同じものを利用する。

## Clangでのコンパイル

+ `-g` でデバッグ情報を生成する
+ `-O0` で最適化をしない

```
$ clang sample1.c -g -O0 -o sample1
```


## lldbで逆アセンブル結果を見る

```
$ lldb ./sample1
```


`disassemble --mixed --name func`、もしくは短い `di -m -n func` でもいける。ソースの情報が入らなければ `—mixed` を外す。

```
$ (lldb) disassemble --mixed --name func

   17  	void func()
** 18  	{

sample1`func:
sample1[0x100000ed0] <+0>:   pushq  %rbp
sample1[0x100000ed1] <+1>:   movq   %rsp, %rbp
sample1[0x100000ed4] <+4>:   subq   $0x30, %rsp
sample1[0x100000ed8] <+8>:   movl   $0xb, %edi
sample1[0x100000edd] <+13>:  movl   $0x16, %esi
sample1[0x100000ee2] <+18>:  movl   $0x21, %edx
sample1[0x100000ee7] <+23>:  movl   $0x2c, %ecx
sample1[0x100000eec] <+28>:  movl   $0x37, %r8d
sample1[0x100000ef2] <+34>:  movl   $0x42, %r9d
sample1[0x100000ef8] <+40>:  movl   $0x4d, %eax
sample1[0x100000efd] <+45>:  movl   $0x58, %r10d
sample1[0x100000f03] <+51>:  movl   $0x63, %r11d

** 19  	    int ret = -1;
   20

sample1[0x100000f09] <+57>:  movl   $0xffffffff, -0x4(%rbp)   ; imm = 0xFFFFFFFF

** 21  	    ret = sum(11, 22, 33, 44, 55, 66, 77, 88, 99);
   22

sample1[0x100000f10] <+64>:  movl   $0x4d, (%rsp)
sample1[0x100000f17] <+71>:  movl   $0x58, 0x8(%rsp)
sample1[0x100000f1f] <+79>:  movl   $0x63, 0x10(%rsp)
sample1[0x100000f27] <+87>:  movl   %r11d, -0x8(%rbp)
sample1[0x100000f2b] <+91>:  movl   %r10d, -0xc(%rbp)
sample1[0x100000f2f] <+95>:  movl   %eax, -0x10(%rbp)
sample1[0x100000f32] <+98>:  callq  0x100000e70               ; sum at sample1.c:9
sample1[0x100000f37] <+103>: leaq   0x68(%rip), %rdi          ; "sum: %d\n"
sample1[0x100000f3e] <+110>: movl   %eax, -0x4(%rbp)

** 23  	    printf("sum: %d\n", ret);

sample1[0x100000f41] <+113>: movl   -0x4(%rbp), %esi
sample1[0x100000f44] <+116>: movb   $0x0, %al
sample1[0x100000f46] <+118>: callq  0x100000f84               ; symbol stub for: printf

** 24  	}
   25
   26  	int main(int argc, char *argv[])

sample1[0x100000f4b] <+123>: movl   %eax, -0x14(%rbp)
sample1[0x100000f4e] <+126>: addq   $0x30, %rsp
sample1[0x100000f52] <+130>: popq   %rbp
sample1[0x100000f53] <+131>: retq
```


## 参考

+ [LLDBとGDBのコマンド対応表](http://lldb.llvm.org/lldb-gdb.html)
