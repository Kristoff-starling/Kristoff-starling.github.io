---
linktitle: "线段树"
title: "线段树"
type: docs
math: true
date: 2023-04-08
draft: true

menu:
    ps-algorithms:
        parent: Contents
        weight: 4
---

“分块”一讲中举了一个中队长把任务分配给小队长的例子。如果我们把视野再放宽一些——小学里还有校大队长 (传说中的“三道杠")——情况就变得更有趣了：如果大队长要统计人数，他应当把任务分配给各个班的中队长，中队长再把任务分配给各个组的小队长；中队长汇总小队长们的信息，再把信息提交给大队长……这个例子比之前要更加复杂一点，因为它不再是扁平化管理，而是形成了等级结构。