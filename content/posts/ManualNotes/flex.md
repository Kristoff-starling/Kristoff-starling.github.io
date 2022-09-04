---
title: "Flex Manual - Notes"
date: 2022-08-14
math: true
summary: 该笔记是对 flex 工具手册精华部分的翻译和解读。flex 是一款基于 C/C++ 的通过正则表达式生成编译器 scanner 代码的工具。

---

## 3. Introduction

flex 是一个生成 scanner 的工具。flex 接受一个 `.flex` 文件，`.flex` 文件用于描述如何生成一个 scanner，由若干 rules 组成，每条规则都是一个正则表达式和一段 C 代码的二元组。flex 工具会输出一个 C 代码 `lex.yy.cc`，该代码与 flex 的 runtime library 链接之后可以生成可执行文件。可执行文件执行时会分析输入的内容，如果某条规则的正则表达式与输入内容匹配则执行 rule 中定义的 C 代码。

## 4. Some Simple Examples

一个最简单的例子如下：

```c++
    int num_lines = 0, num_chars = 0;
%%
\n    ++num_lines; ++num_chars;
.     ++num_chars
%%
```

`.` 表示匹配除了换行符以外的任何字符。上述规则可以统计输入内容中的字符数量和代码行数。

## 5. Format

一个 `.flex` 文件的结构如下，不同 section 由 `%%` 分开：

```
definitions
%%
rules
%%
user code
```

### 5.1 Definitions Section

Definition 区域中的条目服从如下形式：

```
name definition
```

其中 `name` 要求以下划线或字母开头，后面接任意个数的字母，下划线和 `-`。后续的定义中可以用 `{name}` 的方式使用前面的定义，例如

```c++
DIGIT    [0-9]
ID       [a-z][a-z0-9]*
FLOAT    {DIGIT}+"."{DIGIT}*
```

由无缩进的 `/* */` 或 `%{ %}` 包裹住的内容将被原样拷贝到生成的 C 代码中。由 `%top{}` 包裹住的内容将被拷贝到 C 代码的最前面。

### 5.2 Rules Section

Rules 区域中的条目服从如下形式：

```
pattern action
```

其中 `pattern` 是用于识别的模式，`action` 是识别到 pattern 后执行的指令，`pattern` 前不可以有缩进，`action` 必须和 `pattern` 在同一行 (如果 `action` 有多行则第一行需要和 `pattern` 同行)。

在 rules section 的第一条 rule 之前可以用 `%{ %}` 包裹的方式定义一些变量，这些变量可以在 scanning 的过程中使用。

### 5.3 User Code Section

User code section 会被原样拷贝到 C 代码中。这个区域是 optional 的，如果省略的话分割 rules 和 user code 的 `%%` 也可以省略。

### 5.4 Comments in the Input

flex 支持 C 风格的注释，即 `/* */` 之间的内容都会被视为注释，这些注释会被原样拷贝到 C 代码中。注释几乎可以添加在任何地方，除了以下两条限制：

* 注释不能出现在 rules section 中本应该是 pattern 的地方，即 `/* */` 不能无缩进地出现在一行的开头或者出现在 scanner states 之后。
* 注释不能出现在 definitions section 中的 `%option` 行中。

## 6. Patterns

本章节给出了可以用于 rules section 中 pattern 的语法。主要语法和扩展正则表达式相似。

## 7. How the Input Is Matched

如果当前有多条规则 pattern 可以匹配，则 flex 会选择匹配长度最长的规则；如果有多条规则匹配长度相同，则 flex 会选择最先出现的规则 (这两条规则与编译器 lexical analysis 的要求相符)。

如果没有规则可以匹配，那么 flex 会使用默认规则：将第一个字符原样输出。

当 match 决定之后，全局变量 `yytext` (char 指针) 会指向匹配的内容 (token)，`yyleng` (int 变量) 会存储匹配的长度，这些变量可以在 action 的代码中使用。

