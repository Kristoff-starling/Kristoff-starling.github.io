---
linktitle: "逆元"
title: "逆元"
type: docs
math: true
date: 2023-05-13

menu:
    ps-coding:
        parent: Contents
        weight: 5
---

对于整数 $a, b<p$，计算 $(a/b)\text{ mod }p$ 可以转化为计算 $a\cdot b^{-1}\text{ mod }p$，这里 $b^{-1}$ 称为 $b$ 在模 $p$ 下的**逆元**，可以证明当 $p$ 为质数时，$b^{-1}$ 在 $[0, p)$ 范围内是唯一的。逆元的数学推导会在后续的理论课程中给出，这里不做赘述 (如果你不懂也暂时不必深究)。本文主要阐述人们通常是如何书写计算逆元的代码。

当 $p$ 为质数时，计算逆元最简单的方法之一是使用费尔马小定理。由于

$$a^{p-1}\equiv 1(\text{mod }p)$$

所以令 $a^{-1}=a^{p-2}\text{ mod }p$，则有 $a\cdot a^{-1}\equiv 1(\text{mod }p)$。这里的 $a^{-1}$ 即为 $a$ 在模 $p$ 意义下的逆元。你可以使用快速幂算法快速计算 $a^{p-2}$ 的值，时间复杂度为 $O(\log p)$。

另外一种常见的需求是计算 $[1, n]$ 中所有数的逆元。如果对每个数用费尔马小定理计算逆元，时间复杂度将达到 $O(n\log p)$，有时不可接受。前人发明过一些非常精巧的算法递推地 $O(n)$ 求出所有数的逆元，如果你感兴趣可以参考 [这篇博客](https://blog.csdn.net/bcr_233/article/details/87898670)。但我们更推荐你采用如下非常简洁的做法：

* 顺序递推求出 $1!, 2!, \cdots, n!$。
* 使用快速幂计算 $(n!)^{-1}$。
* 从后往前递推算出所有阶乘的逆元 (注：$(k!)^{-1}\cdot k=((k-1)!)^{-1}$)。
* 对于 $x$，$x^{-1}=(x-1)!\cdot (x!)^{-1}$。

该做法虽然看上去多了几遍递推，但时间复杂度仍为 $O(n)$，且通常拥有更加优秀的常数因子 (即在实际运行中效率可能反而比链接中的一遍递推要高，因为链接中的做法有大量的取模操作)。

<details><summary><b>Reference</b> :: <i>click to expand</i></summary>

```c++
const int MOD = 998244353;
int fac[MAXN], ifac[MAXN], inv[MAXN];

void init_inv()
{
    fac[0] = 1;
    for (int i = 1; i <= n; i++)
        fac[i] = 1ll * fac[i - 1] * i % MOD;
    ifac[n] = quick_pow(fac[n], MOD - 2);
    for (int i = n - 1; i >= 0; i--)
    {
        ifac[i] = 1ll * ifac[i + 1] * (i + 1) % MOD;
        inv[i + 1] = 1ll * fac[i] * ifac[i + 1] % MOD;
    }
}
```

</details>