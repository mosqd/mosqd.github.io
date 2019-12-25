---
title: 利用rip寄存器实现循环
abbrlink: 73ea24f2
date: 2019-12-24 13:30:01
tags:
  - cs
---
想要循环调用一个函数，其实有很多种方法，除了以下常见实现方式（for ,while），还可以通过修改rip寄存器来实现循环，文章末尾会介绍具体的案例。
## 常见循环实现
```c
// 1. for
for (;;) {
    loop();
}

// 2. while
while (true) {
    loop();
}

// 3. do while
do {
    loop();
} while (true);

// 4. goto
label:
    loop();
    goto label;
```

## rip实现循环
先来看一段代码

```c
// loop.c
#include <stdio.h>

void loop() {
    long i = 111;
    printf("in loop: %p\n", &i);
    // 关键代码
    *(&i+2) -= 5;
}

int main(int argc, char* argv[])
{
    long i = 110;
    loop();
    printf("before loop: %p\n", &i);
    return 0;
}
```
```shell
# 执行
$ clang loop.c
$ ./a.out
in loop: 0x7ffee225f848
in loop: 0x7ffee225f848
in loop: 0x7ffee225f848
in loop: 0x7ffee225f848
in loop: 0x7ffee225f848
in loop: 0x7ffee225f848
in loop: 0x7ffee225f848
in loop: 0x7ffee225f848
in loop: 0x7ffee225f848
in loop: 0x7ffee225f848
in loop: 0x7ffee225f848
in loop: 0x7ffee225f848
in loop: 0x7ffee225f848
in loop: 0x7ffee225f848
^C
```
为什么会出现这种效果？让我们先了解一下程序的执行过程。
### 运行时栈
main函数中调用loop，调用堆栈如下图所示：
![stack _1_.jpg](https://i.loli.net/2019/12/24/Orp843FZzL5yIoY.jpg)
在main函数的栈帧中可以看到，会将loop函数（例子中没有参数）参数入栈，随后保存PC（即main中printf的地址）。通过call指令实现将PC值push到栈中，并将PC设置为loop函数的地址。loop执行后调用ret指令，从栈中保存的返回地址恢复到PC中，从而main函数继续执行。

> PC (program counter), 程序计数器，在x86-64中，用%rip寄存器表示。
> The instruction pointer register points to the memory address which the processor will next attempt to execute.

### 反汇编
现在我们大致了解了程序执行时栈的变化过程，要想实现loop函数循环，只要将saved PC在栈中的值改为loop()这条指令的地址。接下来，让我们看看a.out的关键部分的反汇编代码。

```shell
# objdump反汇编a.out
$ objdump -D a.out > a.obj
```

![反汇编.png](https://i.loli.net/2019/12/24/FD198mlPCnTcBjS.png)

27行汇编 callq调用loop的时候，会将下一条指令（leaq）的地址写入栈中，同时将PC设置为loop的地址，实现调用loop的效果，callq和leaq的地址相差5，在loop函数中，我们可以通过变量i来寻找saved PC的在栈中的地址，从来达到loop返回后，继续执行loop的效果。

```c
# &i+2是saved PC的地址
# 只要将栈中的数据改为callq的地址即可
*(&i+2) -= 5;
```

## 案例-go defer实现
在函数ret之前，调用deferreturn，deferreturn会循环调用，检查defer函数是否全部调用完成。

```c
runtime.jmpdefer:
 104f890:   48 8b 54 24 08  movq    8(%rsp), %rdx
 104f895:   48 8b 5c 24 10  movq    16(%rsp), %rbx
 104f89a:   48 8d 63 f8     leaq    -8(%rbx), %rsp
 104f89e:   48 8b 6c 24 f8  movq    -8(%rsp), %rbp
 104f8a3:   48 83 2c 24 05  subq    $5, (%rsp)     // 修改saved PC
 104f8a8:   48 8b 1a    movq    (%rdx), %rbx
 104f8ab:   ff e3   jmpq    *%rbx
 104f8ad:   cc  int3
 104f8ae:   cc  int3
 104f8af:   cc  int3

runtime.deferreturn:
 10264a0:   48 83 ec 48     subq    $72, %rsp
 10264a4:   48 89 6c 24 40  movq    %rbp, 64(%rsp)
 10264a9:   48 8d 6c 24 40  leaq    64(%rsp), %rbp
 10264ae:   65 48 8b 04 25 a0 08 00 00  movq    %gs:2208, %rax
 10264b7:   48 8b 48 28     movq    40(%rax), %rcx
 ...
 1026534:   48 8d 44 24 50  leaq    80(%rsp), %rax
 1026539:   48 89 44 24 38  movq    %rax, 56(%rsp)
 102653e:   48 8b 44 24 20  movq    32(%rsp), %rax
 1026543:   48 89 04 24     movq    %rax, (%rsp)
 1026547:   48 8b 44 24 38  movq    56(%rsp), %rax
 102654c:   48 89 44 24 08  movq    %rax, 8(%rsp)
 1026551:   e8 3a 93 02 00  callq   168762 <runtime.jmpdefer>
 1026556:   48 8b 6c 24 40  movq    64(%rsp), %rbp
 102655b:   48 83 c4 48     addq    $72, %rsp
 102655f:   c3  retq
 ...
```

