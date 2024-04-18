---
linktitle: "Typesetting"
title: "Typesetting"
type: docs

math: true

menu:
    cser0:
        parent: Tools
        weight: 2

view: 3
---

从某种角度来说，计算机领域的大牛多少有一点偏执——毕竟写代码的时候漏打一个符号便可能酿成大祸，所以学计算机的人总有点“强迫症”。退一步来说，即使你没有强迫症，写作文的时候歪七扭八、行间距不一、每个字都不一样大，开头时不时地忘记空两格……总不是件好事。

你也许在书面助教的作业要求中注意到了 LaTeX 这个东西。$\LaTeX$ 是一个高质量的排版系统，你可以通过编写代码生成文档的方式来精确地控制文档里的每一处细节，比如分隔线的粗细、图片的大小，页边距的宽窄等等，精度可以达到毫米。我们不强制要求大家使用 LaTeX 书写作业 (因为对于新手来说这可能会耗费很多的时间)，但我们仍然推荐你学习这项工具 (不出意外，你撰写毕业论文的时候是肯定会用到它的)。

如果你认为 $\LaTeX$ 太过麻烦，不妨尝试一下 Markdown。从原理上来说 Markdown 和 LaTeX 很不一样，但你可以简单地认为它们都是通过一些特殊的符号来生成具有格式，编排美观的文档 (助教博客中的大部分页面都是用 Markdown 书写的)。Markdown 比 LaTeX 简明很多，比如你可以通过 `#` 来生成各个级别的标题：

## 这是二级标题

### 这是三级标题

可以通过 `*` 来生成列表：

* 列表1
* 列表2

Markdown 同样支持丰富的数学符号：

```markdown
\begin{align}
f(n)=\sum_{d|n}g(d)\Longleftrightarrow g(n)=\sum_{d|n}\mu(d)f\left(\frac{n}{d}\right).
\end{align}
```

$$
\begin{align}
f(n)=\sum_{d|n}g(d)\Longleftrightarrow g(n)=\sum_{d|n}\mu(d)f\left(\frac{n}{d}\right).
\end{align}
$$

我们强烈建议你至少掌握一些 Markdown 的基本语法，这可以帮助你将来迅速地写出一份还算美观的文档。