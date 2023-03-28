---
linktitle: "DiffTest"
title: "Differential Testing"
type: docs
math: true
date: 2023-03-28

menu:
    ps-coding:
        parent: Contents
        weight: 3
---

如何论证你写的程序是正确的？这是一个历史悠久且难以回答的问题。通常来说有两种思路：

* Verification: 形式化验证简单来说是通过数学手段证明你的程序的行为符合预期。这是对程序正确性的强而有力的证明，但通常非常困难、局限性很大、scalability 较差，因此更多用于对于安全/可靠性要求非常严苛的场景，如自动驾驶，火星车等。问题求解一中提到的“通过 loop invariant 证明循环正确性”可以认为是 verification 的一种非常简单的形式。由于 verificaton 需要大量的程序语言(PL)/形式化方法(FM)的背景知识，且对于简单算法问题来说颇有大材小用之嫌，故在这里不作赘述。
* Testing: 测试是更加常用的检查程序是否有漏洞的方法，其核心思想是构造大量的输入并检查目标程序在这些输入上的输出是否符合预期。测试有着天然的局限性：它只能证明你的程序有 bug，但无法证明你的程序没有 bug (OJ 的本质也是测试，因此在 OJ 上通过的程序并不一定是对的 ~~而且由于助教太懒数据太水很可能确实是错的~~)。不过我们总可以认为，如果我们构造的输入足够多且足够丰富，那么通过了所有测试的程序的可靠性相对会很高。

这里我们介绍一种简单易行的测试技术：differential testing (如果你曾经参加过算法竞赛，“对拍”的本质就是 DiffTest)。DiffTest 的核心思想是针对同一个问题完成两份不同的实现，然后构造大量的输入喂给两份实现观察它们的输出是否相同 (就像你每天早晨和同桌对作业答案，虽然你们都不能保证自己做的是对的，但做错且错得一样的概率毕竟很小，如果你们的解题思路不一样就更小了)。为了避免过于空洞，我们以“计算Fibonacci第n项”这个问题为例说明如何在 OJ 算法题上进行 DiffTest。

假设你已经写好了一个使用矩阵快速幂优化 Fibonacci 计算的程序 `fib.cpp`，现在希望对其进行测试。我们很容易写一个朴素版的 `fib_bruteforce.cpp` 来实现同样的功能 (虽然它能处理的 $n$ 规模较小)：

```c++
#include <bits/stdc++.h>
using namespace std;

const int MOD = 998244353;

int main ()
{
    int n, f0 = 0, f1 = 1, f2;
    cin >> n;
    for (int i = 2; i <= n; i++)
    {
        f2 = (f0 + f1) % MOD;
        f0 = f1; f1 = f2;
    }
    printf("%d\n", f2);
    return 0;
}
```

根据我们的预期，对于任意 $n$，如果 `fib.cpp` 和 `fib_bruteforce.cpp` 都给出了答案，那么答案应该是相同的。这个问题的好处在于测试数据形式非常简单：只有一个数。因此我们很容易写出一个数据生成程序 `gen_data.cpp` (注意我们的朴素版本无法处理过大的 $n$)：

```c++
#include <bits/stdc++.h>
using namespace std;

int main ()
{
    int limit = 10000000;
    mt19937 mt(time(nullptr));

    cout << mt() % limit + 1 << endl;
    return 0;
}
```

`mt19937` class 是一个非常高效的伪随机数生成器。你也可以使用 `rand()` 等其他函数来生成随机数 (注意设置随机种子，否则可能每次生成的随机数都一样)。

有了两份实现和一个数据生成器，接下来的任务就是写一个脚本来做如下循环：

```plaintext
while True
{
    DataGenerator >> data
    data >> Program1 >> output1
    data >> Program2 >> output2
    compare output1 and output2
}
```

如果你会在 Windows 下写 Batch/Powershell 脚本，你应该可以给出一个上述伪代码的轻巧实现；如果你使用类 Unix 系统，你大概率也可以给出一个 bash 脚本。如果你啥也不会，可以考虑使用下面给出的这份 Python 程序进行测试。

<details><summary>一份 Difftest Python 实现 <i>[Click to expand]</i></summary>

```python
"""
usage: difftest.py [-h] --impl IMPL IMPL --gen GEN [--num NUM] [--time TIME]

optional arguments:
  -h, --help        show this help message and exit
  --impl IMPL IMPL  two source code files
  --gen GEN         data generator, cpp or python
  --num NUM         number of tests
  --time TIME       time limit for each test, in seconds
"""

import os, subprocess
import atexit

objs = []

def compile(*progs):
    for prog in progs:
        assert os.path.exists(prog), f"{prog} not exist"
        if prog.endswith(".cpp"):
            obj = f"obj-{prog}"
            objs.append(obj)
            p = subprocess.run(f"g++ -o {obj} {prog} -O2", shell=True)
            assert p.returncode == 0, f"{prog} failed to be compiled, make sure that g++ is in your enviroment path"
    
def destructor():
    for obj in objs:
        if os.path.exists(obj):
            os.remove(obj)

def run(prog: str, tl: float, data: str):
    cmd = f"./obj-{prog}" if prog.endswith(".cpp") else f"python3 {prog}"
    p = subprocess.Popen(cmd, shell=True, stdin=subprocess.PIPE, stdout=subprocess.PIPE)
    try:
        stdout, _ = p.communicate(input=data.encode(), timeout=tl)
    except subprocess.TimeoutExpired:
        p.kill()
        raise Exception(f"{prog} exceed {tl}s time limit")
    return stdout.decode()

def dump(data, o1, o2):
    with open("input", "w") as f: f.write(data)
    with open("output1", "w") as f: f.write(o1)
    with open("output2", "w") as f: f.write(o2)

def cmp(data, o1, o2):
    t1, t2 = o1.strip().split("\n"), o2.strip().split("\n")
    if len(t1) != len(t2):
        dump(data, o1, o2)
        return False
    for line1, line2 in zip(t1, t2):
        if line1.strip() != line2.strip():
            dump(data, o1, o2)
            return False
    return True

if __name__ == "__main__":
    import argparse
    parser = argparse.ArgumentParser()
    parser.add_argument("--impl", nargs=2, required=True, help="two source code files")
    parser.add_argument("--gen", type=str, required=True, help="data generator, cpp or python")
    parser.add_argument("--num", type=int, default=100, help="number of tests")
    parser.add_argument("--time", type=float, default=5.0, help="time limit for each test, in seconds")
    args = parser.parse_args()

    atexit.register(destructor)

    compile(args.impl[0], args.impl[1], args.gen)

    for i in range(args.num):
        data = run(args.gen, args.time, "")
        res1 = run(args.impl[0], args.time, data)
        res2 = run(args.impl[1], args.time, data)
        assert cmp(data, res1, res2), "Wrong answer, input/output files dumped"
        print(f"Test {i+1} OK!")
```

</details>

---

DiffTest 显然比自己出数据，运行程序，再手动验证结果要高效得多。用正确的工具和方法做事能让你事半功倍。