---
title: "UCB-CS61A Lecture 18: Objects"
linktitle: '18: 对象'
type: docs
draft: false

math: true

menu:
    cs61a-lec:
        parent: Contents
        weight: 18
---

## Python OOP Terminology

* 类 (class) 是一个定义新的数据类型的模板。
* class 的一个具体的实例 (instance) 被称为对象 (object)。
* 每个对象有一些 data attribute，称为 instance variable；每个对象也有一些 function attribute，称为 method。

## Classes

### Class Instantiation

```python
pina_bar = Product('Piña Chocolotta", 7.99, ["200 calories", "24 g sugar"]')
```

这里的 `Product(args)` 称为 constructor，constructor 被调用时，Python 会创建 class 的一个实例并调用 class 中的 `__init__()` 函数，该函数的第一个参数是对象自己，用 `self` 表示。

```python
class Product:
    def __init__(self, name, price, nutrition_info):
        self.name = name
        self.price = price
        self.nutrition_info = nutrition_info
        self.inventory = 0
```

在 `__init__()` 中，你可以定义 instance variable 并赋予它们初始值。该 class 的其他 method 可以使用和修改这些 instance variable。

### Method Invocation

```python
pina_bar.increase_inventory(2)
```

调用了该 class 中的一个 method。

```python
class Product:
    def increase_inventory(self, amount):
        self.invetory += amount
```

这里的 `pina_bar.increase_inventory()` 是一个 bound method：第一个参数被预先绑定为了 self，之后传参数只需要传一个就行。`pina_bar.increase_inventory(2)` 和 `Product.increase_inventory(pina_bar, 2)` 是等价的。

## Class Variables

class variable 是指不在任何一个 method 中赋值的变量，例如：

```python
class Product:
    sales_tax = 0.07
```

class variable 的特点在于所有该 class 的 instance 都有这个变量 (注：不同 instance 对该变量的修改是彼此不可见的)。

## Accessing Attributes

`getattr()` 函数允许我们通过 string 的方式来查询一个 attribute 的值。`hasattr()` 可以查询某个 attribute 是否存在：

```python
getattr(pina_bar, 'inventory')   # 1
hasattr(pina_bar, 'reduce_inventory')  # True
```

## Private v.s. Public

Python 的所有 attribute 都是 public 的。一个常见的约定是：如果某个 attribute 的名字以下划线开头，那么该 attritube 被认为是 private 的，虽然从语法上你仍然可以直接访问它，但你应当通过 public method call 对其进行操作。