---
linktitle: "STL-容器"
title: "C++标准模板库——容器"
type: docs
math: true
date: 2022-11-01

menu:
    c++0:
        parent: Contents
        weight: 9
---

之前章节介绍的分支、循环、函数、递归……等概念都是命令式编程语言通用的思想方法。时常有同学问：我们到底学的是 C 还是 C++？可以说之前大家写的程序基本都是 C 风格的 (除了 `cin` `cout` `string` 等少数内容）。这一章介绍的标准模板库 (Standard Template Library, STL)，是 C++ 区别于 C 的重要内容之一。STL 提供的内容将程序员从复杂的底层算法和数据结构中解放出来，使得程序员写程序更加得心应手。

标准模板库由四个部分构成：

* 算法 (Algorithm)
* 容器 (Container)
* 函数 (Function)
* 迭代器 (Iterator)

这一章主要讲解容器。

大家最近学习了“抽象数据结构“，STL的容器可以理解为C++库为大家实现好了一批数据结构，你只要读懂这些容器对外暴露的接口的功能说明，合理地使用这些接口，就可以在不知道底层实现的情况下享受这些数据结构的福利。我们在这里介绍几个最常用的STL容器和相关操作。

注：STL容器的用法极其丰富，这里只是浮光掠影简单介绍，大家可以在 [cppreference](https://en.cppreference.com) 上查询详细的方法列表。

## `vector`

`vector` 的中文名是向量，但这个容器和数学中的向量几乎毫无关系，vector 其实更像一个可以自由变换长度，完成一系列操作的“动态数组”。当你不能确定数组长度，或者需要在任意位置插入/删除元素时，vector 将会成为你的一大助力。下面我们通过例子展示 vector 的常见使用方法：

* 初始化
    ```c++
    // 通用格式： vector<类型> 名字(最大容量，初始值)
    #include <vector>  // 大家常用的 <bits/stdc++.h> 已经包括了该头文件

    int main ()
    {
        vector<double> v_double(0);  // 一个空的，存储的变量为 double 类型的 vector
        vector<int> v_int(10, 0);    // 一个长度为 10 的 vector，初始所有元素为 0
    }
    ```
* 元素访问
    ```c++
    // vector 和数组一样支持用下标访问，下标从 0 开始
    vector<int> v_int(10, 1);
    cout << v_int[2];  // 输出： 0
    cout << v_int[10]; // 下标越界，未定义行为！
    ```
* 迭代器

    虽然迭代器的概念有点复杂，但为了更好地使用容器，我们还是简单地介绍一下。迭代器可以理解为指向容器中某个元素的“指针”，如果你不知道什么是指针，你可以把它想象成一个“小箭头”。

    vector 中最常用的几个和迭代器相关的函数有：
    * `begin()`：返回一个指向第一个元素的迭代器。
    * `end()`：返回一个指向最后一个元素的“下一个位置”的迭代器。
    * `rbegin()`：返回指向最后一个元素的迭代器。
     
    <br/>
    
    迭代器可以通过简单的 “+1/-1” 操作向右/左移动。我们通过迭代器可以顺序访问数组中的所有元素：
    ```c++
    // 假设当前 vector<int> v 中有 5 个元素，分别是 1, 2，3, 4, 5
    for (vector<int>::iterator iter = v.begin(); iter != v.end(); iter++)
        cout << *iter << ' ';   // 通过 *iterator 的方式来获取”小箭头“指向的元素的值
    // 输出结果： 1 2 3 4 5
    for (vector<int>::iterator iter = v.rbegin(); iter != v.begin(); iter--)
        cout << *iter ** ' ';
    // 输出结果： 5 4 3 2，请体会 end() 与 begin() 的不同之处！
    ```

* 元素的添加和删除
    ```c++
    vector<int> v_int(3, 1);        // vector 内容：[1, 1, 1]
    // push_back(x) 方法用于在末尾添加元素 x
    v_int.push_back(2)              // vector 内容：[1, 1, 1, 2]
    v_int.push_back(3)              // vector 内容：[1, 1, 1, 2, 3]
    // insert(iter, x) 方法用于在迭代器 iter 指向的位置前插入元素 x
    v_int.insert(v_int.begin(), 3)  // vector 内容：[3, 1, 1, 1, 2, 3]
    // pop_back() 方法用于删除最后一个元素
    v_int.pop_back()                // vector 内容：[3, 1, 1, 1, 2]
    // erase(iter) 方法用于删除迭代器 iter 指向的元素
    v_int.erase(v_int.begin() + 1)  // vector 内容：[3, 1, 1, 2]
                                    // 注：删除了第二个元素
    // clear() 函数用于清空 vector
    v_int.clear()                   // vector 内容：[]
    ```

* 相关参数
    ```c++
    // 假设当前 vector<int> v 中有 5 个元素，分别是 1, 2，3, 4, 5
    cout << int(v.size());     // 输出：5
    bool isEmpty = v.empty()   // isEmpty = false
    ```

## `stack`

stack (栈) 是一个先进后出的容器，你可以把它想象成一个电梯：第一个进入电梯的人总是最后一个出来。下面是一些简单的用法：

```c++
stack<int> s;                   // 初始为空
// push(x) 方法向栈顶放入元素 x
s.push(1);
s.push(2);
s.push(3);                      // s的内容:  (底) [1, 2, 3] (顶)
// top() 方法用于获取栈顶元素
int currentTop = s.top();       // currentTop = 3, s: [1, 2, 3]
// pop() 方法用于弹出栈顶元素
s.pop();                        // s: [1, 2]
currentTop = s.top();           // currentTop = 2
// size() 方法用于获取 s 中元素个数
cout << int(s.size());     // 输出：2
// empty() 方法用于判断 s 是否为空
bool isEmpty = s.empty();       // isEmpty = false
```

## `queue`

queue (队列) 是一个先进先出的容器，你可以把它想象成一个双开门电梯：第一个进入电梯的人第一个出来。下面是一些简单的用法：

```c++
queue<int> q;                     // 初始为空
// push(x) 方法向队列的尾部加入元素 x
q.push(1);
q.push(2);
q.push(3);                        // q的内容: (队首) [1, 2, 3] (队尾)
// front() 方法用于获取队首元素
int currentFront = q.front();     // currentFront = 1
// q.pop() 方法用于弹出队首元素
q.pop();                          // q: [2, 3]
currentFront = q.front();         // currentFront = 2
// size() 方法用于获取 q 中元素个数
int currentSize = int(q.size());  // currentSize = 2
// empty() 方法用于判断 q 是否为空
bool isEmpty = q.empty();         // isEmpty = false
```

## `set`

set (集合) 顾名思义实现了一个集合应有的功能：插入、去重、判断一个元素是否存在。特别的是 set 中的元素还是按照顺序排列的 (这其实和集合定义中的无序性稍有不符)。下面是一些简单的用法：

```c++
set<int> s;                            // s: {}
// insert(x) 方法向集合插入元素 x
s.insert(2);                           // s: {2}
s.insert(1);                           // s: {1, 2}
s.insert(1);                           // s: {1, 2}，重复元素不会被反复插入
s.insert(3);                           // s: {1, 2, 3}
// find(x) 方法查询 x 是否在集合中，如果在则返回指向该元素的迭代器，否则返回 s.end()
set<int>::iterator iter_1 = s.find(1);
cout << *iter;                    // 输出：1
set<int>::iterator iter_0 = s.find(0);
bool isEnd = (iter_0 == s.end());      // isEnd = true
// erase(x) 方法用于删除元素 x
s.erase(2);                            // s: {1, 3}
// 使用迭代器按顺序访问 s,会发现元素是按顺序排列的
for (set<int>::iterator iter = s.begin(); iter != s.end(); iter++)
    cout << *iter << ' ';         // 输出：1 3
// clear() 方法用于清空集合
s.clear();                             // s: {}
```

## 映射

~~在标题中使用 map 这个单词会被自动渲染成地图。。。因此使用了“映射”作为标题~~

map (映射) 和数学中的映射一样，维护了一个 key-value pair 的集合。下面是一些简单的用法：

```c++
map<string, int> m;                          // 这是一个从 string 到 int 的映射
// 映射的插入非常简单：直接使用数组赋值的语法格式即可
m["apple"] = 1;
m["banana"] = 2;
// 访问一个 key 对应的 value：直接使用数组访问的语法格式
cout << m["apple"];                          // 输出：1
cout << m["orange"];                         // 对于不存在的 key，通常会输出默认值 0
m["apple"] = 3;
cout << m["apple"];                          // 输出：3
// find(x) 方法用于查询映射中是否有 x 这个 key，如果有则返回指向该 pair 的迭代器，否则返回 m.end()
map<string, int>::iterator iter_pear = m.find("pear");
bool isEnd = (iter_pear == m.end())          // isEnd = true
map<string, int>::iterator iter_banana = m.find("banana");
// 使用 .first 获取 key，.second 获取 value
cout << *iter.first << ' ' << *iter.second;  // 输出： banana 2
```