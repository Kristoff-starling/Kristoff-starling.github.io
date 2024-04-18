---
linktitle: "LCA-倍增"
title: "最近公共祖先-倍增"
type: docs
math: true
diagram: true
date: 2023-04-21

menu:
    ps-algorithms:
        parent: Contents
        weight: 5
---

在计算机世界中，所有与2的次幂相关的事情总是充满魔力的——即便是很大的数，对2取对数后也会落入我们容易处理的范围。之前介绍过的快速幂算法其实就是倍增思想的一种运用。这里我们以计算树上最近公共祖先 (lowest common ancestor, LCA) 为例再次展示倍增思想的强大。

{{% alert note %}}

**树**

如果你对“树”一无所知，你可以参考 [维基百科](https://en.wikipedia.org/wiki/Tree_(data_structure)) 中的解释。这里强调一些简明的入门概念。

* 树是一个有 $n$ 个顶点和 $n-1$ 条边构成的连通图 (连通指整个图只有“一块”，即任意两点之间都存在路径可达)。容易发现，树中是不会有环的。
* 如果选择一个节点作为树根 (root)，那么整棵树会形成一个层次结构。树上的每个节点到根有且仅有一条路径，这个路径的长度称为节点的深度。
* 在有根树中，每个节点“上面”相邻的只有一个节点，称为该节点的父亲。每个节点“下面”相邻的有一堆节点 (也可能没有)，称为该节点的孩子。一个节点A的父亲，父亲的父亲，…… 一直向上到根这条链上所有的节点都是A的祖先。

{{% /alert %}}

对于树中的两个节点 $u, v$，$LCA(u, v)$ 指的是 $u$ 和 $v$ 的所有公共祖先中最深的那个 (也可以说是离 $u, v$ 最近的那个)。下面是一个例子:

<img src="/img/LCA.png" style="zoom: 30%;" />

暴力地求解LCA不算困难，总体思想是：我们先让深度大的节点往上爬，爬到和另一个节点相同深度，然后让 $u$ 和 $v$ 一直向上爬，直到它们相遇。下面的代码非常易懂

```c++
int LCA_bruteforce(int u, int v)
{
    if (depth[u] < depth[v]) swap(u, v);
    while (depth[u] > depth[v])
        u = father[u];
    while (u != v)
    {
        u = father[u];
        v = father[v];
    }
    return u;
}
```

该算法的问题在于：如果树的深度很大 (例如达到了和 $n$ 同阶)，那么每次求解两个节点的 LCA 都需要 $O(n)$ 的时间。如果我们需要多次求解多个点对的 LCA (例如 $q$ 次)，就需要 $O(qn)$ 的时间。在 $q$ 较大的情况下这不可接受。

接下来我们向大家展示如何利用倍增思想优化 LCA 的求解：

令 $anc(u,i)$ 表示节点 $u$ 向上爬 $2^i$ 步之后到达的节点编号，如果 $depth(u)<2^i$ 则 $anc(u,i)=0$。我们发现 $anc(u,i)$ 是容易计算的：

$$
anc(u, i)=
\begin{cases}
father(u)&, i=0\\\\
anc(anc(u, i-1), i-1)&, i \geq 1
\end{cases}
$$

简单来说，向上爬 $2^n$ 步的结果等于先向上爬 $2^{n-1}$ 步，再向上爬 $2^{n-1}$ 步的结果。如果我们按照 $i$ 从小到大的顺序计算所有节点的 $anc(u,i)$，那么可以递推地完成计算过程。在实际实现时我们通常树搜索的过程中完成 anc 数组的计算，详见最后的参考代码。

有了 anc 数组后，“向上跳”的流程就可以被大幅加速。我们先假设 $u, v$ 深度相同，这时我们不需要每次向上爬一格，而可以用 anc “赌一把大的”：
```c++
for (int i = 20; i >= 0; i--)
    if (anc[u][i] != anc[v][i])
    {
        u = anc[u][i];
        v = anc[v][i];
    }
```
这里巧妙地利用了整数二进制拆分的唯一性：假设 $depth(u)-depth(LCA(u, v))=d$，且
$$
d-1 = 2^{a_1} + 2^{a_2} + \cdots + 2^{a_k}, a_1>a_2>\cdots>a_k
$$
那么上述循环正好会在 $i=a_1, a_2,\cdots, a_k$ 的地方“向上跳”。之所以是 $d-1$ 而不是 $d$ 是因为我们要求 `anc[u][i] != anc[v][i]`，只有这样我们才能确保没有“跳过头”，因此上述循环结束后 $u, v$ 都会正好在 LCA 的下面 (孩子)。我们强烈建议你手画一个例子体会这个过程。

还剩下一个问题：如果 $u, v$ 深度不同该怎么办。和暴力做法的思路一样，我们可以让深度大的节点向上爬，爬到和另一个节点同深度。不过在 anc 数组的加持下，我们不再需要一个一个地爬了：
```c++
// 假设 depth[u] >= depth[v]
for (int i = 20; i >= 0; i--)
    if (depth[anc[u][i]] >= depth[v])
        u = anc[u][i];
```
你仍然可以用整数拆分的方式证明：$u$ 只会在 $depth(u)-depth(v)$ 的二进制表示中为 1 的那些位置向上跳，且循环结束后 $u$ 会和 $v$ 同深度。

我们来分析这个倍增做法的复杂度：它尝试用 $2^k, 2^{k-1},\cdots, 2^1, 2^0$ 去覆盖 $u, v$ 到 LCA 的深度差距，因此只需要 $O(\log n)$ 的时间即可完成一次查询。虽然预处理 anc 表需要 $O(n\log n)$ 的时间，但在查询次数较多的情况下，$O(n\log n + q\log n)$ 就会比 $O(n + qn)$ 更有优势。

{{% alert note %}}

**通用思想**

从更抽象的层面来说，倍增思想成功的关键很多时候是一种“单一的扩展可能”：

* 在快速幂的例子中，$*2$ 这件事非常固定，这使得 $2^n$ 可以通过重复 $*2$ 得到；
* 在 LCA 的例子中，每个节点的父亲只有一个，这使得往上 $n$ 层的祖先可以通过重复 `u=father[u]` 得到；
* ……

这段话看起来有点玄学，但如果你接触了更多可以通过倍增思想解决的问题结构，回过头看可能会对此有更深的理解。

{{% /alert %}}

{{% alert tip %}}

**更快地求解LCA?**

虽然 $O(\log n)$ 的效率已经足够令人满意，但事实上在充分预处理的情况下，我们可以 $O(1)$ 地完成一对点的 LCA 查询。如果你对此感兴趣，可以尝试搜索 ST 表、dfs 序等关键词。我们会在合适的时机向大家展示这种技术。
{{% /alert %}}

以下是一份参考代码：

<details><summary><b>LCA</b><i>::click to expand</i></summary>

```c++
const int MAXN = 2e5 + 10;
vector<int> v[MAXN];   // vector 存储了每个节点相邻点的编号
int depth[MAXN];       // depth 存储了每个节点的深度
int anc[MAXN][21];     // 2 ^ 21 > MAXN

void dfs(int x, int fa)
{
    // 搜索到 x 时 x 的所有祖先都已经被访问过，anc 数组已被计算
    // 因此现在就可以计算 x 的 anc 数组
    anc[x][0] = fa;
    for (int i = 1; i <= 20; i++)
        anc[x][i] = anc[anc[x][i - 1]][i - 1];
    // 搜索
    for (int y : v[x])
        if (y != fa)   // 相邻的节点不是父亲，那就是孩子，向下搜索
        {
            depth[y] = depth[x] + 1;
            dfs(y, x); // y 是 x 的孩子，x 是 y 的父亲
        }
}

int query_lca(int x, int y)
{
    if (depth[x] < depth[y]) swap(x, y);
    for (int i = 20; i >= 0; i--)
        if (depth[anc[x][i]] >= depth[y])
            x = anc[x][i];
    // 此时有 depth[x] = depth[y]
    if (x == y) return x;
    for (int i = 20; i >= 0; i--)
        if (anc[x][i] != anc[y][i])
        {
            x = anc[x][i];
            y = anc[y][i];
        }
    // 注意不等号条件，此时 x, y 一定都是 LCA 的孩子, LCA = anc[x][0] = anc[y][0]
    return anc[x][0];
}
```

</details>