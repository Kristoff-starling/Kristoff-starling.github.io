---
linktitle: "cdq分治"
title: "cdq分治"
type: docs
math: true
date: 2023-05-13

menu:
    ps-algorithms:
        parent: Contents
        weight: 7
---

cdq分治是一种常用的分治思想，最早由 [陈丹琦](https://www.cs.princeton.edu/~danqic/) 整理和总结 ([原文地址](https://www.cs.princeton.edu/~danqic/papers/divide-and-conquer.pdf))。这种思想从 high-level 层面来说十分简单和抽象：当我们希望解决某个问题 `solve(l, r)` 时，我们可以考虑如下的分治步骤：

* 解决 `solve(l, mid)`。
* 考虑 `[l, mid]` 对 `[mid+1, r]` 的贡献。
* 解决 `solve(mid+1, r)`。

第一步和第三步非常简明，但第二步的“贡献”非常抽象，需要具体问题具体分析。这里我们以二维偏序和三维偏序为例示范一些“贡献”的计算方法。

二维偏序问题指对于一个二元组序列 $(a_1, b_1), (a_2, b_2),\cdots, (a_n, b_n)$，求满足 $a_i<a_j$ 且 $b_i<b_j$ 的数对 $(i, j)$ 的个数。显然我们如果将二元组按照 $a_i$ 从小到大排序，那么该问题就转化为了 $b_i$ 序列的逆序数问题。我们尝试套用cdq分治的“模板”来解决它。令 $solve(l, r)$ 表示区间 $[l, r]$ 内的逆序对个数，最终答案显然为 $solve(1, n)$。$solve(l, r)$ 分为三个步骤：

* $solve(l, mid)$：递归解决.
* 计算下标在 $[l, mid]$ 中的数对 $[mid+1, r]$ 中的数的贡献。换言之，我们需要计算满足 $i\in [l, mid], j\in [mid+1, r]， b_i>b_j$ 的 $(i, j)$-pair 个数。
* $solve(mid+1, r)$：递归解决。

重点关注第二条。虽然看上去还是在数“逆序对”，但它比原问题简单了很多——因为 $[l, r]$ 的数被鲜明地分成了 $[l, mid]$ 和 $[mid+1, r]$ 两类。换言之，每个数 $b_i$ 可以被看作 $(0, b_i)$ 或者 $(1, b_i)$，取决于它在左半边还是右半边，我们只关心 0 的那一类 $b_i$ 比 1 的那一类 $b_i$ 大的数目。我们可以通过按照 $b_i$ 做一遍归并排序，然后统计每个 0 前面有多少个 1 的方法来解决。使用主定理分析容易得知该算法的时间复杂度为 $O(n\log n)$。

在进入三维偏序之前，我们先尝试总结 cdq 分治降低问题难度的本质：它通过分治递归处理子问题，然后在当前层面上只考虑左对右的贡献，从而将**某一维度的排序问题转换成了 0/1 的二元问题**。

再来看三维偏序问题。三维偏序问题和二维偏序定义类似，只不过每个元素都换成了三元组 $(a_i, b_i, c_i)$。我们如法泡制，对第一维排序后套用 cdq 分治，相当于将所有的三元组转换成了 $(0/1, b_i, c_i)$。由于还剩下两维，直接下手有点困难 (除非你掌握了某些高级的数据结构)，但我们可以在 cdq 分治上再套一层 cdq 分治：具体来说，我们可以对 $b_i$ 这一维度做归并排序，然后对其 cdq 分治，这样在第三维上每个三元组实际上被转化成了 $(0/1, 0/1, c_i)$，我们只需要考虑所有形如 $(0, 0, c_1)$ 对 $(1, 1, c_2)$ 的贡献即可。总时间复杂度 $O(n\log^2n)$。

{{% alert note %}}
**致歉**

本讲义写的相当糟糕——cdq分治难度较大，思想比较微妙，笔者功力有限，用语言难以描述清楚。我们提供两份额外的参考资料，一是 [OIWiki](https://oi-wiki.org/misc/cdq-divide/) 上对于 cdq 分治的讲解，二是一份 cdq 套 cdq 的三维偏序的参考实现，希望尽可能帮助感兴趣的同学理解。

{{% /alert %}}

<details><summary><b>Code</b> :: <i>Click to expand</i></summary>

```c++
struct node
{
    int x,y,z;
    int nx,ny,nz;
    bool operator == (const node cp)
    {
        return (x==cp.x) && (y==cp.y) && (z==cp.z);
    }
}p[100048],a[100048],b[100048],c[100048];

int n;
int ans = 0;

bool cmp(node x,node y)
{
    if (x.x!=y.x) return x.x<y.x;
    if (x.y!=y.y) return x.y<y.y;
    return x.z<y.z;
}

void cdq2(int left,int right)
{
    if (left==right) return;
    int i,k1,k2,pt,mid=(left+right)>>1;
    cdq2(left,mid);cdq2(mid+1,right);
    int cnt=0;
    for (k1=left,k2=mid+1,pt=left;pt<=right;pt++)
    {
        if (k2>right || (k1<=mid && k2<=right && b[k1].z<=b[k2].z))
        {
            if (!b[k1].nx) cnt++;
            c[pt]=b[k1++];
        }
        else
        {
            if (b[k2].nx) ans += cnt;
            c[pt]=b[k2++];
        }
    }
    for (i=left;i<=right;i++) b[i] = c[i];
}

void cdq1(int left,int right)
{
    if (left==right) return;
    int i,k1,k2,pt,mid=(left+right)>>1;
    cdq1(left,mid); cdq1(mid+1,right);
    for (k1=left,k2=mid+1,pt=left;pt<=right;pt++)
    {
        if (k2>right || (k1<=mid && k2<=right && a[k1].y<=a[k2].y))
        {
            b[pt]=a[k1++];
            b[pt].nx=0;
        }
        else
        {
            b[pt]=a[k2++];
            b[pt].nx=1;
        }
    }
    for (i=left;i<=right;i++) a[i] = b[i];
    cdq2(left,right);
}

int main ()
{
    int i;
    n=getint();i=getint();
    for (i=1;i<=n;i++) {p[i].x=getint();p[i].y=getint();p[i].z=getint();}
    sort(p+1,p+n+1,cmp);
    for (i=1;i<=n;i++) a[i] = p[i];
    cdq1(1, n);
}
```

</details>