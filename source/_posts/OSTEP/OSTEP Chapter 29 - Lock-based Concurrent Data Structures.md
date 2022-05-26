---
layout: post
date: 2022-02-23
title: "OSTEP Chapter 29: Lock-based Concurrent Data Structures"
categories: ["WISC-CS537 Introduction to Operating Systems", "Book Notes"]
tags: 
mathjax: true
---

## 29.1 Concurrent Counters

一个最简单的 thread-safe 的计数器实现如下 (用锁保护每次读写)：

<!-- more -->

```c
typedef struct __counter_t {
    int value;
    pthread_mutex_t lock;
} counter_t;
void init(counter_t *c) {
    c->value = 0;
    Pthread_mutex_init(&c->lock, NULL);
}
void increment(counter_t *c) {
    Pthread_mutex_lock(&c->lock);
    c->value++;
    Pthread_mutex_unlock(&c->lock);
}
void decrement(counter_t *c) {
    Pthread_mutex_lock(&c->lock);
    c->value--;
    Pthread_mutex_unlock(&c->lock);
}
int get(counter_t *c) {
    Pthread_mutex_lock(&c->lock);
    int rc = c->value;
    Pthread_mutex_unlock(&c->lock);
    return rc;
}
```

但这种实现的 scalability 很差：如果给 $n$ 个核分配等量的任务，最终所消耗的时间几乎是单个任务时长的 $n$ 倍。大部分时间 CPU 核之间在互相等锁，我们没有利用多处理器达到并行的效率。

如果愿意牺牲一部分精确性，我们可以设计一种 approximate counter 来提升效率。approximate counter 中，每个 CPU 核有一个单独的 counter，此外还有一个全局的 counter。全局和本地的 counter 都有锁保护。修改操作中 (假设只有 increment)，CPU 核直接修改本地的 counter，因此各个核可以并行。当本地 counter 的值达到一个阈值时，本地 counter 会和全局 counter 做一次同步，将本地值加到全局值上并将本地值清零。查询操作中直接返回全局 counter 的值即可。

```c
typedef struct __counter_t {
    int             global;
    pthread_mutex_t glock;
    int             local[NUMCPUS];
    pthread_mutex_t llock[NUMCPUS];
    int             threshold;
}counter_t;
void init(counter_t *c, int threshold) {
    c->threshold = threshold;
    c->global = 0;
    Pthread_mutex_init(&c->glock, NULL);
    for (int i = 0; i < NUMCPUS; ++i) {
        c->local[i] = 0;
        Pthread_mutex_init(&c->llock[i], NULL);
    }
}
void update(counter_t *c, int threadID, int amt) {
    int cpu = threadID % NUMCPUS;
    Pthread_mutex_lock(&c->llock[cpu]);
    c->local[cpu] += amt;
    if (c->local[cpu] >= c->threshold) {
        Pthread_mutex_lock(&c->glock);
        c->glocal += c->local[cpu];
        Pthread_mutex_unlock(&c->glock);
        c->local[cpu] = 0;
    }
    Pthread_mutex_unlock(&c->llock[cpu]);
}
int get(counter_t *c)
{
    Pthread_mutex_lock(&c->glock);
    int rt = c->global;
    Pthread_mutex_unlock(&c->glock);
    return rt;
}
```

在该实现版本中，每个线程并不会去确认自己所处的核，而是直接随机 (进程号取模) 分配一个核对应的本地计数器。这样做和之前描述的算法无异。

上述算法中，如果阈值是 0 则与精确的计数器无异。随着阈值的提高，计数器的并行效率会越来越高 (大致呈反比例函数)，但返回的结果会越来越不精确。

## 29.2 Concurrent Linked List

一个朴素的做法是用一把大锁保护整个链表的修改和查询。一种很容易出错的情形是：如果函数中间有多处 return，则每处 return 前都要记得释放锁。为了避免这种 bug，书写代码时可以考虑多用 break 替代 return 减少 return 分支数，或者另写一个 wrapper 调用真正的函数，在 wrapper 中维护锁。

