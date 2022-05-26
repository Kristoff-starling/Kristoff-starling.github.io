---
layout: post
date: 2022-02-25
title: "OSTEP Chapter 27: Interlude: Thread API"
categories: ["WISC-CS537 Introduction to Operating Systems", "Book Notes"]
tags: 
mathjax: true
---

## 27.1 Thread Creation

创建线程的 API 为

```c
int pthread_create(      pthread_t      *thread, 
                   const pthread_attr_t *attr,
                         void *         (*start_routine)(void*),
                         void *         arg);
```

* thread 是指向 pthread_t 类型变量的指针，我们将来要通过这个结构体来控制这个线程，所以现在需要对其进行初始化。
* attr 表明了希望该线程拥有的属性，大多数情况下可以传 NULL，表示按照默认属性创建。
* start_routine 是一个指向函数的指针，新创建的线程会从这个函数开始执行。
* arg 是传给 start_routine() 的参数。

<!-- more -->

这里参数的类型和返回值类型都是 `void *` 类型的指针 (相当于只要求地址，不要求对地址类型的解读)，这是为了使得函数支持任意类型的参数和返回值。

## 27.2 Thread Completion

等待线程完成的 API 为

```c
int pthread_join(pthread_t thread,
                 void      **value_ptr);
```

* thread 表示要等待运行完成的线程结构体。
* value_ptr 是一个指向 `void *` 类型指针的指针，pthread_join() 返回时， value_ptr 会指向线程创建时的 start_routine() 函数的返回值。如果我们不在乎这个返回值，可以直接传入 NULL。

很多时候，我们需要的线程函数返回值只是一个数 (如 0 表示成功，1 表示失败)，这时我们有比较简单的写法：

```c
void *mythread(void *arg) {
    return (void *)(arg + 1);
}

int main () {
    int r; pthread_t p;
    pthread_create(&p, NULL, mythread, (void *)100);
    pthread_join(p, (void **)&r);
}
```

在线程函数中将返回值转换成 `void *` 类型，在主函数中我们只要将整型变量 r 的地址转换成 `void **` 类型，就可以直接把返回值存到 r 里面。

使用线程返回时要格外注意：返回值的实体不能在线程栈上，因为线程返回时线程栈会被释放。比如

```c
void *mythread(void *arg) {
    myret_t r; r = {10, 20};
    return (void *)&r;
}
```

这个写法是不合理的，返回时 r 已经被释放，返回结果是 UB。将结构体定义在堆区可以避免这个问题：

```c
void *mythread(void *arg) {
    myret_t *r = malloc(sizeof(myret_t));
    *r = {10, 20};
    return (void *)r;
}
```

## 27.3 Locks

用于获得互斥锁和释放互斥锁的 API 为：

```c
int pthread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_unlock(pthread_mutex_t *mutex);
```

第一个函数会尝试获得锁，直到获得了之后返回 (互斥锁不会轮询锁的状态，在得不到时会进入睡眠状态)。第二个第二个函数用于释放锁。

注：这两个函数是有返回值的。正常情况下它们应该返回 0。一个好的编程习惯是在调用这些 API 时随手检查返回值，比如封装成这样：

```c
void Pthread_mutex_lock(pthread_mutex_t *mutex) {
    int rc = pthread_mutex_lock(mutex);
    assert(rc == 0);
}
```

互斥锁在使用之前需要先初始化。初始化有两种方式：

* 在定义互斥锁变量时直接使用 INITIALIZER，它会按照默认属性初始化锁：

    ```c
    pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
    ```

* 在运行过程中初始化：

    ```c
    void Pthread_mutex_init(pthread_mutex_t *mutex) {
        int rc = pthread_mutex_init(mutex, NULL);
        assert(rc == 0);
    }
    int main () {
        pthread_mutex_t lock;
        Pthread_mutex_init(&lock);
    }
    ```

    pthread_mutex_init() 的第二个参数用于指定初始化锁的属性，通常情况下使用 NULL 即可。

锁用完之后，应当使用和初始化函数相对应的 pthread_mutex_destroy() 函数来销毁一个锁。

另有两个 API：

```c
int pthread_mutex_trylock(pthread_mutex_t *mutex);
int pthread_mutex_timedlock(pthread_mutex_t *mutex,
                            struct timespec *abs_timeout);
```

trylock() 会尝试获得锁，如果锁正在被占用则直接返回。timedlock() 会在一个指定的时间范围内尝试获得锁，如果这个时间段内未能获得锁就返回。trylock() 可以理解为 timedlock() 的 0 秒版本；之前的 lock() 可以理解为 timedlock() 的无限长时间版本。

这两个 API 不常用，但在一些情境下可以用于避免死锁。

## 27.4 Condition Variables

条件变量的两个主要 API 为

```c
int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);
int pthread_cond_signal(pthread_cond_t *cond);
```

调用 cond_wait() 时，当前线程必须已经拥有互斥锁 mutex。cond_wait() 会释放 mutex 并将当前线程睡眠在条件变量 cond 上。当另外的某个线程调用 cond_signal() 唤醒了该线程时，该线程会重新尝试获得互斥锁 mutex，获得了之后从 cond_wait() 函数返回。

使用条件变量之前要先对条件变量初始化，其方法和互斥锁是类似的：

```c
pthread_cond_t cond = PTREAD_COND_INITALIZER;
```

## 27.5 Compiling and Running

要使用上述的 POSIX 线程库中的函数，我们需要在源代码中 `#include <pthread.h>` ，并在编译时加入 `-lpthread` 选项。

