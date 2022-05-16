# CPU memory order: Part2 total store ordering

As modern x86 follows TSO memory model, Store-Load reordering is allowed. However, sequential execution for Store-Store, Load-Load, Load-Store may be too slow, so modern x86 does [optimized implementation](https://www.zhihu.com/question/23572082) for TSO: in windows speculation(某个窗口时间内的预测执行). Namely out-of-order execution but still in-order commit.

Below is a runtime detection for Store-Load re-ordering on modern x86 refering to [Memory Reordering Caught in the Act](https://preshing.com/20120515/memory-reordering-caught-in-the-act/) 

make sure optimization level is *-O2* so that **Store just precedes Load** as below:

```shell
gcc -O2 -c -S ordering.cpp
```

ordering.s:

```
.L13:
▸       movq▸   %rbx, %rdi
▸       call▸   _ZN15MersenneTwister7integerEv
▸       testb▸  $7, %al
▸       jne▸    .L13
▸       movl▸   $1, X(%rip)          # Store 1 into X
▸       movl▸   Y(%rip), %eax        # Load Y into tmp
▸       leaq▸   endSema(%rip), %rdi
▸       movl▸   %eax, r1(%rip)       # Store tmp into r1
▸       call▸   sem_post@PLT
▸       jmp▸    .L14
▸       .cfi_endproc
```

As a result,  # Store 1 into X and # Load Y into tmp may be re-ordered. The result on Intel Xeon 8180:

```shell
➜  gcc ./ordering
1 reorders detected after 1209 iterations
2 reorders detected after 7438 iterations
3 reorders detected after 131322 iterations
4 reorders detected after 302651 iterations
5 reorders detected after 741857 iterations
6 reorders detected after 750351 iterations
7 reorders detected after 775165 iterations
8 reorders detected after 778083 iterations
9 reorders detected after 1095603 iterations
```

To force ordering, SFENCE is needed to insert between Store and Load:

```
.L13:
▸       movq▸   %rbx, %rdi
▸       call▸   _ZN15MersenneTwister7integerEv
▸       testb▸  $7, %al
▸       jne▸    .L13
▸       movl▸   $1, X(%rip)
#APP
# 88 "ordering.cpp" 1
▸       mfence
# 0 "" 2
#NO_APP
▸       movl▸   Y(%rip), %eax
▸       leaq▸   endSema(%rip), %rdi
▸       movl▸   %eax, r1(%rip)
▸       call▸   sem_post@PLT
▸       jmp▸    .L14
▸       .cfi_endproc
```

So no re-ordering can be detected.

Reference:

1. The Significance of the x86 SFENCE Instruction https://hadibrais.wordpress.com/2019/02/26/the-significance-of-the-x86-sfence-instruction/

2. what is the effects of _mm_lfence and _mm_sfence https://community.intel.com/t5/Intel-C-Compiler/what-is-the-effects-of-mm-lfence-and-mm-sfence/td-p/871883

3. Intel® 64 and IA-32 Architectures Software Developer Manuals [Intel® 64 and IA-32 Architectures Software Developer Manuals](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html) Volum 3 Chapter 11 Memory Cache Control, Volume 3 Chapter 8.2 Memory Ordering
