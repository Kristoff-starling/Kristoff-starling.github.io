---
layout: post
date: 2022-02-25
title: "OSTEP Chapter 27 - Homework"
categories: ["WISC-CS537 Introduction to Operating Systems", "Homework"]
tags: 
mathjax: true
---

## Questions

1. First build main-race.c. Examine the code so you can see the (hopefully obvious) data race in the code. Now run helgrind (by typing valgrind --tool=helgrind main-race) to see how it reports the race. Does it
point to the right lines of code? What other information does it give to you? <!-- more -->
2. What happens when you remove one of the offending lines of code? Now add a lock around one of the updates to the shared variable, and then around both. What does helgrind report in each of these cases?
3. Now let’s look at main-deadlock.c. Examine the code. This code has a problem known as deadlock (which we discuss in much more depth in a forthcoming chapter). Can you see what problem it might have?
4. Now run helgrind on this code. What does helgrind report?
5. Now run helgrind on main-deadlock-global.c. Examine the code; does it have the same problem that main-deadlock.c has? Should helgrind be reporting the same error? What does this tell you about tools like helgrind?
6. Let’s next look at main-signal.c. This code uses a variable (done) to signal that the child is done and that the parent can now continue. Why is this code inefficient? (what does the parent end up spending its time doing, particularly if the child thread takes a long time to complete?)
7. Now run helgrind on this program. What does it report? Is the code correct?
8. Now look at a slightly modified version of the code, which is found in main-signal-cv.c. This version uses a condition variable to do the signaling (and associated lock). Why is this code preferred to the previous
version? Is it correctness, or performance, or both?
9. Once again run helgrind on main-signal-cv. Does it report any errors?

## Solutions

1. helgrind 工具可以准确地报告出 `main-race.c` 的第 15 行和第 8 行的操作存在 data race。此外，helgrind 可以报告出产生数据竞争的地址 (0 bytes inside data symbol "balance")，还可以报告线程创建的过程：

    ```
    ---Thread-Announcement------------------------------------------
    Thread #2 was created
       at 0x49A9D42: clone (clone.S:71)
       by 0x4878281: create_thread (createthread.c:103)
       by 0x4879C8B: pthread_create@@GLIBC_2.2.5 (pthread_create.c:821)
       by 0x484D627: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
       by 0x109209: main (main-race.c:14)
    ```

2. 如果删除掉任意一个对 balance 的操作，helgrind 就不会报告错误。如果只用锁保护一处操作，helgrind 仍然会报告 data race (且能报告处哪一处操作被锁保护了)。如果用锁将两处操作都保护了，helgrind 不会报告错误。

3. `main-deadlock.c` 存在发生死锁的风险：如果 p1 线程获得了锁 m1，然后 p2 线程获得了锁 m2，这时 p1 线程试图获得锁 m2，p2 线程试图获得锁 m1，它们都得不到需要的锁，也不会释放自己已经得到的锁，从而陷入死锁。

4. helgrind 会报告形如下面所示的错误：

    ```
    Thread #3: lock order "0x10C040 before 0x10C080" violated
    
    Observed (incorrect) order is: acquisition of lock at 0x10C080
     ...
     followed by a later acquisition of lock at 0x10C040
       ...
    
    Required order was established by acquisition of lock at 0x10C040
     ...
     followed by a later acquisition of lock at 0x10C080
      ...
    
    Lock at 0x10C040 was first observed
     ...
     Address 0x10c040 is 0 bytes inside data symbol "m1"
    
    Lock at 0x10C080 was first observed
     ...
     Address 0x10c080 is 0 bytes inside data symbol "m2"
    ```

    对于程序中的任意两个锁，所有获取锁的行为都一定要按照相同的顺序，否则就有可能引发死锁。helgrind 根据这个原理检测到了可能存在的死锁，并且给出了两次不同的顺序以及锁的名称。

    > **为什么我无法使死锁暴露出来？**
    >
    > 两个线程的创建是有时间差的，新创建的线程可以利用这个时间差把两个锁都得到，这样死锁就不会暴露出来。
    >
    > 为了更好地观测死锁现象，我们可以在 worker() 函数的开头添加一句 `usleep(1)`，这会让线程在此处等待至少 1 微秒的时间。考虑到系统活动等因素，`usleep(1)` 其实会带来一段时长比较随机的 delay，这让两个线程被拉回“同一起跑线”的几率大大增加。此外，多次重复实验，即可比较容易地观察到死锁现象。

5. `main-deadlock-global.c` 中由于有一个外层的大锁保护，所以不会发生死锁。但使用 helgrind 检查仍然和会报告有非法的锁获得顺序问题。helgrind 只是记录访问锁的顺序并判断是否出现了环，并不能准确地判断死锁是否可能发生。

6. 主线程在子线程打印的时候会轮询 done 变量，让 CPU 空转，所以这个方法是不高效的。

7. helgrind 报告该程序存在数据竞争。两个线程对 done 的读写没有用锁保护起来，存在 race condition。

8. `main-signal-cv.c` 相较于前者有两个改进：一是两个进程对 done 变量的访问都用锁保护了起来，保证了不会发生 race condition；二是该程序使用了条件变量，如果主线程检查 done 结果为 0，会在条件变量上睡眠，睡眠的线程可以被 CPU 调度出去，从而不占据 CPU 时钟周期。等到子线程打印完了唤醒条件变量，主线程再来检查 done 变量并打印自己的内容。

9. helgrind 没有报告任何错误。

