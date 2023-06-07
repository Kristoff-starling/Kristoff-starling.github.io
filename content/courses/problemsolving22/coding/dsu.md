---
linktitle: "并查集"
title: "并查集"
type: docs
math: true
date: 2023-06-07

menu:
    ps-coding:
        parent: Contents
        weight: 8
---

《算法导论》中提到基于路径压缩和按秩合并的并查集更新和查询的均摊时间复杂度为 $O(\alpha(n))$，其中 $\alpha$ 为反阿克曼函数。实践中大家通常写只带有路径压缩的并查集，因为按秩合并写起来更加麻烦 (虽然只需要几行)。我们承认只带有路径压缩的并查集时间复杂度会退化为 $O(\log n)$[^1]，但这样的数据不太容易构造，且 $\log n$ 其实也相当好了。然而，对于新手来说，路径压缩可能也相当难实现，因此本讲义向大家展示并查集的实现技巧。

并查集和树状数组是许多程序员心爱的数据结构，因为它们思维巧妙，功能强大，且优秀的实现极其简洁。这里我们给出一份完整的带有路径压缩和按秩合并的并查集模板：

```c++
namespace DSU
{
    int pre[MAXN], rnk[MAXN];
    void init()
    {
        for (int i = 1; i <= n; i++)
            pre[i] = i, rnk[i] = 1;
    }
    int find_anc(int x)
    {
        if (pre[x] != x) pre[x] = find_anc(pre[x]);
        return pre[x];
    }
    void merge(int x, int y)
    {
        x = find_anc(x); y = find_anc(y);
        if (rnk[x] > rnk[y])
            pre[y] = x;
        else
        {
            if (rnk[x] == rnk[y]) rnk[y]++;
            pre[x] = y;
        }
    }
}
```

你需要重点关注的是 `find_anc()` 函数，它只用一行就在查询集合代表元的同时完成了路径压缩，本质上是在找到代表元之后将路径上所有的节点直接挂到根下。

如果你觉得麻烦，可以省略按秩合并的部分 (但要付出一点点复杂度的代价，通常可以接受)。还有一些程序员会选择用启发式合并代替按秩合并 (即维护每个集合的元素个数，每次将小的集合合并到大的集合当中)，可以证明路径压缩+启发式合并也可以做到 $O(\alpha(n))$[^1]。

[^1]: 如果你感兴趣，可以参考 [这篇博文](https://www.luogu.com.cn/blog/Atalod/shi-jian-fu-za-du-shi-neng-fen-xi-qian-tan) 详细了解。