---
layout: post
date: 2022-02-22
title: "OSTEP Chapter 30: Condition Variables"
categories: ["WISC-CS537 Introduction to Operating Systems", "Book Notes"]
tags: 
mathjax: true
---

很多时候，我们有一个任务等待另一个任务的需求，比如主线程等到子线程结束后再继续执行 (这个函数通常被称为 join() )。一个朴素的想法是利用一个共享变量实现：<!-- more -->

```c
volatile int done = 0;

void *child(void *arg) {
    do_something();
    done = 1;
    return NULL;
}

int main () {
    pthread_t c;
    Pthread_create(&c, NULL, child, NULL);
    while (done == 0)
        ;
    do_something_else();
    return 0;
}
```

但这样做主线程会在 done 变量上不停地自旋，占 CPU 不干活。我们希望有更高效的实现方法。

## 30.1 Definition and Routines

为了让一个线程等待一个条件成立，我们通常使用条件变量 (condition variable)。条件变量可以理解为一个显式的队列，线程可以将自己加入到这个队列中挂起，当其他线程的操作使得条件变化时，它会唤醒这个“队列”中的线程。我们提供两个 API：`wait()` 可以让线程将自己加入到队列中等待；`signal()` 可以让别的线程改变了条件状态时向队列中的线程“发送信号”。

POSIX 提供的条件变量 API 为

```c
int pthread_cond_wait(pthread_cont_t *c, pthread_mutex_t *m);
int pthread_cond_signal(pthread_cond_t *c);
```

注意到 wait() 的参数中除了条件变量 c 还有一个自旋锁 m。wait() 的语义要求调用时线程必须拥有自旋锁 m，wait() 会负责释放这个自旋锁并使当前线程进入睡眠状态 (这两步是原子的)；当 signal() 使当前线程被唤醒后，wait() 会负责让线程重新获得自旋锁 m，然后返回。

