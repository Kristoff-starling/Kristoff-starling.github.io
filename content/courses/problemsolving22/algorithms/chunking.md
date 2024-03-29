---
linktitle: "分块"
title: "分块——优雅的暴力"
type: docs
math: true
date: 2023-04-08

menu:
    ps-algorithms:
        parent: Contents
        weight: 3
---

分块的思想被应用在生活的方方面面：举一个最简单的例子，大家上小学的时候通常班级里的座位被分成若干个“组”。每组有一个小组长 (传说中的“一道杠”)，班里有班干部 (传说中的“二道杠")。班长在统计人数的时候，最便捷的方法就是让各个小组长统计自己组内的人数，汇总到班长手中再做一次简单加法。

在算法设计中，分块的思想也常常被使用。空说无益，我们通过一个比 OJ 练习题更复杂的问题来解释分块思想的运用：

> 给定数列 $a_1, \cdots, a_n$，高效实现 $q$ 次以下两种操作：
> * `1 l r x`：给区间 $[l, r]$ 中的数统一加上 $x$。
> * `2 l r`：查询 $\sum_{k=l}^r a_k$ 的值。

暴力修改和统计有着相同的问题：如果给定的区间很长，那么操作执行就相当缓慢。总时间复杂度可以达到 $O(qn)$。

我们来考虑如下的一种分块策略：将数列分成若干个长度为 $m$ 的小段 (第一段是 $[0,m)$，第二段是 $[m, 2m)$，依次类推，我们假设 $n$ 可以被 $m$ 整除)，命名为 $B_1, B_2, \cdots, B_b$。我们为每个块分配一个累加标记 $t_i$，表示这个块上的数被**整体**加了多少，再维护一个和标记 $s_i$，表示在不考虑累加标记的情况下块内的数的和。那么
* 对于给 $[l, r]$ 上的数加 $x$ 这件事，我们可以把 $[l,r]$ 这个大区间拆分为：

  * $[l, l_1]$，开头的一段，是某个块的后缀
  * $[l_1+1, r_1]$，这一段正好对应若干个完整的块。
  * $[r_1+1, r]$，结尾的零碎一段，是某个块的前缀。
  
  <br>
  对于 $[l, l_1]$ 和 $[r_1, r]$，我们一个一个地给每个数 +x (别忘了同步更新该块对应的 $s_i$)。对于中间的若干个完整的块，我们直接把 +x 标记打在对应的 $t_i$ 上 (即 "ti += x")。
---
* 对于求 $\sum_{k=1}^ra_k$，我们仍然可以把区间拆成上述的三个部分。对于开头和结尾的零碎区间，我们一个一个地累加，对于中间的若干个完整的块，我们发现一个块的实际的和可以通过如下方式算出：
    $$
    sum_i = s_i + t_i * m
    $$
    将中间完整块的 $sum_i$ 累加起来即可。

为了更清楚地说明上述思想，我们给出修改和查询操作的简单伪代码：

<details><summary>Pseudocode :: Click to expand</summary>

```c++
void modify(int l, int r, int x)
{
    decompose [l, r] into [l, l1], [l1, r1] and [r1, r]
    suppose [l1, r1] = Bp, Bp+1, ..., Bq
    // 开头零碎段
    for (int i = l; i <= l1; i++)
    {
        a[i] += x;
        s[p-1] += x;
    }
    // 中间完整段
    for (int i = p; i <= q; i++)
        t[i] += x;
    // 结尾零碎段
    for (int i = r1 + 1; i <= r; i++)
    {
        a[i] += x;
        s[q+1] += x;
    }
}

int query(int l, int r)
{
    decompose [l, r] into [l, l1], [l1, r1] and [r1, r]
    suppose [l1, r1] = Bp, Bi+1, ..., Bq

    int sum = 0;
    // 开头零碎段
    for (int i = l; i <= l1; i++)
        sum += a[i];
    // 中间完整段 
    for (int i = p; i <= q; i++)
        sum += s[i] + t[i] * m;
    // 结尾零碎段 
    for (int i = r1 + 1; i <= r; i++)
        sum += a[i];
    
    return sum
}
```

</details>
<br>

接下来我们考虑这个块的长度应当如何设计。我们容易发现块如果太大，那么一个修改/查询区间首尾的零碎部分就可能太长；如果块太小，那么一个修改/查询区间就可能包含太多的块。具体的，在块大小为 $m$ 时，整个数列被划分成 $n/m$ 个块，那么在一次操作中，首尾的零碎部分的求和复杂度可以达到 $2m = O(m)$，累加区间中间包含的整块的信息的时间复杂度最多可以达到 $O(n/m)$ ，于是有
$$
\arg\min_m \left\\{O(m), O(n/m)\right\\} \Longrightarrow m=\sqrt n
$$
所以，将块的大小定义在 $\sqrt n$ 附近是最合理的选择，这也是分块思想通常被称为“根号暴力”的原因。可以算出，在我们的分块算法下，解决原问题的复杂度被降低到了 $O(q\sqrt n)$。

数列上分块是分块思想最常见的应用，但分块的应用不仅停留在这种线性结构上。从抽象层面说，如果你发现你可以将某一些东西绑成一个整体，然后将某些操作统一在整体层面，那么便可以考虑分块。分块被称为是一种优雅的暴力，因为它没有复杂的数据结构设计，看上去就是一些加加减减，便举重若轻地解决了问题。