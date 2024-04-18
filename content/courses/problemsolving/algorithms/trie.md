---
linktitle: "Trie树"
title: "Trie树"
type: docs
math: true
date: 2023-12-25

menu:
    ps-algorithms:
        parent: Contents
        weight: 10
---

字典树 (又称 Trie 树) 是一个可以 $O(length)$ 地插入，$O(length)$ 地查询的 `unordered_set<string>`，下面的图例展示了它的基本工作原理，这个 Trie 树存储的字符串有 to, tea, ted, ten, A, i, in, inn。

<figure>
    <img src="/img/problemsolving/trie.svg" style="zoom: 70%;" >
    <figcaption>
    <font color="grey">该例子中每个字符串还存储了一个对应的数值，用这种方法，Trie也可以实现 unordered_map&lt;string, Any&gt;</font>
    </figcaption>
</figure>

从根节点出发向下走，每一条路径上的字母连接起来代表一个字符串，拥有相同前缀的字符串可以共享前缀节点 (这也是它能够高效查找的核心)。
* 插入字符串时，只需要根据每一位字符选择对应的边向下走 (如果不存在对应符号的边则新建边和顶点)，在最后停留的节点上打一个“存在标记”，时间复杂度为 $O(length)$。
* 查询字符串时，仍然根据每一位字符选择对应的边向下走，字符串存在当且仅当能完整走完且最后停留的节点上确实有“存在标记”，时间复杂度为 $O(length)$。