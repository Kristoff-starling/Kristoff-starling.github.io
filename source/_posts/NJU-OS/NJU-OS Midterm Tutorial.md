---
layout: post
date: 2022-04-07
title: "NJU-OS Midterm Tutorial"
categories: ["NJU-22020230 Operating Systems", "Material"]
tags: 
mathjax: true
---

## Problems

### 简答题

现有如下代码：

```c
int volatile sum = 0;

void do_sum() {
  int t;
/* 1 */  t = sum;
/* 2 */  t += 1;
/* 3 */  sum = t;

/* 4 */  t = sum;
/* 5 */  t += 1;
/* 6 */  sum = t;

/* 7 */  t = sum;
/* 8 */  t += 1;
/* 9 */  sum = t;
}
```

<!-- more -->

有两个线程 T1 和 T2 均执行 `do_sum` 代码 (共享 sum 变量)。假设线程每次按 1, 2, 3, ... 9 顺序执行一条语句，且读/写会立即在共享内存中生效。例如，若两个线程以如下顺序执行语句，即相当于 T1 执行完后再执行 T2：

```
T1.1 T1.2 T1.3 T1.4 T1.5 T1.6 T1.7 T1.8 T1.9
T2.1 T2.2 T2.3 T2.4 T2.5 T2.6 T2.7 T2.8 T2.9
```

将得到 `sum = 6` (但这显然不是最小值)。你需要给出使 sum 最小化的一个线程执行顺序 (T1 和 T2 可以交错执行)。

### 编程题

假设我们需要对很多 $x(x>0)$ 计算 $f(x)$ 的和。每个 $f(x)$ 都可能会计算较长的时间。你需要在 `thread.h` 和 `thread-sync.h` (Online Judge 会使用此实现) 的基础上实现不少于 4 个 worker 线程的并行处理：

* 所有待处理的  都是 int 类型，从标准输入 (stdin) 读入直到输入结束
* 对每一个整数 ，都调用来自外部的 `long f(int x);` 计算，这个函数是线程安全的
* 你的程序需要输出所有 $f(x)$ 的和 (保证和不超过 long 的范围)，并正常终止
* 你需要尽快处理来自 stdin 的输入 (读入以后统一处理将导致 Time Limit Exceeded)
* 个别的 $f(x)$ 可能会消耗较长的时间，请确保此时在有待计算的  时其他线程不会被闲置

注意：编程时请自觉仅使用手册，不要参考已有的代码或教科书等材料。

## Solutions

### 简答题

本题乍一眼会觉得两个程序完全交错运行会得到最小值 3，但实际上最小值可以达到 2。为了避免繁杂的思考，我们可以直接写一个小程序解决问题：

```c++
#include <bits/stdc++.h>
using namespace std;

int seq[48], ans = 6;

void printseq() {
	printf("ans: %d\n", ans);
	int c1 = 0, c2 = 0;
	for (int i = 1; i <= 18; i++)
		if (seq[i] == 0) c1++, printf("T1.%d ", c1);
		else c2++, printf("T2.%d ", c2);
	puts("");
}

void check() {
	int sum = 0, tid, t[2] = {0, 0}, c[2] = {0, 0}, type;
	for (int i = 1; i <= 18; i++) {
		tid = seq[i]; c[tid]++; type = c[tid] % 3;
		switch (type) {
			case 1: t[tid] = sum; break;
			case 2: t[tid]++;     break;
			case 0: sum = t[tid]; break;
		}
	}
	if (sum < ans) ans = sum, printseq();
}

void dfs(int step,int cnt) {
	if (step == 19) {if (cnt == 9) check(); return;}
	seq[step] = 0; dfs(step + 1, cnt);
	seq[step] = 1; dfs(step + 1, cnt + 1);
}

int main () {
	dfs(1,0); return 0;
}
```

我们只要为两个线程各开一个变量，就可以枚举所有可能的并发行为。一种可以使最后答案达到 2 的并发执行序列为

```
T1.1 T1.2
T2.1
T1.3 T1.4 T1.5 T1.6
T2.2 T2.3
T1.7 T1.8
T2.4 T2.5 T2.6 T2.7 T2.8 T2.9
T1.9
```

事实上，对于 $k$ 个线程加 $n$ 次的程序，只要 $n,k\geq 2$，sum 的最小值都是 2。我们考虑这样一个调度：

* 线程 1 执行第一次 `t = sum;` `t++` ，然后暂停。此时线程 1 的 t 为 1。
* 线程 2 执行到最后一轮。
* 线程 1 执行 `sum = t;`，然后暂停。此时 sum 变回 1。
* 线程 2 执行最后一轮的 `t = sum;` `t++`，然后暂停。此时线程 2 的 t 为 2。
* 让除了线程 2 以外的所有线程都跑完。
* 线程 2 执行最后一轮的 `sum = t;`，所有线程执行完毕。sum 的最终结果为 2。

这题旨在说明：仅仅是非常简单的并发程序，其运行过程也可能超出人类的理解范围。“反直觉“的来源是： Verifying Sequential Consistency (VSC) 问题是一个 NPC 问题。这里所说的 VSC 问题是这样一个问题：

> 现在有若干个线程，每个线程中能看到若干个线程本地的顺序操作。操作分为两类：
>
> * `write(x, val)`：向全局变量 x 中写入数值 val。
> * `read(x) = val`：读全局变量 x 的值，得到的结果是 val。
>
> 我们能否将各个线程的操作排出一个全序，使得所有 read() 操作的结果都是正确的？

