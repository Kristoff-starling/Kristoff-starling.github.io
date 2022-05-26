---
layout: post
date: 2022-03-08
title: "NJU-OS Lecture 07: Concurrent Programming - Real World"
categories: ["NJU-22020230 Operating Systems", "Lecture Notes"]
tags: 
mathjax: true
---

## High Performance Computing

HPC: 解决需要 massive computation 的任务。

基本思路：计算图会分成若干层，每层会有很多要计算的节点。将每层的任务先分配到机器，机器里再分配到线程 (两级分解)，并行计算这些节点后，用一个 "join()" 汇总一下 (涉及线程、机器、共享内存之间的通信)。

HPC 可以实现的原因：数据的局部性。例如我们想利用 HPC 进行一团气体的运动模拟，那么我们可以将立方体容器内的气体切分成很多块，每个线程管一块。每个线程管理的气体绝大部分的运动都是本地的，我们只需要处理少部分的相邻块的气体分子交换即可。

<!-- more -->

### Example: Mandelbrot Set

```c
// mandelbrot.c
#include "thread.h"
#include <math.h>

int NT;
#define W 6400
#define H 6400
#define IMG_FILE "./mandelbrot.ppm"

static inline int belongs(int x, int y, int t) {
  return x / (W / NT) == t;
}

int x[W][H];
int volatile done = 0;

void display(FILE *fp, int step) { 
  static int rnd = 1;
  int w = W / step, h = H / step;
  // STFW: Portable Pixel Map
  fprintf(fp, "P6\n %d %d 255\n", w, h);
  for (int j = 0; j < H; j += step) {
    for (int i = 0; i < W; i += step) {
      int n = x[i][j];
      int r = 255 * pow((n - 80) / 800.0, 3);
      int g = 255 * pow((n - 80) / 800.0, 0.7);
      int b = 255 * pow((n - 80) / 800.0, 0.5);
      fputc(r, fp); fputc(g, fp); fputc(b, fp);
    }
  }
}

void Tworker(int tid) {
  for (int i = 0; i < W; i++)
    for (int j = 0; j < H; j++)
      if (belongs(i, j, tid - 1)) {
        double a = 0, b = 0, c, d;
        while ((c = a * a) + (d = b * b) < 4 && x[i][j]++ < 880) {
          b = 2 * a * b + j * 1024.0 / H * 8e-9 - 0.645411;
          a = c - d + i * 1024.0 / W * 8e-9 + 0.356888;
        }
      }
  done++;
}

void Tdisplay() {
  float ms = 0;
  while (1) {
    FILE *fp = popen("viu -", "w"); assert(fp);
    display(fp, W / 256);
    pclose(fp);
    if (done == NT) break;
    usleep(1000000 / 5);
    ms += 1000.0 / 5;
  }
  printf("Approximate render time: %.1lfs\n", ms / 1000);

  FILE *fp = fopen(IMG_FILE, "w"); assert(fp);
  display(fp, 2);
  fclose(fp);
}

int main(int argc, char *argv[]) {
  assert(argc == 2);
  NT = atoi(argv[1]);
  for (int i = 0; i < NT; i++) {
    create(Tworker);
  }
  create(Tdisplay);
  join();
  return 0;
}
```

这段代码使用一些数学手法画出一幅清晰度很高，非常美观的分形图。代码中有一些有意思的小工具：

* Tdisplay 线程中使用了 viu 命令行工具，它可以将一张图片以较低的分辨率打印在终端上，这可以在让我们看到图像生成的大致过程。我们可以看到该程序将画面切分成了多个部分，每个线程控制一个。

* Tdisplay 线程最终将像素信息输出到了一个 ppm (portable pixel map) 文件中。对于 C 语言来说，这是一种非常方便的输出图像的方式：只需要在文件开头输出

    ```c
    P6
    W H COLOR // 宽，高，颜色数
    ```

    然后一行一行地将每个点的 RGB 值输出即可。

## Data Center

多副本情况下的低延迟、高可靠性问题。这其中存在一些互相矛盾的点

* Availability - 数据在 data center 应该有多个副本，这样如果某个副本坏了，数据不会丢失。此外，为了各地地人可以快速读取，服务器会把本地城市 data center 的数据直接返回。
* Consistency - ”我在南京屏蔽我妈，我妈在广州也要响应这个屏蔽。” 不能因为图快而不同步地直接读取本地副本。
* Partition Tolerance

这门课的问题：如何在一台计算机上既可能高效地处理并行请求？

我们的工具：线程、协程

* 线程：可以在操作系统的调度算法下进行切换，但线程切换很“重”：需要保存上下文，需要陷入内核 (privilege level 的改变)……
* 协程：更轻量级的并发 (只要遵守 calling convention，切换时不需要保存那么多寄存器，不需要进操作系统)，但只有执行 co_yield() 才会切换，如果某个线程执行了很慢的 I/O 系统调用，剩下的协程就都在摸鱼。

### Goroutine

Go 是一门专门服务系统编程的语言。go 支持所谓的 goroutine: 在概念上它是线程，但它是使用类似于协程的原理实现的。每个 CPU 上只有一个线程，但这个线程下有多个 goroutine，每个 CPU 上有一个 Go worker 负责调度这些 goroutine。goroutine 切换的方式和协程相似，因此非常轻量级。对于某些会导致阻塞的系统调用 (如 sleep, rand)，go 会将他们改成 non-blocking 的版本，如果需要陷入内核进行长时间操作，它就会 yield 到另一个 goroutine 执行。这样，goroutine 将操作系统和 CPU 都利用到了100%。

```go
// Example from "The Go Programming Language"

package main

import (
  "fmt"
  "time"
)

func main() {
  go spinner(100 * time.Millisecond)
  const n = 45
  fibN := fib(n) // slow
  fmt.Printf("\rFibonacci(%d) = %d\n", n, fibN)
}

func spinner(delay time.Duration) {
  for {
    for _, r := range `-\|/` {
      fmt.Printf("\r%c", r)
      time.Sleep(delay)
    }
  }
}

func fib(x int) int {
  if x < 2 { return x }
  return fib(x - 1) + fib(x - 2)
}
```

在主函数中，我们使用 `go spinner` 创建了一个新的 goroutine。该程序可以一边计算 Fibinacci 数列，一边在 spinner 协程中打印旋转的进度条。

绝大部分的并发问题都可以通过生产者-消费者模型来解决。Go 语言封装了生产者-消费者队列，提供了用于 goroutines 之间通信的 API：

```go
package main

import "fmt"

var stream = make(chan int, 10)
const n = 4

func produce() {
  for i := 0; ; i++ {
    fmt.Println("produce", i)
    stream <- i
  }
}

func consume() {
  for {
    x := <-stream
    fmt.Println("consume", x)
  }
}

func main() {
  for i := 0; i < n; i++ {
    go produce()
  }
  consume()
}
```

`make(chan int, 10)` 定义了一个通道，并规定了其最大容量 (最大容量为 0 的 channel 可以用来同步)。生产者通过 `stream <- i` 往队列里面加东西，消费者通过 `x := <-stream` 从队列里面取东西。其内部的锁，睡眠等细节问题全部对程序员透明。

## Web 2.0

在网页这种计算量、并发量都不大的场景下，我们使用的模型是单线程+事件模型。事件按照顺序执行，具有原子性 (减少了发生并发 bug 的可能)，遇到耗时的 API 就立刻返回 (结束一个事件)。

为了代码的可维护性，描述流图的语言正在发展。