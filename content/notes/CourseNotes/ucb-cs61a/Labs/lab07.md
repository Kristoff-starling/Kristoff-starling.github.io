---
title: "UCB-CS61a Lab 07: Object-Oriented Programming"
linktitle: 'Lab 07'
type: docs
draft: false

math: true

menu:
    cs61a-lab:
        parent: Contents
        weight: 7
---


本实验中比较有意思的题目是实现卡牌游戏的“小项目”。我们在填写代码之前就可以思考一下如何用 oop 的思想来设计程序框架：卡牌、玩家、卡牌池应当被设计为对象，对象之间可以交互；普通卡牌是一个大类，拥有特殊效果的卡牌可以使用继承机制使用大类的属性并添加自己的独特属性；玩家手上有 5 张手牌，这是一个 "has a" 的关系，应当使用 composition 维护一个卡牌对象的列表……

> **Pass by Value v.s. Pass by Reference**
>
> Python 不像 C 语言那样可以明确给函数传递的参数是指针还是普通变量，但 Python 有自己的 specification：如果传递的是一个 immutable 的东西 (如 string, int)，那么是按值传递的；如果传递的是 mutable 的东西 (如 list, object)，那么是按引用传递的。因此在该实验中，将对象传给另一个对象的 method，method 对参数对象的修改可以直接影响原对象的状态。