---
linktitle: "快速输入输出"
title: "快速输入输出"
type: docs
math: true
date: 2023-04-19

menu:
    ps-coding:
        parent: Contents
        weight: 4
---

大家比较熟悉的输入输出有两种：

```c++
int x;

scanf("%d", &x);         // C Style
printf("%d\n", x);

std::cin >> x;           // C++ style
std::cout << x << endl; 
```

如果输入/输出量过大，以至于输入输出部分成为算法问题实现的效率瓶颈，那么采取合适的方法进行快速输入/输出就变得非常重要。本章节介绍若干种常见的输入输出优化。

{{% alert warning %}}
输入输出函数背后的实现非常复杂，其效率与缓冲区设计、操作系统、磁盘IO等多个环节息息相关。因此本章节会尽可能略去原理的讲解 (或者以补充链接的形式给出)。如果你无法理解这其中的奥妙，不用担心，先把写法学会，等你学过了计算机体系结构/计算机系统基础/操作系统后就会对它们有更深刻的认识。
{{% /alert %}}

### 关闭与 stdio 的同步

一种[主流的说法](https://stackoverflow.com/questions/1042110/using-scanf-in-c-programs-is-faster-than-using-cin) 是: `cin` 比 `scanf` 读入要慢。在选择接受这条“定理”之前，你应当自己做个实验来验证一下：

```c++
int N = 30000000;
int a[N];
int main ()
{
    for (int i = 0; i < N; i++) cin >> a[i];
    for (int i = 0; i < N; i++) scanf("%d", a + i);
}
```

将 $N$ 设置成一个很大的数，并用 `cin` 和 `scanf` 分别跑一遍测试时间。不出意外的话你会发现这条定理确有其正确之处。但事实上 `cin` 比 `scanf` 看上去慢的一个主要原因是：`cin` 花了很多代价进行缓冲区同步，从而让你可以在混用 `cin` 和 `scanf` 的情况保证程序的正确性。我们可以在 `main()` 函数开头显式地通过添加一句 `ios::sync_with_stdio(false);` 来关闭这种同步。关闭同步后，你可以再尝试一次上述的效率实验，你会发现 `cin` 和 `scanf` 其实没什么差别。 

需要注意的是：一旦手动关闭了同步，你就不能再混用两种风格的输入函数。你可以尝试运行下面的例子：
```c++
#include <bits/stdc++.h>
using namespace std;

int main ()
{
	ios::sync_with_stdio(false);
	int a, b, c;
	cin >> a >> b;
	scanf("%d", &c);
	cout << a << ' ' << b << ' ' << c << endl;
	return 0;
}
```
你会发现该程序会给你意想不到的结果。

### 使用 `getchar()` 代替 `scanf()`

C库提供的 `scanf()` 函数功能丰富，例如可以用 `%d` `%s` `%p` `%x`，甚至正则表达式，去匹配各种类型的数据。但功能丰富的代价就是效率不够高。如果你确定当前场景下需要读入的一定是某种特定类型的数据 (例如整数)，那么可以考虑使用 `getchar()` (其功能为读入一个字符) 来手写一个读入函数，这通常能获得更好的效率。下面展示一个读入 int 类型整数的函数：

```c++
int getint()
{
    char ch; int res; bool f;
    while (!isdigit(ch = getchar()) && ch != '-');   // 过滤所有不是数字和"-"的字符
    if (ch=='-')
        f = false, res = 0;
    else
        f = true, res = ch - '0';                    // 判断正负性
    while (isdigit(ch = getchar()))
        res = res * 10 + ch - '0';                   // 读取数字并计算
    return f ? res : -res;
}
```

### 使用 `\n` 代替 `endl`

有的时候，你会发现 `cout << "\n";` 比 `cout << endl;` 要快。这是因为 `endl` 会强制冲刷缓冲区，在缓冲区没满的时候多次冲刷会让效率变低。([参考链接](https://stackoverflow.com/questions/213907/stdendl-vs-n) )

{{% alert tip %}}

**什么是缓冲区?**

了解缓冲区设计的动机需要对计算机系统中的 memory hierarchy 有一定了解，这里尝试尽可能短和通俗地地解释清楚这个概念。

假设你人在宿舍，要去图书馆借书。上午要去借A，下午要去借B，晚上去借C，于是你跑了三趟图书馆，这非常耗时。假设你是一个先知，上午就知道了自己要借 ABC 三本书，为了节省跑图书馆的时间，你在宿舍楼下放了一个书架，上午你一次性抱了三本书回来放在楼下的书架上，后面每次你需要书了就只需要下楼从书架上拿即可。还书也是相同的道理，你发现与其跑三趟，不如把要还的书先放在书架上，等手里所有的书都看完了再把书架上的书一起还到图书馆。

这个例子里
* 从图书馆借书/还书就是输入输出。相比较 `a[i]=b[j]` `a++` 这样的操作，执行一次输入输出是十分耗时的。
* 书A/B/C是数据，或者说即将读入/准备输出的字符。
* 书架是缓冲区。缓冲区就像一个大数组，暂存读入和输出数据。所谓的“冲刷缓冲区”就是将书架里的书全部还回图书馆/将缓冲区的数据真正输出。

因此，使用 `endl` 就像每次往书架上放书时都强制把书架上所有的书放回图书馆，效率自然不高。

{{% /alert %}}

### 使用 `fread()`/`fwrite()`

一般只有在读入/输出量极大的时候我们才会考虑使用 `fread()`/`fwrite()` 完成输入输出。如果延用之前借书的例子，`fread()` 和 `fwrite()` 相当于手动开辟一个巨大的书架，这个书架比 `scanf()`/`cin` 的书架大的多，从而显著减少了“把书从图书馆搬到书架”的次数。

这里提供一份使用 fread/fwrite 的代码供参考。

<details><summary><b>fastio class</b> <i>::click to expand</i></summary>

```c++
struct fastio
{
    static const int S=1e7;
    char rbuf[S+48],wbuf[S+48];int rpos,wpos,len;
    fastio() {rpos=len=wpos=0;}
    inline char Getchar()
    {
        if (rpos==len) rpos=0,len=fread(rbuf,1,S,stdin);
        if (!len) return EOF;
        return rbuf[rpos++];
    }
    template <class T> inline void Get(T &x)
    {
        char ch;bool f;T res;
        while (!isdigit(ch=Getchar()) && ch!='-') {}
        if (ch=='-') f=false,res=0; else f=true,res=ch-'0';
        while (isdigit(ch=Getchar())) res=res*10+ch-'0';
        x=(f?res:-res);
    }
    inline void getstring(char *s)
    {
        char ch;
        while ((ch=Getchar())<=32) {}
        for (;ch>32;ch=Getchar()) *s++=ch;
        *s='\0';
    }
    inline void flush() {fwrite(wbuf,1,wpos,stdout);fflush(stdout);wpos=0;}
    inline void Writechar(char ch)
    {
        if (wpos==S) flush();
        wbuf[wpos++]=ch;
    }
    template <class T> inline void Print(T x,char ch)
    {
        char s[20];int pt=0;
        if (x==0) s[++pt]='0';
        else
        {
            if (x<0) Writechar('-'),x=-x;
            while (x) s[++pt]='0'+x%10,x/=10;
        }
        while (pt) Writechar(s[pt--]);
        Writechar(ch);
    }
    inline void printstring(char *s)
    {
        int pt=1;
        while (s[pt]!='\0') Writechar(s[pt++]);
    }
}io;
```

</details>
<br>

需要注意： 因为 `fread()` 和 `fwrite()` 开辟了大数组作为缓冲区一次性搬入很多数据，所以 `fread()`/`fwrite()` 不能和其他使用自带 buffer 的IO方法混用。换句话说，你只能从通过操作自定义的缓冲区的方式来输入输出。