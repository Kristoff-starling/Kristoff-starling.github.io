---
linktitle: "矩阵快速幂"
title: "矩阵快速幂"
type: docs
math: true
date: 2023-03-18

menu:
    ps-algorithms:
        parent: Contents
        weight: 2
---

关于基础理论，参见[这篇博文](/posts/recurrences/)。如果你之前从未了解过线性递推相关的知识/目前以完成OJ为主要目标，只需看懂该文章的 Getting Started 章节。本文主要对矩阵乘法和矩阵快速幂的实现给出一点建议。

{{% alert warning %}}

**Don't Panic!**

只有 `Problem Solving` 目录下的内容才是大家需要且应当掌握的内容。这篇博文的大部分内容与问题求解课程并不相关，且我不认为掌握这些数学强相关的知识对计算机思维有太多提升。因此如果你不愿意读 Getting Started 之后的内容/读不懂后面的内容，这非常正常也无关紧要！

{{% /alert %}}

按照线性代数中的矩阵乘法规则，常见的矩阵乘法写法如下：

```c++
for (int i = 1; i <= n; i++)
    for (int j = 1; j <= n; j++)
        for (int k = 1; k <= n; k++)
            c[i][j] += a[i][k] * b[k][j]
```

但是我们强烈建议你调整循环的顺序，按照如下方式书写矩阵乘法：

```c++
for (int i = 1; i <= n; i++)
    for (int k = 1; k <= n; k++)
        for (int j = 1; j <= n; j++)
            c[i][j] += a[i][k] * b[k][j]
```

虽然这样有点“反直觉”，但在稍大规模的数据下运行时，你会发现这种写法的效率比前者高很多。理解效率提高的原因需要对计算机体系结构和 C/C++ 语言中数组在内存中的存储有一定的了解，这里不作赘述。你可以暂且将其背下来 😂

为了更优雅地实现矩阵快速幂，我们强烈建议你学习 C++ Class 相关的内容并了解操作符重载。这样你可以自己定义矩阵类并为矩阵类书写乘法规则，从而直接复用“快速幂”一讲中的代码计算矩阵快速幂。下面给出一个参考实现：

```c++
class Matrix
{
    int b[10][10];
    Matrix () { memset(b, 0, sizeof(b)); }
    void init_I() { for (int i = 1; i <= n; i++) b[i][i] = 1; }
    Matrix operator * (Matrix other)
    {
        Matrix res;
        for (int i = 1; i <= n; i++)
            for (int k = 1; k <= n; k++)
                for (int j = 1; j <= n; j++)
                    res.b[i][j] += b[i][k] * other.b[k][j];
        return res;
    }
};

Matrix quick_pow(Matrix x, int y)
{
    T res; res.init_I();
    while (y)
    {
        if (y & 1) res = res * x;
        x = x * x; y >>= 1;
    }
    return res;
}
```