`yytext` 有两种可以选取的定义方式：一个是 `char *`，一个是 `char []`，在 definitions section 可以通过 `%pointer`/`%array` 来指定 (默认采取指针)。

* 采取指针的好处是更快的速度以及不用担心 token 过长导致的 overflow 问题。
* 采取数组的好处是可以自由地修改 yytext 的内容。数组的大小默认是 `YYLMAX`，在 definitions section 可以通过宏定义的方式自定义这个长度。

## 8. Actions

每条 rule 中的 pattern 都有一个对应的 action，action 可以是一段任意的 C 代码。如果一个 pattern 后的 action 为空，那么这个 pattern 匹配到的输入内容将被忽略。

下面是一个简单的 action 的例子，它可以将连续的多个 whitespace 缩减成一个，并且过滤行末 whitespace：

```c++
%%
[ \t]+    putchar(' ');
[ \t]+$   /* ignore this token */
```

action 中可以书写任意 C 代码，甚至可以使用 `return` 向 `yylex()` 的调用者返回内容。

如果 action 中有 `{`，那么这个 action 是可以跨行的，flex 会把直到与其对应的 `}` 前的所有内容当作本条 rule 的 action。

如果 action 的内容是 `|`，那么它的意思是“该 rule 的 action 和下一条 rule 的 action 相同”。

action 中有一些特殊的命令可以使用：

* `ECHO`：将 yytext 复制到 scanner 的输出中。

* `BEGIN`：后面跟一个 start condition，后面会详细介绍这一用法。

* `REJECT`：使用 REJECT 将会使 scanner 转向下一条 second best 的规则进行匹配。例如：

    ```c++
        int word_count = 0;
    %%
    frob       special(); REJECT;
    [^ \t\n]+  word_count++;
    ```

    这段代码的意思是遇到单词 "frob" 就执行一个特殊的函数 `special()`。在 frob 后使用 REJECT 则会在执行完 `special()` 后使 scanner 转向第二条规则的 pattern 再匹配一次，从而 frob 也会被计入到单词总数中。如果去掉这个 REJECT，"frob" 就只会触发第一条规则，从而单词个数统计出错。

* `yymore()`：告诉 scanner 下一次匹配到某条规则的时候要将 token 追加到本次的 token 后面而不是覆盖，例如：

    ```c++
    %%
    mega-    ECHO; yymore();
    kludge   ECHO;
    ```

    如果 scanner 的输入是 "mega-kludge"，那么输出结果将会是 "mega-mega-kludge"，因为匹配到 kludge 时由于前一次 "mega-" 调用了 `yymore()`，所以 ECHO 的时候将 "kludge" 追加到之前的 "mega-" 后面。

    两条注记：

    * `yymore()` 的正确执行要求用户代码不能随意更改 `yyleng`。
    * `yymore()` 会轻微影响 scanner 匹配的效率。

* `yyless(n)`：将当前 token 除了前 n 个字符以外的后续字符退回到输入流中供 scanner 下次再扫描。例如：

    ```c++
    %%
    foobar    ECHO; yyless(3);
    [a-z]+    ECHO;
    ```

    如果 scanner 的输入是 "foobar"，那么输出结果将会是 "foobarbar"。

    特别地，`yyless(0)` 会将整个 token 退回到输入流中，所以除非 action 中有 BEGIN，调用 `yyless(0)` 将会导致死循环。

* `unput(c)`：将字符 c 放入到输入流中，scanner 下一个扫描到的字符将会是字符 c。例如，下面的代码会将匹配到的 token 左右加上括号退回到输入流中 (注意 `unput(c)` 的语义要求必须要将字符按照倒序退回到输入流中，这样再次扫描得到的才是正序)：

    ```c++
    {
    int i;
    /* Copy yytext because unput() trashes yytext */
    char *yycopy = strdup(yytext);
    unput(')');
    for (i = yyleng - 1; i >= 0; i--)
        unput(yycopy[i]);
    unput('(');
    free(yycopy);
    }
    ```

    使用 `unput()` 需要格外注意的一点是：如果 yytext 使用的是默认的 %pointer，调用 `unput()` 会损坏 yytext 的内容 (每调用一次都会覆盖最右侧的一个字符)，因此如果想要保留 yytext 原本的内容，要么像示例代码一样提前把内容复制出来 (不能只复制指针！)，要么使用 %array 作为 yytext。

    此外，不可以调用 `unput()` 将 `<EOF>` 放入到输入流中。