[join.c](https://github.com/Kristoff-starling/OSTEP/blob/master/bookcode/Chapter%2030%20-%20threads_cv/join.c) 提供了一个主线程等待子线程结束的正确实现。事实上这个过程是很容易实现错的，比如如下一些写法：

```c
void thr_exit() {
    Pthread_mutex_lock(&m);
    Pthread_cond_signal(&c);
    Pthread_mutex_unlock(&m);
}
void thr_join() {
    Pthread_mutex_lock(&m);
    Pthread_cond_wait(&c, &m);
    Pthread_mutex_unlock(&m);
}
```

这个写法只有在 thr_join() 在 thr_exit() 之前执行的情况下才是正确的。如果颠倒了顺序，thr_join() 将永远不会被唤醒 ([join_no_state_var.c](https://github.com/Kristoff-starling/OSTEP/blob/master/bookcode/Chapter%2030%20-%20threads_cv/join_no_state_var.c) 中，主线程里使用了 sleep() 以精确复现这一情形)。因此我们可以看到，使用条件变量时理应有一个条件状态的判断。

```c
void thr_exit() {
    done = 1;
    Pthread_cond_signal(&c);
}
void thr_join() {
    while (done == 0)
        Pthread_cond_wait(&c);
}
```

这个写法没有用锁保护状态变量 done 的读写，缺失了原子性。这里的原子性缺失指的是：虽然在 while 判断时 `done == 0`，但在 wait() 时 done 可能已经不等于 0 了。考虑这样一种执行流：thr_join() 做完 while 判断进入循环，还没来得及 wait() 时，执行流被打断，thr_exit() 执行并 signal()，但此时队列中并没有线程。接着 thr_join() 继续执行陷入睡眠，那么它将永远不会被唤醒 ([join_no_lock()](https://github.com/Kristoff-starling/OSTEP/blob/master/bookcode/Chapter%2030%20-%20threads_cv/join_no_lock.c) 中在主线程使用了比子线程时间更长的 sleep() 以精确复现这一情况)。

> **Tip: Always Hold The Lock While Signaling**
>
> POSIX 的 wait() 函数的语义已经要求了我们在调用 wait() 时必须要拥有自旋锁；虽然少数情况下我们可以无锁地调用 signal()，但为了安全和简单，我们应当在 signal() 时也用自旋锁保护。

## 30.2 The Producer/Consumer (Bounded Buffer) Problem

生产者-消费者问题在系统中很常见，比如下面的例子：

```bash
grep foo file.txt | wc -l
```

grep 找到的内容通过管道传送给 wc 统计个数。管道在内核中就是一个缓冲区，因此这里 grep 是一个生产者，wc 是一个消费者，wc 不能在 buffer 为空的时候从管道里读东西，grep 也不能在缓冲区满了的时候再往里填。

### A Broken Solution

一个比较自然但却错误的实现如下所示：

```c
void *producer(void *arg) {
    for (int i = 0; i < LOOP; i++) {
        lock(&lk);
        if (count == 1) wait(&cond, &lk);
        put(i); // count becomes 1
        signal(&cond);
        unlock(&lk);
    }
}
void *consumer(void *arg) {
    while (1) {
        lock(&lk);
        if (count == 0) wait(&cond, &lk);
        int rt = get(); // count becomes 0
        signal(&cond);
        unlock(&lk);
        printf("%d\n", rt);
    }
}
```

如果只有一个生产者线程和一个消费者线程，这个实现是正确的。但如果有多个，比如一个生产者和两个消费者，就会有并发 bug。考虑如下执行过程：

* $T_{c_1}$ 运行，发现 `count == 0`，挂起；
* $T_p$ 运行，往 buffer 里放了一个数，然后通过 signal() 唤醒 $T_{c_1}$。
* $T_{c_1}$ 刚被唤醒，还没来得及获得自旋锁返回的时候，$T_{c_2}$ 运行，从 buffer 里取走了这个数打印；
* 这时返回到 $T_{c_1}$，获得自旋锁返回后，由于之前的判断是 if 语句，它无法发现 count 此时又变成 0 了，于时再次从 buffer 中取数，触发 assertion fail。

并发 bug 产生的原因是：从消费者线程被唤醒到消费者线程真正获得自旋锁开始工作这段时间内，buffer 的状态改变了。**signal() 的语义只是通知一个正在等待的线程：世界的状态改变了，但它不保证世界的状态被改变成了调用 signal() 之前那一瞬的状态。** 这种 signal() 的语义被称为 **Mesa semantics**。与之相对的是 **Hoare Semantics**，它保证调用 signal() 之后一个线程被唤醒并立即被执行 (原子性)。但后者难实现的多，现在绝大部分系统的 signal() 使用的都是 Mesa semantics。

### Better, But Still Broken: While, Not If

Mesa semantics 是比较容易克服的：我们只要将上面程序中对状态变量做判断的 if 换成 while 即可。这样即使被唤醒后状态又被改变，线程获得自旋锁后会再次检查状态，发现不对后可以再次进入睡眠。

> **Always use while loops when working with condition variables!**

但即使将 if 改成 while，我们的程序仍然有并发 bug。考虑如下执行过程：

* $T_{c_1}$ 运行，发现 `count == 0`，挂起；
* $T_{c_2}$ 运行，发现 `count == 0`，挂起；
* $T_p$ 运行，往 buffer 里添加了一个数字，然后通过 signal() 唤醒了 $T_{c_1}$。唤醒后 $T_{c_1}$ 并未立刻执行，而是 $T_p$ 继续运行，第二次循环时 $T_p$ 发现 `count == 1`，挂起；
* $T_{c_1}$ 运行，从 buffer 里读取了一个数字 (count 变为 0) 然后通过 signal() 唤醒一个线程。注意此时睡眠在条件变量上的线程有 $T_p$ 和 $T_{c_2}$ 两个。$T_{c_1}$ 选择唤醒 $T_{c_2}$。
* $T_{c_1}$ 和 $T_{c_2}$ 都发现没有数据可读，挂起。

到这里，三个线程全部陷入睡眠，程序停滞。从这个例子中我们可以看出：消费者线程消费完后应该唤醒生产者线程；生产者线程生产完后应该唤醒消费者线程，我们应该有两个条件变量。

### The Correct Producer/Consumer Solution

pc.c 提供了一个完整、正确的生产者-消费者实现，其中的要点如下：

* 使用两个条件变量，保证消费者只能唤醒生产者，生产者只能唤醒消费者。
* 判断状态变量使用 while 语句。
* 使用一个循环数组作为 buffer，count 的上限不再是 1 而是一个指定的 max，这使得 buffer 有了一定的容量。
* 这是一个单生产者-多消费者的程序，生产者最后为每个消费者准备了一个 -1，以保证消费者线程能够全部退出。

> **Tip: Use While (Not If) for Conditions**
>
> 使用 while 总是对的，在少数场合下使用 if 也可以达到目的，但为了安全性最好通通使用 while。
>
> 我们推荐使用 while 而不是 if 的另一个原因是：一些实现的有问题的 signal() 可能会唤醒不止一个等待的线程。这种情况下，while 仍然能帮我们保证正确。

## 30.3 Covering Conditions

假设我们希望用条件变量实现一个内存分配器，alloc() 时如果剩余内存不足则挂起，一段伪代码如下：

```c
int bytesLeft = MAX_HEAP_SIZE;
void *allocate(int size) {
    lock(&lk);
    while (bytesLeft < size)
        wait(&cond, &lk);
    void *ptr = GetMemoryFromHeap(size);
    bytesLeft -= size;
    unlock(&lk);
    return ptr;
}
void free(void *ptr, int size) {
    lock(&lk);
    bytesLeft += size;
    FreeMemoryToHeap(ptr, size);
    signal(&cond);
    unlock(&lk);
}
```

假设我们有一个 `alloc(100)` 和 `alloc(10)` 处于睡眠状态，现在来了一个 `free(50)`，问题出现了：如果我们随意挑一个线程唤醒，假如挑了 `alloc(100)` 的那个，分配还是会失败，且我们错过了让 `alloc(10)` 成功的机会。一种解决方案是：用一个 broadcast() 函数替代 signal() 函数，它可以唤醒睡眠在条件变量上的所有线程。这种方式被称为 covering conditions。

broadcast() 相较于 signal() 的坏处在于：并不是所有的线程都能在被唤醒后运行下去，比如现在 buffer 里只有一个字符，而 broadcast() 唤醒了 100 个消费者，那么只有一个消费者能得到字符。运行不下去的字符会释放锁并再次进入睡眠，这种无谓打搅了睡眠线程的方式对性能影响很大。

正因如此，虽然我们在之前的生产者-消费者问题中可以使用一个条件变量+broadcast 的方式解决问题，但那时我们显然有很简单的 2-条件变量解决方案，所以不考虑这种效率较低的方式。不过在内存分配的场景中，broadcast() 几乎是唯一的选择。

## 30.4 Summary

略。
