---
linktitle: "I-Final"
title: "问题求解 I-Final 题解"
type: docs
math: true
date: 2023-03-05

menu:
    ps-oj-solutions-2022:
        parent: Contents
        weight: 1100
---

{{% alert note %}}

**Problem A: BrainF\*\*k Syntax Checker**

题意概括：给定一个 BF 程序，判断它是否符合语法要求。

{{% /alert %}}

BF 程序中的 `+` `-` `,` `.` `>` `<` 显然不会对程序的合法性造成影响，因此本题的主要任务是判断 BF 程序中的 `[` `]` 是否形成了合法的括号序列，即两两匹配。

值得注意的是，BF 程序中 `[` 和 `]` 的个数相同并不是括号序列合法的充要条件，比如 `][` 这个序列就是不合法的。在此基础上，我们还要保证 BF 程序的任意前缀中，`[` 的个数要大于等于 `]` 的个数。

<details><summary><b>Code</b> <i>[Click to expand]</i></summary>

```c++
#include <bits/stdc++.h>
using namespace std;
 
char program[1000];
string ValidChar = "+-,.<>[]";
 
int main()
{
    scanf("%s", program);
    int len = strlen(program), LeftRightDelta = 0;
    bool flag = true;
    for (int i = 0; i < len; i++)
    {
        if (ValidChar.find(program[i]) == string::npos) flag = false;
        if (program[i] == '[') LeftRightDelta++;
        if (program[i] == ']') LeftRightDelta--;
        if (LeftRightDelta < 0) flag = false;
    }
    puts((flag && LeftRightDelta == 0) ? "Yes" : "No");
    return 0;
}
```

</details>

---

{{% alert note %}}

**Problem B: BrainF\*\*k Interpreter**

题意概括： 给定一个 BF 程序，模拟其运行过程并输出最后的内存状态。

{{% /alert %}}

这是一道相当有意思的题目，解法也很多。该问题的核心难点是如何处理嵌套的中括号。考场上有很多同学利用递归处理中括号的嵌套，这是一个很好且可行的思路。我们在这里介绍另外一种不需要递归的思路，它书写起来更加简洁，且在一定程度上揭示了计算机运行的本质——大家在本学期的 *数字逻辑与计算机组成* 和下学期的 *计算机系统基础* 中会不断与这种逻辑打交道。

在题面的提示中我们已经给出了计算机执行指令的四步骤，我们在这里再重复一遍：

* 取指：取出下一条执行的指令。
* 译码：根据指令的格式确定该指令的类型。
* 执行：根据指令类型，执行该指令定义的动作。
* 跳转：根据指令类型和执行过程，确定下一条应当执行的指令的位置。
  
真正的计算机处理的是“汇编语言”，我们的 BF 解释器处理的是 BF 语言，这两者在本质上没有区别。计算机内部通常有一个程序计数器 (program counter, 简称 pc)，它的功能和计数没什么关系，反而像一个指针，用来指向下一条要执行的指令。我们来具体地看看四个步骤对应到 BF 语言应当如何操作。

* 取指：这是最简单的一步，对应到 C++ 代码大概就是 `char instruction = program[pc]`。
* 译码&执行：这两步在 BF 中可以放在一起做 (因为 BF 的所有指令都没有“操作数”)，你大抵会用一个 if/switch 语句来判断 instruction 是 8 个符号中的哪一种并执行相应的操作。前 6 种指令的操作都非常简单，这里不再赘述。`[` 和 `]` 本质上是分支控制指令，所以在“执行”这个步骤中它们什么也不用做。
* 跳转：前 6 种指令的跳转都非常简单：做完了就做紧接着的下一条指令，即 `pc++`。我们着重讲解 `[` 和 `]` 的跳转如何处理：
  * `[`：左中括号指令的逻辑是
    ```c++
    if (memory[pointer] != 0)
        pc++ // go into the loop
    else
        pc = (the position of the corresponding "]") + 1 // skip the loop
    ```
    `pc` 直接加一意味着进入循环，`pc` 跳转到对应的 `]` 的下一条指令意味着跳过循环。
  * `]`：右中括号指令的逻辑是
    ```c++
    pc = (the position of the corresponding "[")
    ```
    遇到右中括号，无条件跳转到对应的左中括号，开始新一轮循环条件判断。

  注意到 `[` 和 `]` 的处理都依赖与之匹配的“另一半”，因此在正式开始执行程序之前，我们要预处理一遍程序以知道与每个 `[`/`]` 配对的 `]`/`[` 在什么位置。

按照这四个步骤书写本题的代码，可以完成得轻松且高效，从中你也可以领悟到计算机系统设计的伟大智慧。

<details><summary><b>Code</b> <i>[Click to expand]</i></summary>

注：下面参考程序仅仅实现了一个 BF 解释器，输入格式和输出格式与原题并不相同。

