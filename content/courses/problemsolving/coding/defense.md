---
linktitle: "调试艺术"
title: "调试艺术"
type: docs
math: true
date: 2023-05-18

menu:
    ps-coding:
        parent: Contents
        weight: 7
---

### 调试理论

为什么 debug 如此困难？因为 bug 的传播链总是非常长。我们给出如下三个概念：

* Fault: 这是 bug 产生的地方，例如你失手打错了循环变量或者逻辑运算符。
* Error: 这是 bug 第一次导致程序的内部状态与正确状态发生偏离的地方 (例如某个变量的值不正确)。
* Failure: 这是你最早能观测到 bug 的地方，例如你的程序输出了错误的结果，或者发生了段错误。

Debug 的本质就是在观测到 Failure 后向前追溯找到 Fault 的过程。

我们通过一个例子来体会这三个概念：

```c++
for (int i = 1; i <= n; i++)
    for (int j = i + 2; j <= n; j++)
        if (a[i] > a[j]) swap(a[i], a[j]);
// lots of other code
for (int i = 1; i <= n; i++)
    printf("%d ", a[i]);
```

上面的代码块展示了一个选择法排序的实现。在这个例子中

* Fault 发生在第二行：`j` 应当从 `i + 1` 开始循环而不是 `i + 2`。
* Error 发生的地方很难确定，在某些输入下，这个“错误”的选择法排序仍然能输出正确的结果。但如果在某一轮， `a[i+1]` 恰好是最小值而 `j` 略过了 `i+1`，导致该轮结束后 `a[i]` 存储的不是 `a[i...n]` 中的最小值，那么 a 数组的数据的状态就与正确状态发生了偏离，产生了 Error。
* Failure 发生在打印的地方，你发现在某些输入下输出的序列并不是有序的。

通过这个例子，我们可以从理论角度总结出 bug 难找的一些原因和启发性的调试思路：

* Fault 并不一定能立刻转化成 Error，甚至在某些输入下不会产生 Error。
  $\Longrightarrow$ <b>我们需要生成更多的输入对程序进行全方面检查。Differential testing 中包含的自动化测试的思想可以视作一种解决方案。</b>
* 对于程序员来说，容易观察到的是 Fault 和 Failure (前者在源代码中，后者有明显的症状)，而 Error 难以观测，因为程序的内部状态 (内存，寄存器，etc.) 不是直接可见的，且程序员不容易想清楚一个正确的中间状态应该长什么样。$\Longrightarrow$ <b>我们需要想办法以人类可以理解的方式让程序员观测到程序的内部状态。</b>
* 从 Error 到 Failure 往往要经过很多代码，因此 debug 时需要向前看很多代码。$\Longrightarrow$ <b>我们需要想办法让 Error 迅速地暴露为 Failure。</b>

分析清楚了 debug 困难的原因，我们就可以对症下药，给出一些有针对性的调试技巧。由于大家对计算机的底层细节尚不了解，本讲义主要总结一些在源代码层面/通过工具可以轻松完成的技巧。

### 打印

打印是最朴素也最有效的 debug 方法之一，它的理论依据是调试理论的第二条困难——打印可以帮助我们查看程序的中间状态。以之前的选择法排序为例，在看到最终排序结果错误时，你最可能采取的 debug 方法就是在循环中添加打印语句，在每轮内层循环结束后查看当前数组的情况，于是你容易发现在某个特定的轮次元素顺序错误，这也就帮你从 failure 追溯到 error 了。

### GDB

GDB 是 C/C++ 程序员人手一个的王牌调试器，它的理论依据也是调试理论的第二条，但它比打印更加灵活和强大——你可以在任何一个你想要的时刻让程序暂停，然后查看任何你想要看的程序状态 (各个变量的值，寄存器的值，内存的值，……)。如果你喜欢 CLI，你可以用命令行打开 gdb，但我们更推荐你使用一些与图形化界面集成在一起的 gdb 工具 (例如 vscode 的 gdb)，它可以给你带来更好的调试体验。

