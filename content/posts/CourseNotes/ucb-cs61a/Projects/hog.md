---
title: "UCB-CS61A Project 01: Hog"
linktitle: '01: Hog'
type: docs
draft: false

math: true

menu:
    cs61a-projects:
        parent: Contents
        weight: 1
---

## Phase 1: Simulator

### Problem 1

模拟扔骰子计算得分。唯一需要注意的是即使中间摇出了 1 也要将传入的摇骰子次数做完。因为摇骰子的 index 是一个全局变量，这次的不摇完会影响下一次的结果。

### Problem 2

模拟 `picky_piggy`，本题代码中不允许使用 string 和 list，因此要使用分离数位的方法获得结果 (或者用大 if)。

### Problem 3

完成 take_turn()，调用 problem 1, 2 中写的两个函数即可。

### Problem 4

模拟 `hog_pile`，判断两人得分是否相同即可。

### Problem 5

模拟两个人完整的游戏流程。每轮先调用对应的 strategy 获得摇骰子的次数，然后调用 take_turn() 获得分数，再调用 hog_pile() 获得附加得分，最后切换玩家。

## Phase 2: Commentary

### Commentary Examples

say 函数是一类返回值为函数的高阶函数。调用 say 函数会打印出一些信息，并返回一个 say 函数用于下一次调用。最简单的 say 函数是每次逻辑不变的函数：

```python
def say(score0, score1):
    print(score0, score1)
    return say
```

稍复杂一些的 say 函数每次返回的函数行为可以不同，比如下面的例子在两人分数领先情况发生变化时会打印信息：

```python
def annouce_leader_change(last_leader=None):
    def say(score0, score1):
        if score0 > score1:
            leader = 0
        elif: score1 > score0:
            leader = 1
       	else:
            leader = None
        if leader != None and leader != last_leader:
            print('Player', leader, 'take the lead by', abs(score0 - score1))
        return annouce_leader_change(leader)
   	return say
```

`announce_leader_change()` 返回的 `say()` 函数被定义在 `announce_leader_change()`函数体内，变量 last_leader 会影响 `say()` 的逻辑。

下面展示比较有意思的 `both()` 函数，它会依次执行两个 say 函数：

```python
def both(f, g):
    def say(score0, score1):
        return both(f(score0, score1), g(score0, score1))
   	return say
```

### Problem 6

在 `play()` 的最后每次调用 say 函数，并令 say 等于新的 say 函数即可。

### Problem 7

自己书写一个稍复杂的 say 函数 `announce_highest()`。该问题中需要注意的是：如果没有声明 non local，我们不能在子函数中直接修改父函数中的变量，我们可以选择通过参数传递的方式把新的 last_score 和 running_high 传给父函数。

## Phase 3: Strategy

### Problem 8

计算若干次运行摇骰子函数的结果的平均值。除了编写高阶函数外，该 problem 的一个知识点在于：我们可以用 `*arg` 来将某个函数接收的所有参数灌给另一个要执行的函数。例如：

```python
def printed(f):
    def print_and_return(*args):
        result = f(*args)
        print('Result:', result)
        return result
    return print_and_return
```

随着调用 `printed()` 时的函数 f 的不同，我们后续执行 `print_and_return()` 时可以使用各种数量的参数。

### Problem 9

通过平均值选取当前最优的摇骰子次数。将摇骰子的函数 `roll_dice()` 喂给之前编写的 `make_averaged()` 即可。此外，为了通过 ok 的测试，循环必须按照从 1 到 10 的顺序写。

### Problem 10

调用 problem 2 中编写的 `picky_piggy()` 计算摇 0 的收益并和 cutoff 比较即可。

### Problem 11

和 problem 10 几乎没有差别，额外判断一下会不会触发 hog_pile 规则即可。

### Problem 12

可以参考的一些策略有：

* `picky_piggy_strategy()` 和 `hog_pile_strategy()`。
* 如果能够确保胜利，则应该选择小的摇骰子次数规避风险。
* ……