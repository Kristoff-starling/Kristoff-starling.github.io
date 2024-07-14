---
linktitle: "III-Final"
title: "问题求解III-Final 题解"
type: docs
math: true
draft: true
date: 2024-01-01

menu:
    ps-oj-solutions-2022:
        parent: Contents
        weight: 3200
---

{{% alert note %}}

**Problem A**

* 给定一棵 $n$ 个点的树以及树上的两个点 $x, y$。一个树上的点对 $(u, v)$ 是好的当且仅当从 $u$ 到 $v$ 的路径不会先经过 $x$ 后经过 $y$ (两者不同时经过，或者先经过 $y$ 后经过 $x$)。问有多少个好的点对。
* $n\leq 3\times 10^5$。

{{% /alert %}}