GDB 唯一的缺点是上手难度比较高，我们这里给出 [官方手册](https://sourceware.org/gdb/current/onlinedocs/) ，其中有完整的文档 (将近 1000 页 🤯)，还有 [cheatsheet](https://darkdust.net/files/GDB%20Cheat%20Sheet.pdf)。不过在大语言模型时代，让 LLM 帮助你阅读 1000 多页的手册总是不坏的，如果你有任何使用问题，你可以写一个 prompt 丢给 ChatGPT，它通常能给你非常不错的建议。这里我们总结几条目前对于大家来说比较重要的 GDB feature:
* 断点 (breakpoint): 你可以在程序中打断点，程序运行到断点后就会暂停，供你查询各种程序状态。
* 打印：你可以用打印任何你想要的内容，但如果你使用 CLI，你可能要学习各种小技巧让 GDB 输入人类可以理解的内容 (如 `p/s` `p/i` `p/x` 等)，多问问 ChatGPT/多读手册。
* 监视点 (watchpoint)：你可以给某个变量/某个地址打监视点，在之后运行的过程中一旦监视点的值发生变化 GDB 就会暂停下来供你调试。

为了说服你克服学习新事物的惰性并多用 GDB，这里我们展示一个 GDB 极其好用的场景。假设你的程序 `error.cpp` 发生了段错误，寻找到底是哪条语句触发了段错误本身就不是一件简单的事情。但如果你使用 GDB，你可以使用 `-g` 参数编译代码，然后启动 GDB 运行可执行文件。
```bash
g++ -o error error.cpp -g && gdb ./error
```
之后直接输入 `run` 命令运行，你可以看到段错误发生，然后使用 `where` 命令查看函数调用链，你可以清楚地看到源程序中触发段错误的行号，这极大加速了 debug 的过程。

### 防御性编程

防御性编程 (defensive programming) 指的是在程序中加入一些显式的断言 (assertion) 来对程序的状态进行检查。这恰恰对应了调试理论中的第三条困难的解决方案。我们以书写平衡树的旋转操作为例：

```c++
void rotate(Node *u)
{
    // 结构约束
    assert(u->parent == u /* u is root */ || u->parent->left == u || u->parent->right == u);
    assert(!u->left || u->left->parent == u);
    assert(!u->right || u->right->parent == u);
    // 数值约束
    assert(!u->left || u->left->val < u->val);

    // rotation code
}
```

`assert(expr)` 的语义是：如果其中包含的逻辑表达式 `expr` 值为假，则抛出异常 (在没有 error handler 的情况下这通常会使得程序直接终止，注意程序终止是一种 Failure)。例子中的这些 assertion 描述了一个平衡树节点理应满足的性质，它们看上去非常地显然，但如果你的程序有 bug，你的错误很可能被这些 assertion 抓住。这使得你可以从尽可能接近 Error 的地方出发寻找 Fault。

防御性编程的代价在于，作为程序员你需要额外写很多代码，以及有些 assertion 在抓 error 方面很有价值，有些则可能无关紧要，这其中的判断力需要你不断积累经验。我们的建议是：能加尽可能多加，写几条 assertion 的时间和深夜 debug 到头秃的时间相比不值一提。

### Sanitizers

手写 assertion 终究还是太过繁琐。例如从极其严谨的角度来说，你应当将数组封装成这样：

```c++
class Array
{
    int N = 1000, a[1000];
public:
    void store(int pos, int value)
    {
        assert(0 <= pos && pos < N);
        a[pos] = value;
    }
    int access(int pos)
    {
        assert(0 <= pos && pos < N);
        return a[pos];
    }
};
```

这两条 assertion 可以帮助你检查对数组的越界操作，~~但如果把程序写成这样，那程序员可别活了~~。有没有什么工具可以帮我们在每次访问数组时自动检查是否越界？

> **计算机世界的两条公理**
> 1. 机器永远是对的，未测试代码永远是错的。
> 2. 你如果发现自己有某种需求，一定有某种工具可以帮助你实现它。

根据公理第二条，这样的工具应该是存在的。这里我们为大家介绍 Address Sanitizer (它好像应当被翻译成地址消毒剂，但我从未见过这样的表达)。Address Sanitizer (ASAN) 可以检查 C/C++ 程序中与内存访问相关的错误，主流的编译器/IDE，例如 Visual Studio, GCC, Clang 都支持 ASAN。下面是一个例子：

```c++
// error.cpp
#include <bits/stdc++.h>
using namespace std;

int a[10];

int main ()
{
    a[11] = 1;
    return 0;
}
```

该程序在 `main()` 函数中包含了对全局数组的越界读取。虽然直接编译运行这个程序(通常)不会导致段错误，但这样的操作仍然是危险的，因为它属于 [undefined behavior](https://en.wikipedia.org/wiki/Undefined_behavior)。我们来看看使用 ASAN 会得到什么样的结果 (在编译命令中加上 `-fsanitize=address` 即可使用 ASAN)：

```bash
> g++ -o error error.cpp -fsanitize=address && ./error
==40526==ERROR: AddressSanitizer: global-buffer-overflow on address 0x55c41dbbdf6c at pc 0x55c41dbb928b bp 0x7ffda7143ce0 sp 0x7ffda7143cd0
WRITE of size 4 at 0x55c41dbbdf6c thread T0
    #0 0x55c41dbb928a in main (error+0x228a)
    #1 0x7ff6ab1a0564 in __libc_start_main (/lib/x86_64-linux-gnu/libc.so.6+0x28564)
    #2 0x55c41dbb918d in _start (error+0x218d)

0x55c41dbbdf6c is located 4 bytes to the right of global variable 'a' defined in 'error.cpp:5:5' (0x55c41dbbdf40) of size 40
SUMMARY: AddressSanitizer: global-buffer-overflow (error+0x228a) in main
Shadow bytes around the buggy address:
  0x0ab903b6fb90: f9 f9 f9 f9 f9 f9 f9 f9 f9 f9 f9 f9 f9 f9 f9 f9
  0x0ab903b6fba0: f9 f9 f9 f9 f9 f9 f9 f9 f9 f9 f9 f9 f9 f9 f9 f9
  0x0ab903b6fbb0: f9 f9 f9 f9 f9 f9 f9 f9 f9 f9 f9 f9 f9 f9 f9 f9
  0x0ab903b6fbc0: f9 f9 f9 f9 f9 f9 f9 f9 f9 f9 f9 f9 f9 f9 f9 f9
  0x0ab903b6fbd0: f9 f9 f9 f9 f9 f9 f9 f9 f9 f9 f9 f9 00 00 00 00
=>0x0ab903b6fbe0: 01 f9 f9 f9 f9 f9 f9 f9 00 00 00 00 00[f9]f9 f9
  0x0ab903b6fbf0: f9 f9 f9 f9 00 00 00 00 00 00 00 00 00 00 00 00
  0x0ab903b6fc00: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x0ab903b6fc10: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x0ab903b6fc20: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x0ab903b6fc30: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07 
  Heap left redzone:       fa
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
  Left alloca redzone:     ca
  Right alloca redzone:    cb
  Shadow gap:              cc
==40526==ABORTING
```

可以看到 ASAN 提醒我们全局数组越界，甚至告诉了我们是在第五行定义的 `a` 数组合法范围向右偏移 4 字节的地方发生了越界访问。有了这些信息 debug 将会变得非常方便 (更重要的是，它告诉了我们 bug 的存在！)。

{{% alert tip %}}

**为什么这样就有段错误了？**

```c++
int main ()
{
    int a[10];
    a[11] = 1; // 等等，怎么换成 a[100] = 1 就又没有段错误了 😵‍💫
    return 0;
}
```

非常好的观察！解释清楚这个问题需要较多的计算机底层知识，如果你真的感兴趣，可以上网搜索 Stack Canary 相关的内容。 

{{% /alert %}}