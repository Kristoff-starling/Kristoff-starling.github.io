---
title: "UCB-CS61A Lecture 19: Inheritance"
linktitle: '19: 继承'
type: docs
draft: false

math: true

menu:
    cs61a-lec:
        parent: Contents
        weight: 19
---

## Motivation

比如我们想要定义一些动物的 class：rabbit, elephant, lion... 它们作为动物有一些共同的属性和方法，各自也有独特的属性和方法。那么与其在各自的 class 中重复书写代码，我们不如定义一个动物 class，在这个 class 中定义动物共有的属性和方法，然后为各个动物建立 subclass，让这些 subclass **继承 (inherit)** 动物 class。

## Inheritance

上文中的动物 class 称为 base class，或者 superclass。下面是一个例子：

```python
class Animal:
    species_name = "Animal"
    scientific_name = "Animalia"
    play_multiplier = 2
    interact_increment = 1

    def __init__(self, name, age=0):
        self.name = name
        self.age = age
        self.calories_eaten  = 0
        self.happiness = 0

    def eat(self, food):
        self.calories_eaten += food.calories
        print(f"Om nom nom yummy {food.name}")
        if self.calories_eaten > self.calories_needed:
            self.happiness -= 1
            print("Ugh so full")
```

```python
class AmorphousBlob(Animal): # 在括号中写上superclass的名字！
    pass
```

subclass 中可以定义已经存在的变量和方法，这被称为 overriding。

在 subclass 的 method 中如果想要使用 superclass 中的内容，可以用 `super().xxx` 的方式访问。

## Multiple Inheritance

一个 class 可以继承多个 class 的内容，下面是一个例子：

```python
class Predator(Animal):
    def interact_with(self, other):
        if other.type == "meat":
            self.eat(other)
            print("om nom nom, I'm a predator")
        else:
            super().interact_with(other)
class Prey(Animal):
    type = "meat"
    calories = 200
```

接下来我们可以定义一些动物 class，同时继承 Animal 和 Predator/Prey：

```python
class Rabbit(Prey, Animal):
    pass
class Lion(Predator, Animal):
    pass
```

Python 会在所有的 superclass 中寻找对应的属性和方法。

## Composition

一个对象中可以包含其他 class 对象的 refenrence。例如我们可以在 animal 类中添加一个 mate 属性：

```python
class Animal:
    def mate_with(self, other):
        if other is not self and other.species_name == self.species_name:
            self.mate = other
            other.mate = self
```

## Composition v.s. Inheritance

Inheritance 适合表示 "is-a" 关系：

* Rabbit is a specific type of animal, so `Rabbit` inherits from `Animal`.

Composition 适合表示 "has-a" 关系：

* An animal has a mate, so `Animal` has `mate` as an instance variable.

## Quiz

```python
class Parent:
    def f(s):
        print("Parent.f")

    def g(s):
        s.f()

class Child(Parent):
    def f(me):
        print("Child.f")

a_child = Child()
a_child.g()
```

有两点需要注意：

* class 中的 method 都是 bound method，第一个参数代表的是自己，不论这个参数名字是 `self` `s` 还是 `me` 效果都是一样的。
* 在执行到 Parent 中的 `g()` 时，由于 Child override 了 `f()` 方法，所以 `g()` 调用的是 Child 中的 f 而不是 Parent 中的 f。