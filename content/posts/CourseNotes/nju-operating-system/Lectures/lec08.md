---
title: "Lecture 08: Concurrent Bugs and Confrontation"
linktitle: '08: 并发bug'
type: docs
date: '2020-12-13T00:00:00Z'
draft: false
featured: false

diagram: true
math: true

menu:
    njuos-lectures:
        parent: Contents
        weight: 8
---

Bug 多的根本原因：软件是需求在计算机世界的投影，人类世界的很多性质在编程语言中丧失了。

## Defensive Programming

一个好的策略是添加很多的 assert()，assert() 的核心意义在于将程序的需求表达出来。

assert() 虽然不一定能写得对，但如果 assert() 和代码出错的概率是独立的，那么写 assert() 就使得程序出错的概率降低了一个数量级。

## Concurrent Bugs

### Deadlock

> **一个中断相关的死锁**
>
> ```c
> void os_run()
> {
>  spin_lock(&lk);
>  spin_lock(&xxx);
>  spin_unlock(&xxx); //------+
> }					   //  	   |
> 					   //	   | Interrupt
> void on_interrupt()	   //	   |
> {					   //	   |
>  spin_lock(&lk); // <-------+
>  ...
>  spin_unlock(&lk);
> }
> ```
>
> 上锁的时候要关中断，且锁出现嵌套的时候，需要一个计数器来维护嵌套层数，只有完全无锁的时候才能开中断 (释放锁的时候不能莽开中断)。

* AA-deadlock：某个线程在试图获得自己已经获得的锁。这很容易检测出来。在编程的时候理应多使用如下的防御性编程：

    ```c
    if (holding(lk)) panic();
    ```

* ABBA-deadlock：进程在互相等待别人的锁。lock-ordering 是一种好的防御方法：系统中所有的锁必须排出一个无环的拓扑序。

    > model checker 可以可视化地帮我们检查是否有死锁。如果某个节点所有的出边都指向自己，这就是一个陷入死锁的状态。

### Data Race

两个线程在同一时间访问同一个地址，且至少有一个是写。之所以被称为“竞争”是因为运行的结果取决于“谁跑赢了”。

### Atomicity Violation (AV) & Order Violation (OV)

TOCTTOU (time of check to time of use)：某段代码通常会检查某个条件 (time to check)，然后在认为这个条件的成立是 invariant 的情况下对其进行某种操作 (time of use)，但如果其他程序在中间插入做了某个改变 invariant 的操作，就会产生 bug。

```
                       TIME		
/home/abc/mailbox       |
a symbolic link?        |
      |                 |   Delete /home/abc/mailbox
      |                 |
      | No              |   Create symbolic link
      |                 |   ~/mailbox, pointing to 
      |                 |   /etc/passwd
Append the new message  |
to /home/abc/mailbox    |
```

在上述过程中，如果攻击者利用 TOC 和 TOU 之间的窗口期将 `/home/abc/mailbox` 指向某个非法的文件 (打破了 kernel 之前检查的条件)，kernel 就可能会给普通用户 root 权限。

## Bugs Confrontation

### Lockdep

我们可以为每把锁一个全局唯一的 allocation site。在所有获得锁/释放锁的时间点记录一份日志，然后对日志进行分析 (观察上锁顺序)，如果对于某两把锁 $x,y$，我们能观测到 $x\rightsquigarrow y\and y\rightsquigarrow x$，则检测出了可能的死锁。

```c
// lock-site.c
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>

typedef struct lock {
  int locked;
  const char *site;
} lock_t;

#define STRINGIFY(s) #s
#define TOSTRING(s)  STRINGIFY(s)
#define LOCK_INIT() \
  ( (lock_t) { .locked = 0, .site = __FILE__ ":" TOSTRING(__LINE__), } )

lock_t lk1 = LOCK_INIT();
lock_t lk2 = LOCK_INIT();

void lock(lock_t *lk) {
  printf("LOCK   %s\n", lk->site);
}

void unlock(lock_t *lk) {
  printf("UNLOCK %s\n", lk->site);
}

struct some_object {
  lock_t lock;
  int data;
};

void object_init(struct some_object *obj) {
  obj->lock = LOCK_INIT();
}

int main() {
  lock(&lk1);
  lock(&lk2);
  unlock(&lk1);
  unlock(&lk2);

  struct some_object *obj = malloc(sizeof(struct some_object));
  assert(obj);
  object_init(obj);
  lock(&obj->lock);

  lock(&lk2);
  lock(&lk1);
  return 0;
}
```

上面的 `lock-site.c` 提供了一种简单的为每个锁提供唯一 allocation site 的方法，其中的一些宏定义值得学习。

### Sanitizer

运行时的动态检查。AddressSanitizer 可以检查内存访问的 bug (监控所有的内存访问)，ThreadSanitizer 可以检查数据竞争 (内存访问 + lock/unlock)，此外还有 UBSanitizer, MemorySanitizer 等。

```c
// uaf.c
#include <stdlib.h>
#include <string.h>

int main() {
  int *ptr = malloc(sizeof(int));
  *ptr = 1;
  free(ptr);
  *ptr = 1;
}
```

上述程序存在 use-after-free 问题。在编译时加入 `-fsanitize=address`，运行程序时，我们可以得到丰富的报错信息。

### Lightweight Tools

#### Stack Canary

在栈顶和栈底设置一些存储了特殊值的内存单元 (金丝雀) 并定期检查这些单元。如果这些单元被修改了，说明发生了栈溢出。我们只需很短的代码就可以完成 stack canary：

```c
#define MAGIC 0x55555555
#define BOTTOM (STK_SZ / sizeof(u32) - 1)
struct stack { char data[STK_SZ]; };

void canary_init(struct stack *s) {
  u32 *ptr = (u32 *)s;
  for (int i = 0; i < CANARY_SZ; i++)
    ptr[BOTTOM - i] = ptr[i] = MAGIC;
}

void canary_check(struct stack *s) {
  u32 *ptr = (u32 *)s;
  for (int i = 0; i < CANARY_SZ; i++) {
    panic_on(ptr[BOTTOM - i] != MAGIC, "underflow");
    panic_on(ptr[i] != MAGIC, "overflow");
  }
}
```

我们有时在 windows 中会看到的 "烫烫烫" 等字，其实是 stack canary 在 GB2312 编码下的结果：

- 未初始化栈: `0xcccccccc`
- 未初始化堆: `0xcdcdcdcd`
- 对象头尾: `0xfdfdfdfd`
- 已回收内存: `0xdddddddd`

我们可以用

```python
(b'\xcc' * 80).decode('gb2312')
```

查看使用未初始化栈时得到的内容 (烫烫烫)。

#### Lightweight Lockdep

很多情况下，我们可以在自旋锁自旋次数过多的时候直接报警，然后利用 GDB 的 backtrace 功能观察是否真的发生了死锁。

```c
int spin_cnt = 0;
while (xchg(&locked, 1)) {
  if (spin_cnt++ > SPIN_LIMIT) {
    printf("Too many spin @ %s:%d\n", __FILE__, __LINE__);
  }
}
```