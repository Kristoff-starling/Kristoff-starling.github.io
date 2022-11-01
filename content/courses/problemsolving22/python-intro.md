---
title: 'Python 快速入门教程'
summary: 一份简短的 Python 教程，帮助你快速掌握 Python 中最基础的语法(正在更新中）
date: 2022-10-31
---

{{% alert note %}}
**声明**


Python语言的学习不是这门课的硬性要求，因此我们**不会**在OJ作业/期末测试中强制大家使用Python完成习题。但我们仍然强烈建议你在初学阶段至少涉猎一下这样一门与 C/C++ 风格迥异的语言。

接触 Python 可以给你带来不限于以下好处：

* 感受 Python 精简灵活的语法：在这里你不用书写头文件，不需要书写 `main()` 函数，不需要在每行后面写分号，不需要定义变量……Python 提供的丰富的内置数据类型让你可以轻松自由地操纵数据。
* 感受函数式编程范式：Python 是一门兼有命令式和函数式风格的语言。在 Python 中你可以轻而易举地玩转高阶函数、lambda表达式等，感受函数式编程的独特思想和魅力。
* 强大的第三方库：Python 丰富的第三方库的支持可以让你轻松地完成一些“惊为天人”的东西，比如处理电脑文件的自动化脚本，带有图形界面的小游戏等等。

~~接触 Python 可能带来的坏处：总是在 C/C++ 程序中写出 Python 风格的会导致编译错误的代码。~~

我们不会介绍 Python 的安装，如果你感兴趣可以上网自行搜索教程安装 Python 的运行环境，并亲自上手感受这样一门现代语言。你目前只需要能看懂这份教程即可。
{{% /alert %}}

想必大家听过一句著名的广告词“人生苦短，我用Python”。Python是世界上程序员最喜爱的编程语言之一 (主要原因已经在上方的蓝框中涉及了一部分)。我们仍然用输出 "Hello, world!" 的例子来入门：
```python
print('Hello, world!')
```
是的，只需要一行。不需要头文件、`main()` 函数，也不需要理解复杂的 `scanf()`/`printf()` 语法和输入输出流，一个直观的 `print()` 函数解决一切！

下一个例子是喜闻乐见的 a+b problem：
```python
a = int(input())
b = int(input())
print(a + b)
```
在 Python 中你不需要提前定义变量，可以“拿来就用”，这是因为 Python 是一门动态语言，可以在执行的过程中进行 type inference。这里需要注意的是 `input()` 读入进来的内容默认是字符串类型，我们需要用 `int()` 将其转换为整数类型。如果你想要查看一个变量存储的数据的类型，你可以使用 `type()` 函数：
```python
a = input()    
print(type(a))   # 在 Python 中，你可以用"#"进行行末注释！
"""
在 Python 中，你可以用三引号框住
一段注释！
"""
```
在 Python 中定义函数可以使用 "def" 关键字：
```python
def add(a, b):       # 注意这里必须有冒号！
    return a + b

print(add(1, 2))     # 输出：3
```
值得注意的是，Python没有C/C++语言中一层层的大括号，所以Python程序需要依靠严格的缩进来保证清晰的程序结构。因此函数体的语句前方必须有缩进！类似地，if语句的分支，循环语句的循环体也必须保证正确的缩进。

Python中的判断语句：
```python
def judge_even_odd(x):
    if x % 2 == 0:     # 注意这里要有冒号！
        return 'even'
    else:              # 注意这里要有冒号！
        return 'odd'

print(judge_even_odd(1))  # 输出：odd
```

Python中的循环语句：
```python
def cumulative_sum(n):
    res = 0
    while n >= 0:     # 注意这里要有冒号！
        res += n
        n -= 1
    return res

print(cumulative_sum(10)) # 输出：55
```

注意：在 Python 中 if/while 的判断条件左右是不需要加括号的。

一个简单的递归程序：
```python
def fibonacci(n):
    if n == 0:
        return 0
    elif n == 1:
        return 1
    else:
        return fibonacci(n-1) + fibonacci(n-2)

print(fibonacci(9)) # 输出：34 
```