---
title: "UCB-CS61A Lab 01: Variables & Functions, Control"
linktitle: 'Lab 01'
type: docs
draft: false

math: true

menu:
    cs61a-lab:
        parent: Contents
        weight: 1
---

## `return` and `print`

```python
def what_prints():
    print('Hello World!')
    return 'Exiting this function.'
    print('61A is awesome!')
```

该程序的运行结果是

```python
>>> what_prints()
Hello World!
'Exiting this function.'
```

注意 **print 的结果是不带引号的，return 的结果是带引号的**。

## Boolean Operators

如果多个语句用 AND 连接，那么 python 会执行到第一个为假的表达式，后面的不再执行，这就是**短路 (short circuiting)**。此时即使后面写了执行会导致 Error 的语句也不会 crash。OR 同理。

```python
>>> 1 / 0
ZeroDivisionError
>>> True or (1 / 0)
True
```

逻辑表达式总是会返回**最后一个元素**，该规则在短路情况下同样适用，下面是一些例子：

```python
>>> True and 13
13
>>> not 10
False
>>> False or []
[]
>>> 0 or False or 2 or 1 / 0
2
```

`0` `[]` `None` 也会被视为 False，但如果对一个 True 的东西取 not，返回结果只可能是 False。

## Debugging

### Traceback Message

Traceback Message 类似于 GDB 的回溯顺序，实际的函数调用顺序是从下往上的。

### Debugging Techniques

#### Running doctests

python3 的 `-m` 选项提供 doctest 功能：我们可以把简单的测试用例写在源代码中。例如

```python
# square.py
def square(x):
	"""
	>>> square(4)
	16
	"""
    return x * x
```

用一对 `"""` 括起来的注释内容中可以书写测试用例，测试用例的写法类似于 python 的交互界面，在 `>>>` 后写下要执行的语句，并另起一行写下期望的结果。使用 

```bash
python3 -m doctest square.py
```

进行测试，如果程序运行的结果和期望结果不同 python 会报错。

如果使用 `-v` 选项，python 不仅会报错，还会将通过的测试用例的信息也显示出来。

#### Interactive Debugging

使用一个交互式的 REPL 是 debug 的好方式。假设当前我们要 debug `file.py` 这个程序，使用命令

```bash
python3 -i file.py
```

可以打开一个 python 的 REPL，它已经执行了 `file.py` 中的所有定义，我们可以输入任意的命令进行测试。

#### Using ok

```bash
python3 ok -q MODULE_NAME -i
```

可以打开 REPL。

```bash
python3 ok q MODULE_NAME --trace
```

可以打开程序的可视化窗口。

