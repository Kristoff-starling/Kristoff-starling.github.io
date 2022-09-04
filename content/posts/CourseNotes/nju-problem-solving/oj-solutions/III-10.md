---
title: "问题求解 III - 10 题解"
linktitle: 'III-10 Sol'
type: docs
draft: false

math: true

menu:
    oj-solutions:
        parent: Contents
        weight: 310
---

## A. 加密

### 题面描述

* 使用题目中给定的一套类 RSA 算法加密，公钥 $E=2^{30}+3$。已知密文 $w^E\equiv C(mod\space n)$，给定 $n,C$，求原文 $w$。
* $T\leq 10^5,n\leq 10^{18}$。

### 题解

根据题目中 $n$ 的生成方式可知，$n=pq$ 且 $p,q$ 的差非常小，因此可以从 $\sqrt n$ 开始往上往下枚举第一个 $n$ 的因子，即可得到 $p$ 和 $q$。那么 $\phi(n)=(p-1)(q-1)$，从而可以用扩展欧几里得求出 $E$ 关于 $\phi(n)$ 的逆元 $D$（密钥）。

根据数论知识，我们有 $C^{DE}\equiv w^{DE}\equiv w(mod\space n)$。因此使用快速幂求解 $C^{DE}$ 即可。

注：如果使用嵌套快速乘的快速幂，时间复杂度将达到 $O(T\log^2n)$，可能会超时。使用 `__int128` 类型可以避免这个问题。

## B. 可以看见的点

### 题面描述

* 给定 $n\times n$ 的方格纸，问从 $(0,0)$ 出发可以画出多少种不同斜率的直线（直线必须经过至少两个格点）。
* $n\leq 10^5$。

### 题解

设直线经过的另一个格点是 $(a,b)$ ，那么显然只有 $a,b$ 互质我们才应该将其计入答案，否则会导致重复计数。因此
$$
\begin{align}
ans\&=\sum_{i=1}^n\sum_{j=1}^n[\gcd(i,j)=1]\\\\
\&=1+2\sum_{i=1}^n\sum_{j=1}^{i-1}[\gcd(i,j)=1]\\\\
\&=1+2\sum_{i=1}^n\varphi(i)
\end{align}
$$因为此题 $n$ 只有 1e5，所以 $O(\log n)$ 地求每个数的欧拉函数值再求和即可。当 $n$ 更大时可以借助线性筛或数论筛法来解决。

## C. POWERMOD

### 题面描述

* 给定 $a,b,m$，求 $a^b(mod\space m)$。
* $a,b,m\leq 10^{18}$。

### 题解

此题的难点在于普通的快速幂在执行乘法时会爆 long long，所以我们应当用“快速乘”来代替普通的乘法。快速乘用 $O(\log n)$ 次加法来代替一次乘法，其原理和快速幂完全一样，大致的代码如下：

```c
#define LL long long
LL quick_mul(LL x, LL y, LL m) {
    LL res = 0;
    while (y)
    {
        if (y & 1) res = (res + x) % m;
        x = (x + x) % m; y >>= 1;
    }
    return res;
}
```

这样我们可以在 $O(\log ^2n)$ 的时间复杂度内完成快速幂。 