* `input()`：读取输入流中的下一个字符。例如下面的代码可以吃掉输入中的 C 注释：

    ```c++
    %%
    "/*"    {
            int c;
            for (;;)
            {
                while ((c = input()) != '*' && c != EOF)
                    ; /* eat up text of comment */
                if (c == '*')
                {
                    while ((c = input()) == '*')
                        ;
                    if (c == '/')
                        break; /* found the end */
                }
                if (c == EOF)
                {
                    error("EOF in comment");
                    break;
                }
            }
            }
    ```

    注：如果 scanner 是用 C++ 编译的，那么应当使用 `yyinput()` 以避免函数名与 C++ 自带函数冲突。

## 9. The Generated Scanner

flex 的输出是一个 C 文件 `lex.yy.c`，其中有一个 scanning routine `yylex()`，一些用于匹配 token 的表格和一些其他的辅助 routine 和 macro。默认情况下 `yylex()` 的声明是这样的：

```c++
int yylex()
{
    ... various definitions and actions in here
}
```

通过定义 `YY_DECL` 宏的方式我们可以修改 `yylex()` 的声明，比如

```c++
#define YY_DECL float lexscan(a, b) float a, b;
```

的方式将 `yylex()` 改名为 `lexscan()`，返回值是 float 且接受两个参数 `a` `b`。

当 `yylex()` 被调用后，它会从文件 `yyin` (默认为 stdin) 中扫描 token 并执行 action，当 action 主动返回或者遇到 EOF (此时返回值是 0) 时 `yylex()` 会停止运行，否则会一直执行。

当 `yylex()` 遇到 EOF 时，后续的执行过程是未定义的，除非 `yyin` 指向了新的输入文件或者 `yyrestart()` 被调用。注意调用 `yyrestart()` 不会将 start condition 置为 INITIAL；当 `yylex()` 因 action 返回而终止时，`yylex()` 会再次被调用且从上一次终止的地方继续扫描。

scanner 中的 ECHO 命令会输出到 `yyout` 文件中 (默认为 stdout)，可以通过宏定义的方式来修改。

## 10. Start Conditions

flex 提供一种机制，可以让 rules 条件性地被激活。如果一条 rule 的前缀是 `<sc>`，那么只有当 scanner 处于 "sc" 这个 start condition 的时候该 rule 才处于 active 的状态。

start condition 在 definitions section 中定义。有两种定义方法：`%s sc` 和 `%x sc`，第一种方法是 inclusive 的 start condition，它的意思是没有任何前缀的规则默认也可以被该 sc 激活；第二种方法是 exclusive 的 start condition，它的意思是只有显式地标注了 `<sc>` 前缀的规则才能被 sc 激活。举一个例子，下面的两段代码等价：

```c++
%s example
%%
<example>foo    do_something();
bar             something_else();
```

```c++
%x example
%%
<example>foo             do_something();
<INITIAL, example>bar    something_else();
```

一个特殊的前缀是 `<*>`，它表示所有的 start condition，有这个前缀的规则在任何 start condition 下都处于 active 状态。

在 action 中使用 `BEGIN` 语句可以激活一个 start condition。一个特殊的用法是 `BEGIN(0)` 和 `BEGIN(INITIAL)`，它们的功能相同，都是指激活那些没有前缀的规则。

如果有多条规则使用相同的 sc 前缀，我们可以使用如下语法简化代码的书写：

```c++
<SCs> {
       pattern1    action1;
    pattern2    action2;
    ...
}
    /* equivalent to 
     * <SCs>pattern1    action1
     * <SCs>pattern2    action2
     * ...
     */
```