一种增加链表访问并行度的方案是所谓的 hand-over-hand locking (lock coupling)。在查询链表中是否有某个元素时，我们平常的做法是用一个大锁保护整个链表，然后依次扫描链表节点。hand-over-hand locking 的思想是为每个链表节点创建一把锁，每次准备访问下一个节点时，先获得下一个节点的锁，再释放本节点的锁。这样多个线程就可以并行地查询链表。

这个方案在理论上可行，但实际操作中，频繁地获得和释放锁会带来昂贵的代价。一个更可行的方案是每 $n$ 个节点用一把锁保护，准备进入下一个“区域”时做一次 overhead acquiring and releasing。

## 29.3 Concurrent Queues

我们可以在队头和队尾分别维护一把锁，这样从队头取元素和向队尾添加元素的操作可以并行地执行：

```c
typedef struct __node_t {
    int              val;
    struct __node_t *next;
}node_t;
typedef struct __queue_t {
    node_t          *head;
    node_t          *tail;
    pthread_mutex_t  headlock;
    pthread_mutex_t  taillock;
}queue_t;
void queue_init(queue_t *q) {
    node *tmp = malloc(sizeof(node_t));
    tmp->next = NULL;
    q->head = q->tail = tmp;
    pthread_mutex_init(&q->headlock, NULL);
    pthread_mutex_init(&q->taillock, NULL);
}
int queue_enqueue(queue_t *q, int val) {
    node *tmp = malloc(sizeof(node_t));
    if (tmp == NULL) return -1;
    tmp->val = val; tmp->next = NULL;
    pthread_mutex_lock(&q->taillock);
    q->tail->next = tmp;
    q->tail = tmp;
    pthread_mutex_unlock(&q->taillock);
}
int queue_dequeue(queue_t *q, int *val) {
    pthread_mutex_lock(&q->headlock);
    node *tmp = q->head;
    node *newhead = tmp->next;
    if (newhead == NULL) {
        pthread_mutex_unlock(&q->headlock);
        return -1; // queue was empty
    }
    *val = tmp->val;
    q->head = newhead;
    pthread_mutex_unlock(&q->headlock);
    free(tmp);
    return 0;
}
```

这里的一个重要的技巧是：我们在初始化队列时创建了一个 dummy node，并让 `q->head` 和 `q->tail` 都指向它。这个 dummy node 是永远不会出队的。这样我们避免了队列在形状上完全为空的情况，从而保证 headlock 和 taillock 做的工作永远是没有交叉的。

## 29.4 Concurrent Hash Table

一个支持并发访问的 Hash table 实现很简单：使用多个之前提到的链表即可：

```c
#define BUCKETS (101)

typedef struct __hash_t {
    list_t lists[BUCKETS];
}hash_t;

void Hash_Init(hash_t *H) {
    for (int i = 0; i < BUCKETS; i++)
        List_Init(&H->lists[i]);
}

int Hash_Insert(hash_t *H, int key) {
    int bucket = key % BUCKETS;
    return List_Insert(&H->lists[bucket], key);
}

int Hash_Lookup(hash_t *H, int key) {
    int bucket = key % BUCKETS;
    return List_Lookup(&H->lists[bucket], key);
}
```

这个实现在实际中效率不低的原因是：我们并没有用一把大锁保护整个哈希表，而是对每个链表单独用一把所保护。多个线程同时访问哈希表的一个 bucket 的几率较低，这使得 lock contention 发生次数较少。

> **Tip: Avoid Premature Optimization**
>
> "Premature Optimization is the root of all evil." - Knuth
>
> 很多操作系统内核开发时，都是先使用一个大内核锁 (big kernel lock, BKL)，先保证正确性，再考虑如何把锁拆开提升效率。

## 29.5 Summary

略。
