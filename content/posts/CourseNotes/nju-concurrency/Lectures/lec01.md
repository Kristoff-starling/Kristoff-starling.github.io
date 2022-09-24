---
title: "Lecture 01: Introduction"
linktitle: 'Lecture 01'
type: docs
date: '2022-09-06'
draft: false
featured: false

math: true

menu:
    nju-concurrency-lectures:
        parent: Contents
        weight: 1
---

- [Implementations of Concurrent Programs](#implementations-of-concurrent-programs)
- [Problem with Concurrency](#problem-with-concurrency)
- [Model Summary](#model-summary)
- [Programmers View](#programmers-view)
- [Memory Model](#memory-model)
  - [Store Buffering](#store-buffering)
  - [Load Buffering](#load-buffering)
- [WMM for High-Level Languages](#wmm-for-high-level-languages)

## Implementations of Concurrent Programs

* Python:

    ```python
    import threading
    def func():
        ...
    if __name__ == '__main__':
        t = threading.Thread(target=func,args=...)
        t.start()
        t.join()
    ```

* Java: use "Runnable" module

* C++: use `<thread>` header file.

Some programming languages use message-passing mechanism instead of sharing memory, such as Go. But we can use the same abstraction.

## Problem with Concurrency

Interleaving semantics means that the execution is non-deterministic, which is difficult for bug detection and reproduction. 

```c++
void thread_function() {
    for (int i = 0; i < 100; i++)
        std::cout << "new thread" << i << '\n';
}
int main () {
    std::thread t(thread_function);
    for (int i = 0; i < 100; i++)
        std::cout << "main thread" << i << '\n';
}
```

If we use `<<` to  concatenate multiple outputs, the printing is not thread-safe and the example code results in interleaving outputs.

The simplest solution to this is to use locks (synchronization operations):

```c++
std::mutex mu;
void shared_output(std::string msg, int i) {
    std::lock_guard<std::mutex> guard(mu);
    std::cout << msg << i << '\n';
}
```

## Model Summary

* Multiple threads
* Single shared memory
* Objects live in memory
* Unpredictable asynchronous delays.

## Programmers View

* Parallel composition, shared memory & interleaving semantics
* Lock & synchronization operations
* Concurrent objects (e.g. implementation of locks) and their clients

## Memory Model

In computing, a memory model describes the interactions of threads through memory and their shared use of data.

When we write codes, we use sequential consistency (SC) model to understand it. However:

* No multiprocessor implements SC.
* Modern compilers may invalidate the order.

Each hardware has its own memory model. Theses models are called **weak/relaxed memory model**. (Here the "weak" means that practical memory model may have more "unexpected" behavior.)

### Store Buffering

```c++
Initially: x = y = 0;
x = 1;   |   y = 1;
r1 = y;  |   r2 = x;
```

It's possible that `r1 = r2 = 0` ! (e.g. in X86.)

X86's TSO memory model:

<img src="/img/x86-TSO.png" style="zoom:50%;" />

When we write data to the memory, the processor puts the data to the buffer and updates the buffer to the memory later. Each processor's buffer is invisible to other processors.

A possible execution leading to `r1 = r2 = 0` is shown as follows:

* `x = 1` in T1 (write to buffer)
* `y = 1` in T2 (write to buffer)
* `r1 = y` in T1 (no `y` in T1's buffer, read from memory, get 0)
* `r2 = x` in T2 (no `x` in T2's buffer, read from memory, get 0)
* Update buffers to memory.

### Load Buffering

```c++
Initially: x = y = 0;
r1 = x;  ||  r2 = y;
y = 1;   ||  x = 1;
```

It's possible that `r1 = r2 = 1` ! (e.g. in ARM)

## WMM for High-Level Languages

For programmers, if they write code that satisfies DRF property, the compilers and processors guarantee SC behavior.

We should embrace WMM since it's part of the reality. Furthermore, many programs don't rely on the strict SC model to ensure correctness.