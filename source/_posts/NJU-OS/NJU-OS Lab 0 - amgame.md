---
layout: post
date: 2022-03-11
title: "NJU-OS Lab 0: amgame"
categories: ["NJU-22020230 Operating Systems", "Lab Reports"]
tags: 
mathjax: true
---

该实验的主要目的是在 abstract-machine 抽象的 bare-metal 上编写一个小游戏。笔者实现的小游戏是贪吃蛇。

<!-- more -->

## API

我们需要使用的比较重要的 AM 的接口有：

* `putch()` 及其封装版本 `puts()` ，负责向控制台输出字符和字符串。
* `AM_TIMER_UPTIME` ，对该抽象寄存器进行读取可以获得系统启动以来运行时间的微秒数。
* `AM_INPUT_KEYBRD`，对该抽象寄存器进行读取可以获得一个 AM_INPUT_KEYBRD_T 结构体，其中存储了当前是否有按键被按下，被按下的按键编码等。
* `AM_GPU_CONFIG`，对该抽象寄存器进行读取可以获得 VGA 的基本信息，包括窗口宽高的像素数量。
* `AM_GPU_FBDRAW`，对该抽象寄存器写入一个 AM_GPU_FBDRAW_T 结构体可以向 VGA 输出一个方格的颜色。

## Game Logic

一个游戏的基本框架为：

```c
while (1)
{
    while (uptime() < next_frame);
    
    while ((key = read_key() != AM_KEY_NONE)
    	kbd_event(key);
    
    game_process();
    screen_update();
    next_frame += 1000 / FPS;
}
```

对于贪吃蛇游戏：

* 在 kbd_event() 中，我们需要处理 ESC 按键，令其调用 halt()，以及四个方向键用于控制贪吃蛇的行走方向。笔者还设置了按 `R` 键重新开始游戏的功能。

* 在 game_process() 中

    * 我们需要维护贪吃蛇每一节身体的位置，并将头部按照当前方向移动一格，判断有没有出现撞墙或者吃了自己的情况，如果出现了，在游戏主循环中应该停止画面更新。
    * 我们需要维护食物，如果食物和蛇头重合，则应该分数+1，并随机一个位置 (不能和蛇重合) 重新放置食物。
    * 为了逐步增加游戏的难度，笔者维护 `FPS = 5 + (score / 2)`，即每多得两分，贪吃蛇的移动速度便会加快一级。

* 在 screen_update() 中，我们可以将背景涂成黑色，将边缘涂成橙色，将蛇身涂成绿色，蛇头涂成蓝色，食物涂成红色。

  ​    