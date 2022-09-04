---
title: "Stanford-CS143 PA1: Lexical Analysis"
linktitle: 'PA1: Lexer'
type: docs
draft: false

math: true

menu:
    stanford-compiler-pa:
        parent: Contents
        weight: 2
---

## Overview

该实验的目的是完成 cool compiler 的 lexical analysis 部分。课堂上讲解了用正则表达式描述词汇格式以及如何根据正则表达式生成 NFA，通过 NFA 转 DFA，再将 DFA 以二维表的形式存储下来。不过现在有成熟的工具可以完成正则表达式的 implementation，所以本实验中我们只需要考虑如何用正则表达式描述 cool 语言的词汇，不需要考虑自动机。

笔者选择了 C++ 版本的实验。本实验中我们使用 flex 工具来完成正则表达式的 implementation。我们识别出一个 token 后需要向 flex 生成的 `yylex()` 函数返回 token 的编号 (在 `cool-parse.h` 中定义)，如果当前 token 有额外的语义信息 (比如字符串，bool const) 则需要在 `coolyyval` 这个 union 的相应字段中填写信息。

顺利完成该实验的重要前提是仔细阅读官方提供的各种手册：

* 阅读 flex manual 的前 10 个 section 以熟悉 flex 的语法。
* 阅读 cool manual 的第 10 章以及 Figure 1 以了解 cool 的语法规则。
* 阅读 PA1 的讲义，该讲义的逻辑不太清晰，各种对返回值的 specification 散落在讲义的各个角落，需要非常仔细地寻找。

## Tokens

cool 的编译器需要识别以下几类 token：

* Integer: 非空的由 0~9 构成的字符串。
* Identifier: identifier 都是由小写/大写字母、数字和下划线构成的非空字符串。type identifier 要求第一个字符必须是小写字母，object identifier 要求第一个字符必须是大写字母。
* Operator: 各种符号，除了 `<=` `<-` `=>` 三个多字符符号有专门的 token id，其余的符号直接返回 ascii 码。
* Keyword: 关键字都有各自定义好的 token id。true 和 false 这两个关键字要求首字母必须小写，其余的关键字都是大小写不敏感的。
* White space: 编译器需要根据 `\n` 正确判断行号，并过滤 `\v` `\r` `\f` `\t` ` ` 几种空白字符。
* String, Comments: 这两个类别比较复杂，下面专门讨论。

### Comments

注释分为 inline comment 和 block comment 两种，写法大同小异。因为我们要吞掉注释中的所有字符，注释中字符的 action 和外面不一样，所以对于 comment 我们最好使用 flex 的 start condition 机制来识别。

当遇到 `(*` 时，进入 COMMENT 状态，在 COMMENT 状态中，我们只需要特殊处理换行符和 `<EOF>`，其余的字符都可以直接吞掉。

### Strings

处理 string 我们最好也使用 start condition 机制。string 中有更多的限制，比如不能有 unescaped 的 `\n`，不能有 `\0`，不能有 `<EOF>`，以及要单独处理特殊字符等。

注意手册中对 string 错误处理的要求：如果一个 string 中有多个错误，只返回第一个错误的相关信息；当一个 string 错误时，下一次扫描应当从这个 string 结束的地方开始 (unescaped `\n` or `\"`)。有的人选择先将整个 string 无脑识别并存储下来，然后对着 char 数组处理各种特殊字符和错误。笔者采取的做法是在遇到 ERROR 后再进入一个 `<STR_ERROR>` 的 start condition，在这种状态中不再字符处理和 error handling，一路扫到 string 的结束。

## Miscellaneous

这里列举一些易漏易错点：

* 要注意识别没有 `(*` 匹配的单独的 `*)`，返回 `unmatched *)`。
* 在 string, comment 等状态中要返回 ERROR 时，不要忘了在返回前将 start condition 重置为 INITIAL，因为我们要完整地处理整个输入，遇到 ERROR 之后剩下的内容也不能摆烂。
* \, n 和 `\n` 是不一样的。在 string 中单独出现的两个字符 (例如 "\n") 应当保留并合并成一个字符，真正的特殊字符是一个不可见字符。
* `\` 之后再跟特殊字符 `\0` 是一个容易忽略的情况。我们通常会对 \0 设置一个 rule 识别，使用正则表达式处理转义字符，需要注意转义字符后面跟的字符也可能是 null character。
* 为了正确地获得 string 的行号，对于 unterminated 的 string 最好使用 `yyless(0)` 将 `\n` 退回，让编译器在 INITIAL 的状态下处理这个换行符，否则 unterminated 的 string 的行号会大1。 

