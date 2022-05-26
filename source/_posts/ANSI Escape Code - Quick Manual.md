---
layout: post
date: 2022-02-18
title: "ANSI Escape Code: Quick Tutorial"
categories: ["NJU-22020230 Operating Systems", "Material"]
tags: 
mathjax: true
---

(注：该文档主要参考/翻译了[这篇博客](https://notes.burke.libbey.me/ansi-escape-codes/#:~:text=ANSI%20escapes%20always%20start%20with%20%5Cx1b%2C%20or%20%5Ce%2C,of%20these%20escape%20codes%20start%20with%20%5Cx1b%20%5B.)。)

ANSI Escape Code 是一种屏幕序列控制码，主要格式介绍如下：

<!-- more -->

## Control Sequence

一个控制序列总是由 `<ESC>` 开头的。`<ESC>` 的表示方法有 `\x1b`、`\e` 和 `\033` 三种，这三种表示方法等价。控制序列的语法为

```
control sequence ::= <\033[> + (p1;)(p2)...(pn) + <a letter> 
```

其中 `<>` 中的内容是必须要有的，`()` 中的内容是可以选择的。这里的 `<a letter>` 可以理解为一个函数的名字，而前面的若干个以 `;` 隔开的数字可以理解为函数的参数，例如

```
\033[0;1;3m
```

就可以理解为函数 `m(0, 1, 3)`。如果字母之前没有参数，则函数没有参数。

## Available Function

ANSI Escape Code 中可使用的常见函数如下表：

|      | name                       | signature  | description                                                  |
| ---- | -------------------------- | ---------- | ------------------------------------------------------------ |
| A    | Cursor Up                  | (n=1)      | Move cursor up by `n`                                        |
| B    | Cursor Down                | (n=1)      | Move cursor down by `n`                                      |
| C    | Cursor Forward             | (n=1)      | Move cursor forward by `n`                                   |
| D    | Cursor Back                | (n=1)      | Move cursor back by `n`                                      |
| E    | Cursor Next Line           | (n=1)      | Move cursor to the beginning of the line `n` lines down      |
| F    | Cursor Previous Line       | (n=1)      | Move cursor to the beginning of the line `n` lines up        |
| G    | Cursor Horizontal Absolute | (n=1)      | Move cursor to the the column `n` within the current row     |
| H    | Cursor Position            | (n=1, m=1) | Move cursor to row `n`, column `m`, counting from the top left corner |
| J    | Erase in Display           | (n=0)      | Clear part of the screen. 0, 1, 2, and 3 have various specific functions |
| K    | Erase in Line              | (n=0)      | Clear part of the line. 0, 1, and 2 have various specific functions |
| S    | Scroll Up                  | (n=1)      | Scroll window up by `n` lines                                |
| T    | Scroll Down                | (n=1)      | Scroll window down by `n` lines                              |
| s    | Save Cursor Position       | ()         | Save current cursor position for use with `u`                |
| u    | Restore Cursor Position    | ()         | Set cursor back to position last saved by `s`                |
| f    | …                          | …          | (same as G)                                                  |
| m    | SGR                        | (*)        | Set graphics mode. More below                                |

## SGR

SGR 函数的参数较为复杂，下表专门列举了 SGR 的各个参数的意义：

| value            | name / description                                           |
| ---------------- | ------------------------------------------------------------ |
| 0                | Reset: turn off all attributes                               |
| 1                | Bold (or bright, it’s up to the terminal and the user config to some extent) |
| 3                | Italic                                                       |
| 4                | Underline                                                    |
| 30–37            | Set text colour from the basic colour palette of 0–7         |
| 38;5;*n*         | Set text colour to index `n` in a [256-colour palette](https://commons.wikimedia.org/wiki/File:Xterm_256color_chart.svg) (e.g. `\x1b[38;5;34m`) |
| 38;2;*r*;*g*;*b* | Set text colour to an RGB value (e.g. `\x1b[38;2;255;255;0m`) |
| 40–47            | Set background colour                                        |
| 48;5;*n*         | Set background colour to index `n` in a 256-colour palette   |
| 48;2;*r*;*g*;*b* | Set background colour to an RGB value                        |
| 90–97            | Set text colour from the **bright** colour palette of 0–7    |
| 100–107          | Set background colour from the **bright** colour palette of 0–7 |

其中比较重要的用法列举如下：

* 30-37 号用于设置字符的颜色，编号与颜色的对应关系为

    | No.       | 0     | 1    | 2     | 3      | 4    | 5               | 6          | 7     |
    | --------- | ----- | ---- | ----- | ------ | ---- | --------------- | ---------- | ----- |
    | **Color** | Black | Red  | Green | Yellow | Blue | Magenta(紫罗兰) | cyan(青色) | white |

* 在设置完字符颜色/背景颜色后，之后输出的字符都会是当前的颜色。如果想要关闭这个颜色设置，需要再使用一次 `\033[0m` 来清空。

**Example**

```
\x1b[38;2;255;255;0mH\x1b[0;1;3;35me\x1b[95ml\x1b[42ml\x1b[0;41mo\x1b[0m
```

上述代码可以打印出一个每个字符颜色和背景各异的 "Hello" 字符串。

## Miscellany

另外一对有用的命令是 `\033[?25h` 和 `\033[?25l` ，用于显示和隐藏光标。此外，`\r` 和 `\033[1G]` 的作用相当：将光标移动到该行的开头。