```c++
#include <bits/stdc++.h>
using namespace std;

int memory[1000], pointer;
char program[1000]; int n;

int input_count, input[1000];

int jumpto[1000], stk[1000], stot;

int main ()
{
    scanf("%s", program); n = strlen(program);
    scanf("%d", &input_count);
    for (int i = 1; i <= input_count; i++) scanf("%d", input + i);

    // 预处理每个 [ 和 ] 对应的 ]/[
    stot = 0;
    for (int i = 0; i < n; i++)
    {
        if (program[i] == '[') stk[++stot] = i;
        if (program[i] == ']')
        {
            jumpto[i] = stk[stot];
            jumpto[stk[stot]] = i + 1;
            stot--;
        }
    }

    pointer = 0;
    int pc = 0, input_pt = 0;
    bool printed = false;
    while (pc < n)
    {
        int next_pc = pc + 1;
        switch (program[pc])
        {
            case '+': memory[pointer]++; break;
            case '-': memory[pointer]--; break;
            case '<': pointer--; break;
            case '>': pointer++; break;
            case ',': memory[pointer] = input[++input_pt]; break;
            case '.': printf("%d ", memory[pointer]); printed = true; break;
            case '[':
                if (memory[pointer] == 0)
                    next_pc = jumpto[pc];
                break;
            case ']': next_pc = jumpto[pc]; break;
        }
        pc = next_pc;
    }
    if (printed) puts("");
    for (int i = 0; i < 10; i++)
        printf("%d ", memory[i]);
    puts("");
    return 0;
}
```

</details>


---

{{% alert note %}}

**Problem C: DXY's Graph Problem**

题意概括：给定一张图 $G=(V, E)$，其中 $V$, $E$ 分别是点和边的集合。对于图中任意顶点 $v\in V$，令 $\text{cover}(v)\triangleq \\{(x, y)\in E: x=v\vee y=v\\}$ (即与该点相邻的所有边的集合)。问是否存在原图顶点的划分 $S_1, S_2$，满足 $S_1\uplus S_2=V$ (即 $S_1\cup S_2=V$ 且 $S_1\cap S_2=\emptyset$)，且

$$
\bigcup_{v\in S_1}\text{cover}(v)=\bigcup_{v\in S_2}\text{cover}(v)=E
$$

(注：本题完整题意里中文描述中有关“DXY可以通过自己的点集将原图恢复”的部分给大家带来了较大困扰，对此我们深表歉意。)

{{% /alert %}}

每条边都有两个端点，要想这条边被双方都覆盖到，那么这两个端点必须分属于不同的集合。因此可以成功划分的充要条件是：存在将整张图上的顶点黑白染色，且每条边的两个端点一黑一白的方案。

我们容易发现这样的方案如果存在一定是唯一的，因为我们已经规定了 1 号点属于 DXY，我们不妨将白色分配给 DXY，那么整个过程相当于从一个白色的 1 号点开始向外搜索，每搜索到一个新的节点就给它涂上和邻居节点不同的颜色。由于整个图是连通的 (即任意两个节点都存在路径互相可达)，所以这样的搜索一定能给每个节点一个唯一的颜色 (如果你想要严谨地证明，可以考虑数学归纳法)。在搜索的过程中，如果发现无法解决的矛盾 (例如某个节点同时和一个黑点和白点相邻)，则问题无解。

如果你愿意继续深究这个问题 (这部分知识对于解决这道题并无必要)，你会发现可以成功黑白染色充要条件是**图中不存在奇数长度的环**，这也是判断二分图 (Bipartite Graph) 的方法，大家在后续的问题求解课程中会学习到这个概念和相关的证明。

<details><summary><b>Code</b> <i>[Click to expand]</i></summary>

```c++
#include <bits/stdc++.h>
using namespace std;
 
const int MAXN = 1e5 + 10;
 
int n, m;
vector<int> v[MAXN];
 
int color[MAXN];
bool flag;
 
void dfs(int cur)
{
    for (auto neighbor : v[cur])
        if (color[neighbor] == 0)
        {
            color[neighbor] = 3 - color[cur]; 
            // 两种颜色用 1, 2 表示
            // 3 - color[cur] 则表示“另一种颜色”
            dfs(neighbor);
        }
        else if (color[neighbor] != 3 - color[cur])
            flag = false;
         
}
 
int main ()
{
    scanf("%d%d", &n, &m); int x, y;
    for (int i = 1; i <= m; i++)
    {
        scanf("%d%d", &x, &y);
        v[x].push_back(y); v[y].push_back(x);
    }
     
    color[1] = 1; flag = true;
    dfs(1);
     
    if (flag)
    {
        puts("Yes");
        int cnt1 = 0, cnt2 = 0;
        for (int i = 1; i <= n; i++) if (color[i] == 1) cnt1++; else cnt2++;
        printf("%d %d\n", cnt1, cnt2);
        for (int i = 1; i <= n; i++) if (color[i] == 1) printf("%d ", i);
        puts("");
        for (int i = 1; i <= n; i++) if (color[i] == 2) printf("%d ", i);
        puts("");
    }
    else
        puts("No");
     
    return 0;
     
}
```

</details>