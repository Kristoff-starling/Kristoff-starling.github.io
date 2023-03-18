---
linktitle: "快速幂"
title: "快速幂"
type: docs
math: true
date: 2023-03-18

menu:
    ps-algorithms:
        parent: Contents
        weight: 1
---

给定 $a,n$，考虑如何计算 $a^n$ 。如果使用如下循环计算结果

```c++
int res = 1;
for (int i = 1; i <= n; i++)
    res = res * a;
```

时间复杂度为 $O(n)$，当 $n$ 很大时效率太低。快速幂是用于计算该类问题的常见算法，其本质思想是递归/倍增:

$$
a^n=\begin{cases}1&,n=0\\\\(a^{n/2})^2&,n是偶数, n\geq 1\\\\(a^{\lfloor n/2\rfloor})^2\cdot a&,n是奇数, n\geq 1\end{cases}
$$

因此我们可以通过不断折半幂次的方式来求解 $a^n$。写出该算法的复杂度递归表达式：

$$
T(n)=T(n/2)+O(1)
$$

容易得出 $T(n)=O(\log n)$。

直接根据递归式，我们容易写出如下递归代码:

```c++
int quick_pow(int a, int n)
{
    if (n == 0) return 1;
    res = quick_pow(a, n / 2);
    if (n % 2 == 0)
        return res * res;
    else
        return res * res * a;
}
```

但我们通常使用一种更加优美的写法来使用循环替代递归：

```c++
int quick_pow(int a, int n)
{
    int res = 1;
    while (n)
    {
        if (n & 1) res = res * a;
        a = a * a; n >>= 1;
    }
    return res;
}
```

该实现的核心思想是利用 $n$ 的二进制表示中 1 的位置来判断 $a^n$ 是由哪些 $a^{2^k}$ 组合而成的。即如果 $n$ 的二进制表示中在 $k_1,k_2,\cdots, k_m$ 这些位置上是 1，那么

$$
a^n=a^{\sum_{i=1}^m2^{k_i}}=\prod_{i=1}^ma^{2^{k_i}}
$$