该问题是 NP 问题是显然的，我们考虑将 3-SAT 问题规约到它来证明 VSC 问题是 NP-hard 问题：不妨设一个 3-SAT 问题的实例中出现的变量为 $x_1,x_2,\cdots, x_n$，

* 阶段1 (给变量赋值)：我们创建 $2n$ 个线程，它们的第一个操作分别是 `write(x1,0)` `write(x1,1)` `write(x2,0)` `write(x2,1)` ... `write(xn,0)` `write(xn,1)`。
* 阶段2 (确认变量的值并给从句赋值) 对于每个从句，这里以 $C_i=(x\or \lnot y)$ 举例，我们在某一个线程中做两步操作：`read(x)=1` `write(Si,1)`；在另一个线程中做两步操作：`read(y)=0` `write(Si,1)`。
* 阶段3 (收答案)：在一个收答案线程中连做 `read(S1)=1` `read(S2)=1` ... `read(Sn)=1` 。
* 阶段4 (收尾)：阶段 1 的赋值可能会导致阶段 2 有的线程被阻塞注。为了保证每个线程可以结束，我们要在另外的一些线程中对于每个变量 $x_i$ 把 `write(xi,0)` 和 `write(xi,1)` 再做一遍。

此外，我们需要一些同步机制，保证所有线程阶段 $i$ 的操作都做完了再进入阶段 $i+1$。我们可以额外设置一些标志变量表示阶段 $i$ 的某一个操作有没有完成。在下一个阶段开始之前 read 上一个阶段所有的标志变量，确定都是 1 了再开始做。

以上的操作次数显然是多项式。一个 3-SAT 式如果有解，我们可以在阶段1通过线程的调度、覆盖让变量赋上正确的值，阶段2中可以使从句为真的线程继续执行并给所有 $S_i$ 赋值 1。阶段3收答案线程可以运行完毕，阶段4释放阶段2被阻塞的线程。如果线程调度有解，说明收答案线程看到了所有从句被满足，又因为我们有阶段同步机制，所以我们只要查看阶段1结束时各个变量的值，就知道什么样的赋值可以使所有从句满足。

综上，VSC 是一个 NPC 问题。

### 编程题

该问题其实只要将所有的任务交给线程，线程读入时上锁即可。

```c
#include "thread.h"
#include "thread-sync.h"

extern long f(int x);

long sum[5] = {0, 0, 0, 0, 0};
mutex_t lock = MUTEX_INIT();

void calc(int tid) {
	long rt; int x;
	for (;;)
	{
		mutex_lock(&lock);
		if (feof(stdin) || (scanf("%d", &x) != 1))
		{
			mutex_unlock(&lock);
			break;
		}
		mutex_unlock(&lock);
		sum[tid] += f(x);
	}
}

int main () {
	for (int i = 0; i < 4; i++) create(calc);
	join();
	long ans = 0;
	for (int i = 1; i <= 4; i++) ans += sum[i];
	printf("%ld\n", ans);
	return 0;
}
```

事实上，C 标准库是线程安全的，因此读入根本不需要上锁。上面的实现中可以把锁去掉：

```c
void Tworker(int tid) {
    int x;
    while (!feof(stdin) && scanf("%d", &x) == 1)
        sum[tid] += f(x);
}
```

我们也可以用生产者-消费者模型，让 main() 函数负责所有的读入，将数据分发给各个线程。为了让各个线程能够结束，main() 函数读完数据后再向队列中放线程个数个 -1，线程读到负数就 break。

```c
#include "thread.h"
#define MAX 100

extern long f(int x);

long sum[5];

typedef struct _sync_t {
	pthread_mutex_t lock;
	pthread_cond_t cond;
	int buf[MAX], rp, wp;
}sync_t;

sync_t s;

void init_sync(sync_t *sync) {
	pthread_mutex_init(&sync->lock, NULL);
	pthread_cond_init(&sync->cond, NULL);
	sync->rp = sync->wp = 0;
}

int Tconsumer_wait(sync_t *sync) {
	int rt;
	pthread_mutex_lock(&sync->lock);
	while (sync->rp == sync->wp)
		pthread_cond_wait(&sync->cond, &sync->lock);
	rt = sync->buf[sync->rp % MAX];
	sync->rp++;
	pthread_cond_broadcast(&sync->cond);
	pthread_mutex_unlock(&sync->lock);
	return rt;
}

void Tproducer_wait(sync_t *sync, int val) {
	pthread_mutex_lock(&sync->lock);
	while (sync->wp == sync->rp + MAX)
		pthread_cond_wait(&sync->cond, &sync->lock);
	sync->buf[sync->wp % MAX] = val;
	sync->wp++;
	pthread_cond_broadcast(&sync->cond);
	pthread_mutex_unlock(&sync->lock);
}

void Tconsumer(int tid) {
	int x;
	while ((x = Tconsumer_wait(&s)) > 0) sum[tid] += f(x);
}

int main () {
	int x;
	init_sync(&s);
	for (int i = 1; i <= 4; i++)
		create(Tconsumer);
	while (!feof(stdin) && scanf("%d", &x) == 1) Tproducer_wait(&s, x);
	for (int i = 1; i <= 4; i++)
		Tproducer_wait(&s, -1);
	join();
	
	long ans = 0;
	for (int i = 1; i <= 4; i++) ans += sum[i];

	printf("%ld\n", ans); return 0;
}
```

