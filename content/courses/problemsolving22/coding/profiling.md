---
linktitle: "性能优化"
title: "性能优化"
type: docs
math: true
date: 2023-05-15

menu:
    ps-coding:
        parent: Contents
        weight: 6
---

> Premature optimization is the root of all evil. -- Donald Knuth

性能优化的需求非常普遍——在 OJ 层面，这主要体现为将 TLE 的程序改到 AC。本文旨在对于 OJ 层面的性能优化问题给予一些最基本的指导。

### 计算程序的运行时间

衡量一个程序的性能的指标有很多，其中最简单、最直接的方式就是运行时间，因此你至少应该学会如何计算一个程序的运行时间。
* 如果你使用类 unix 系统，你可以直接使用 `time` 命令。
* 如果你使用的系统没有可以直接测算时间的命令，你可以使用 C/C++ 库的 `time()` 函数来打印运行时间，具体的使用方法请自行上网搜索。

### “对抗”式的输入构造策略

假设你写了一个如下的快速排序程序：

```c++
void quick_sort(int l, int r)
{
    if (l == r) return;
    int pos = partition(l, r, a[l]);
    quick_sort(l, pos - 1);
    quick_sort(pos + 1, r);
}
```

这个每次选择第一个数作为 pivot 的快速排序程序在随机数据上可以给到 $O(n\log n)$ 的时间复杂度，但假设你现在是一个“找茬”的人，为了让这个程序跑得很慢，你一定会构造一个很长且原本有序的数列，这样每次 `partition()` 只能去掉 pivot 一个数，从而时间复杂度退化到 $O(n^2)$。

对于算法题来说，我们通常在意的是算法的“最坏时间复杂度”。所以测试程序性能时，你应该代入“找茬”的角色，去思考什么样的输入能将程序卡到最慢。这也是 online judge 对大家的程序进行性能测试时需要考虑的点。

### 寻找程序运行的时间瓶颈

假设你写了一个如下的排序程序：

```c++
int a[100000];

int main ()
{
    for (int i = 1; i <= n; i++)
        scanf("%d", a + i);
    for (int i = 1; i <= n; i++)
        for (int j = i + 1; j <= n; j++)
            if (a[i] > a[j])
                swap(a[i], a[j]);
}
```

并发现它在处理 $n=10^5$ 的数列时超时了。此时你有两个优化方案：

* 用“快速输入输出”章节中的 `getchar()` 方法代替 `scanf()` 进行读入。
* 修改排序算法，使用归并排序。

你一定会采纳第二条建议，因为这个程序运行的时间瓶颈在核心排序算法的部分——它的复杂度达到了 $O(n^2)$，而输入部分是线性的。换句话说，对着仅占总运行时间 $1\%$ 的部分优化，即使你让该部分快了一倍，它对整个程序的优化效果也是微乎其微的。因此，在做性能优化时你应该先仔细分析哪个部分是耗时最长的，对着耗时最长的 critical path 优化才能取得最显著的效果。至于如何找出运行时间最慢的部分，对于 OJ 程序来说，最简单的方法是注释掉某些部分，然后观察程序的运行时长有无显著的变化。

再举一个例子：

```c++
for (int i = 1; i <= n; i++)
{
    a[i] = 0;
    for (int j = i + 1; j <= n; j++)
        a[i] += compute(b[j]);
    a[i] %= MOD;
}
```

假设你通过定位确定了这个程序段是效率瓶颈，此时你有两个优化方案：

* 将 `a[i] %= MOD` 改写为效率更高的减法 (内层循环也要同步修改)。
* 优化函数 `compute()` 的效率。

你仍然应该选择第二条方案。虽然取模的效率不高，但这条语句不在最内层循环。换句话说，从时间复杂度的角度来讲，取模操作被执行了 $O(n)$ 次，而 `compute()` 被执行了 $O(n^2)$ 次。因此，我们最需要关注的，是**效率瓶颈模块的最内层语句**。

{{% alert note %}}

**Profiling: The Real World**

在真实世界中，profiling 也是被广泛使用的一项技术。

> “*Computer Architecture: A Quantitative Approach* 这本书对于计算机体系结构的 insight 可以简单概括为两条：(1)处理器也是一个编译器。(2) 木桶效应。” —— jyy

这里的“木桶效应”指的是：一个计算机系统的真实性能由最短的那块木板决定。因此优化工程师的日常工作便是盯着 profiling report，找“最短的木板”，尝试让它变长一点，再去寻找新的短板。

现实世界中 profiling 的工具有很多 (比“注释-测试”的 OJ 程序法要方便)，它们的主要原理是运行大量的单元测试/压力测试，然后统计每个“基本单元”运行时长占总时长的比例。这里的“基本单元”对于工作在不同层级的人来说有不同的颗粒度。对于软件系统的开发者，“基本单元”可能是颗粒度较粗的高级语言语句、函数甚至模块；对于编译器/硬件开发者，“基本单元”则是中间代码、机器指令甚至是微指令。

{{% /alert %}}