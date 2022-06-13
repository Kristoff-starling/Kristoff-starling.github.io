---
layout: post
date: 2022-06-01
title: "NJU-OS Lecture 30: Modern Storage System"
categories: ["NJU-22020230 Operating Systems", "Lecture Notes"]
tags: 
mathjax: true
---

## Abilities and Limitations of FS

OJ 最简单的实现方式可以就是一个文件系统。比如 OS2022 课程的学生名单存放在 `/OS2022/students.csv` 中，向 Lab1 提交的代码以一个压缩包的形式存放在 `/OS2022/L1/studentid/xxxxxxxx.tar.bz2` 中。OJ 的前端和后端部署在不同的机器上，后端机器通过 ssh 连接前端，扫描文件系统寻找符合格式的提交拉回后台，评测以后发送一个 `.tar.bz2.result` 文件给前端，前端就可以将评测结果显示出来。<!-- more -->

使用文件系统的最大好处是简单：我们有海量的 UNIX 工具/标准库可以处理文件。比如查询某个同学的提交可以使用如下 python 代码：

```python
for f in wiki.UPLOAD_PATH.glob(
	f'{course}/{module}/{stuid}/{file_pattern}'):
    if not f.name.endswith('.result'):
        # f是一个提交, do something
```

再比如添加了测试数据/更新了测试代码后我们只需要一行 UNIX 命令就可以开启 rejudge：

```bash
find OS2022/L1 -name "*.result" | xargs rm
```

删除所有的 `.result` 结尾的文件后，后端便会自动抓取没有对应 `.result` 文件的提交压缩包评测，从而实现重测。

文件系统的局限在于：

* scalability 不好：任何一个页面的渲染都要遍历所有目录，对于庞大系统来说效率太低；
* 可靠性低：几乎无法抵抗崩溃。比如后端完成评测后会通过 scp 命令将 `.result` 文件传送到前端服务器，但如果传送的过程中网断了，那么文件系统无法做到 all or nothing，从而前端可能会处于一个 inconsistent 的状态， 比如已经创建了 `.result` 文件但由于复制没有完成，文件是空的。这便是评测结果中 "server error" 的一种来源。

## Relational Database

所有的数据都可以以二维表的形式存储在数据库内。Structured Query Language (SQL) 用于描述需求，数据库引擎负责将需求翻译成具体的实现。比如如下的 SQL 语句：

```sql
SELECT *
FROM students, submissions
WHERE students.sid == submissions.sid AND
	  submissions.course == 'OS2022' AND
	  submissions.module == 'L1'
```

它在功能上等价于

```python
for student in students:
    for submission in submissions:
        if students.sid == submissions.sid and submission.course == 'OS2022' and submissions.module == 'L1':
            yield student, submission
```

但要注意的是：SQL 只是一种 high level 的描述需求的语言，它底层的实现并不一定像这段 python 代码一样。比如在这个例子中，用双重循环在两个表单中 join 是非常低效的，数据库会使用数据结构 (hash, B tree etc.) 来优化实现。SQL 将上层需求和底层实现解耦，这样的设计有诸多好处，比如比较容易实现原子性：

```sql
BEGIN WORK;
-- all or nothing
INSERT INTO students VALUES(...);
INSERT INTO students VALUES(...);
INSERT INTO students VALUES(...);
COMMIT;
```

直到 commit 前，数据都不会真正落实。

> **例子：稍大型的 Online Judge**
>
> 数据库需要保证 ACID - Atomicity, Consistency, Isolation, Durability。对于数千个连接的并发事务，数据库要在保证并发正确性的基础上提升查询效率。一个典型的简单应用场景是 Online Judge：
>
> <center><img src="https://kristoff-starling.github.io/img/APIO-OJ.png" alt="APIO-OJ" style="zoom:33%;" /></center>
>
> 比赛中 Online Judge 评测结果的即时性要求很高，因此我们要组建一个小的分布式系统来并行地评测多组数据。上图所示系统的工作原理是：100 个 worker 负责评测，supervisor 负责给 worker 分配任务 (supervisor 会不断 ping 各个 worker 以了解机器实时的状态)，所有的数据都存储在 database 中。supervisor 和所有的 worker 与数据库相连。当 Online Judge 前端接收到一个新的提交后，他会把文件存储到数据库中，supervisor 会指挥一个 worker 把源代码从数据库拉到本地编译，并将可执行文件传回数据库。接着 supervisor 指挥一部分 worker 从数据库中获取可执行文件 (这一瞬间会产生一个很大的带宽)，然后执行对应的测试数据，worker 将执行结果发送给数据库保存。最后前端从数据库中查询所有测试点的运行结果并显示在网页上。
>
> 从这个例子中我们可以看到数据库作为整个系统的存储枢纽，其可靠性，处理大量并发事务的效率等是整个系统能力的关键。

## Distributed System

在新时代，存储系统需要应对海量的实时数据。构造一个 planet-scale 的数据库遭遇了前所未有的挑战。我们通常用 CAP Theorem 来衡量一个数据库的性能：

* Availability (A)：用户能否在短时间内迅速获取需要的数据
* Consistency (C)：系统是否处于一个一致的状态，同步是否正确 (比如先“取关”再“发送pyq”的操作顺序如果在地理上的另一个数据中心反序会造成严重的后果)。
* Partition Tolerance (P)：系统可以忍受怎样规模的延迟。

分布式系统要面对的问题比“并发编程”要更加严峻：多个线程至少有一个共享的内存可以交换数据、实现同步，而分布式系统各个节点之间没有这样的共享资源；此外，分布式系统的假设是任何一个节点都可以在任何一个时刻“突然消失”。

对分布式系统更友好的数据模型是 key-value (可以理解为 C++ 的 `std::map<>`)。LevelDB 是 Google 开发的实现 key-value storage 的库。我们最基本的需求是增加/删除 key-value 对，以及对当前的状态打快照。LevelDB 使用类似于日志的想法，并不是维护一个“平衡树“，而是记录所有操作的内容。为了解决读放大的问题，LevelDB 构建了一个多级的类似于 memory hierarchy 的日志，读操作优先到 Level 0 的 tree 里寻找信息，找不到去下一层，以此类推。