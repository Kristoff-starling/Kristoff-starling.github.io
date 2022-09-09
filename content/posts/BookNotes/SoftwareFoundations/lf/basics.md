---
title: "Functional Programming in Coq"
linktitle: "01: Basics"
type: docs
draft: false
featured: false

math: true

menu:
    sf-lf:
        parent: Chapters
        weight: 1
---

- [Introduction](#introduction)
- [Data and Functions](#data-and-functions)
  - [Enumerated Types](#enumerated-types)
  - [Days of the Week](#days-of-the-week)
  - [Booleans](#booleans)
  - [Types](#types)
  - [New Types from Old](#new-types-from-old)
  - [Module](#module)
  - [Tuples](#tuples)
  - [Numbers](#numbers)
- [Proof by Simplification](#proof-by-simplification)
- [Proof by Rewriting](#proof-by-rewriting)
- [Proof by Case Analysis](#proof-by-case-analysis)
- [Fixpoints and Structural Recursion (Optional)](#fixpoints-and-structural-recursion-optional)
- [Exercises](#exercises)
  - [Exercise: 1 star, standard (nandb)](#exercise-1-star-standard-nandb)
  - [Exercise: 1 star, standard (andb3)](#exercise-1-star-standard-andb3)
  - [Exercise: 1 star, standard (factorial)](#exercise-1-star-standard-factorial)
  - [Exercise: 1 star, standard (ltb)](#exercise-1-star-standard-ltb)
  - [Exercise: 1 star, standard (plus_id_exercise)](#exercise-1-star-standard-plus_id_exercise)
  - [Exercise: 1 star, standard (mult_n_1)](#exercise-1-star-standard-mult_n_1)
  - [Exercise: 2 stars, standard (andb_true_elim2)](#exercise-2-stars-standard-andb_true_elim2)
  - [Exercise: 1 star, standard (zero_nbeq_plus_1)](#exercise-1-star-standard-zero_nbeq_plus_1)
  - [Exercise: 2 stars, standard, optional (decreasing)](#exercise-2-stars-standard-optional-decreasing)
  - [Exercise: 1 star, standard (identity_fn_applied_twice)](#exercise-1-star-standard-identity_fn_applied_twice)
  - [Exercise: 1 star, standard (negation_fn_applied_twice)](#exercise-1-star-standard-negation_fn_applied_twice)
  - [Exercise: 3 stars, standard, optional (andb_eq_orb)](#exercise-3-stars-standard-optional-andb_eq_orb)
  - [Exercise: 3 stars, standard (binary)](#exercise-3-stars-standard-binary)

## Introduction

函数式编程 (functional programming) 的思想来自于：如果一个过程或方法没有副作用，那么我们只关心它如何将输入映射到输出，即这个过程/方法是一个数学函数的具体实现。函数式编程的另一个想法是：所有的函数都应当被当作 first-classs value：函数可以作为参数，可以作为返回值，就像普通的数据一样。

## Data and Functions

### Enumerated Types

Coq 的一个显著特点在于：它的内置 features 非常的少。例如除了一些最基本的类型 (如 boolean, integer, strings)，其他的类型都可以由用户自己通过基本类型来定义。

### Days of the Week

```plaintext
Inductive day : Type := 
  | monday
  | tuesday
  | wednesday
  | thursday
  | friday
  | saturday
  | sunday.
```

这段代码定义了一个叫 `day` 的数据类型，它的成员有 7 个。基于这个类型，我们可以书写一些函数：

```plaintext
Definition next_weekday (d : day) : day :=
  match d with
  | monday => tuesday
  | tuesday => wednesday
  | wednesday => thursday
  | thursday => friday
  | friday => monday
  | saturday => monday
  | sunday => monday
  end.
```

这段代码显式地给出了函数参数和返回值的类型。事实上 Coq 支持 type inference，但为了阅读方便这里还是写上了。

定义了函数之后，我们可以利用 `Compute` 命令来计算一些使用了这个函数的表达式，例如：

```plaintext
Compute (next_weekday friday).
(* ==> monday: day *)
```

我们还可以通过 Coq Example 来给出一个 assertion：

```plaintext
Example test_next_weekday:
  (next_weekday (next_weekday saturday)) = tuesday.
```

这样的一个 Example 是需要证明的。一个合法的证明如下 (tactic 的意义见后文)：

```plaintext
Proof.
  simpl.
  reflexivity.
Qed.
```

### Booleans

类似地我们可以定义 bool 类型，以及 negation, and, or 三个运算函数：

```plaintext
Inductive bool : Type :=
  | true
  | false.

Definition negb (b : bool) : bool :=
  match b with
  | true => false
  | false => true
  end.

Definition andb (b1 : bool) (b2 : bool) : bool :=
  match b1 with
  | true => b2
  | false => false
  end.

Definition orb (b1 : bool) (b2 : bool) : bool :=
  match b1 with
  | true => true
  | false => b2
  end.
```

我们可以使用 `Notation` 命令来通过已有的 definitions 定义新符号：

```plaintext
Notation "x && y" := (andb x y).
Notation "x || y" := (orb x y).
```

在 Coq 中我们也可以使用条件分支语句，下面是用条件分支语句描述 and, or, neg 的例子 ( if-then-else 的折行是比较随意的)：

```plaintext
Definition negb' (b : bool) : bool :=
  if b then false
  else true.

Definition andb' (b1 : bool) (b2 : bool) : bool :=
  if b1 then b2 else false.

Definition orb' (b1 : bool) (b2 : bool) : bool :=
  if b1 then 
    true
  else
    b2.
```

Coq 的条件分支语句比一般编程语言的语句的功能更 general 一些：对于只有两个 constructor 的类型，`if X` 可以表示如果 X 等于第一个 constructor，这里只是因为 bool 类型的第一个 constructor 正好是 true 所以看上去和一般编程语言没有区别。

### Types

Coq 中的每个表达式都有类型，我们可以使用 `Check` 命令来打印一个表达式的类型：

```plaintext
Check true.
(* ==> true: bool *)
Check (negb true) : bool.
```

第二种写法相当于一个 assertion，如果类型正确 Coq 不会有反应，如果错误 Coq 会报错。

在 Coq 中，函数也是有类型的：

```plaintext
Check negb : bool -> bool.
```

### New Types from Old

我们之前定义的类型都是 enumerated types，即这些定义显式地列举了一个有穷的元素集合，这些元素被称为 constructor。下面的一个更复杂的例子中出现了 constructor 带有参数的用法：

```plaintext
Inductive rgb : Type :=
  | red
  | blue
  | green.
Inductive color : Type :=
  | black
  | white
  | primary (p : rgb).
```

这里我们引出 constructor expression 的概念：constructor expression 指的是以符合定义的方式将一个 constructor apply 到零个或多个 constructor expression 上。这里的 red, blue, black, white, primary red 等都是 constructor expression。

这里 color 类型描述了属于 color 这个集合的 constructor expression 的三种形式：

* black
* white
* 如果 p 是一个属于 rgb 的 constructor expression，那么 primary p 就是属于 color 的 constructor expression。

我们仍然可以用 pattern matching 的方式来定义函数：

```plaintext
Definition isred (c : color) : bool :=
  match c with
  | black => false
  | white => false
  | pattern red => true
  | pattern _ => false
  end.
```

这里出现了一个新用法：wildcard pattern `_` 是通配符，在这里可以匹配所有不是 red 的其他情形。

### Module

Coq 提供一套 module system：如果我们将一个定义 `foo` 包含在了 `Module X` 和 `End X` 之间，那么在模块外我们想要使用 `foo` 时就要写成 `X.foo` 而不是直接 `foo`。module system 让我们可以不用太担心名字重复的问题。

### Tuples

一个拥有多个参数的 constructor 可以用来表示 tuple，下面的例子展示了一个 nybble (half byte) 类型：

```plaintext
Inductive bit : Type :=
  | B0
  | B1

Inductive nybble: Type :=
  | bits (b0, b1, b2, b3 : bit).
```

同样地我们可以使用 pattern matching 的方式书写函数：

```plaintext
Definition all_zero (nb : nybble) : bool :=	
  match nb with:
  | (bits B0 B0 B0 B0) => true
  | (bits _ _ _ _	) => false
  end.
  
Compute (all_zero(bits B1 B0 B1 B0))
(* ===> false : bool *)
Compute (all_zero(bits B0 B0 B0 B0))
(* ===> true : bool *)
```

### Numbers

之前我们定义的 type 都是有穷集合，而自然数集是一个无穷集合，因此我们在这里使用归纳的方法，根据自然数在朴素集合论中的定义方式给出自然数的定义：

```plaintext
Inductive nat : Type :=
  | O
  | S (n : nat).
```

该定义的意思是：

* `O` 是一个属于 nat 集合的 constructor expression。
* 如果 `n` 是一个属于 nat 集合的 constructor expression，那么 `S n` 也属于 nat 集合。
* 除了以上两条，没有别的 constructor expression 在 nat 集合中。

值得注意的是：这里的 `O` 和 `S` 没有任何实际的含义，我们可以用任意别的字符/单词来替换它们。

下面展示两个函数：`pred` 和 `minustwo`，值得学习的是其中 pattern matching 的方法：

```plaintext
Definition pred (n : nat) : nat :=
  match n with:
  | O => O
  | S n' => n'
  end.

Definition minustwo (n : nat) : nat :=
  match n with:
  | O => O
  | S O => O
  | S (S n') => n'
  end.
```

由于自然数是一种非常普遍的数据类型，所以 Coq 内置了一些解析和打印自然数的小魔法：`O` 会被输出为 0，`S O` 会被输出为 1，依次类推。此外我们给函数传递参数的时候也可以直接使用阿拉伯数字而不是形式化的 `S ... S O`：

```plaintext
Compute (minustwo 4).
(* ===> 2 : nat *)
Check (S (S (S (S O)))).
(* ===> 4 : nat *)
```

我们在定义自然数时使用的 constructor S 和函数 `pred` `minustwo` 在类型上是一样的，

```plaintext
Check S : nat -> nat.
Check pred : nat -> nat.
Check minustwo : nat -> nat.
```

但 `S` 和其他两者有着本质的区别：`pred` 和 `minustwo` 是通过计算规则的方式定义的，例如 `pred 4` 和 3 没有本质区别；但 `S` 只是一个表示数的方式，正如十进制用 0~9 这 10 个字符表示数字一样，在我们的归纳系统中我们使用 S 和 O 这两个字符表示数字，S 本身不包含任何的计算功能。

接下来我们考虑一些更加复杂的关于 number 的函数。例如判断一个数是否是偶数：对于这个问题我们无法通过 pattern matching 的方法直接给出答案，因为偶数有无穷多个。我们只能使用递归的方式：首先规定 O 是偶数，然后对于整数 n 不断 -2 来判断奇偶性。这类需要使用递归的函数应当使用 `Fixpoint` 而不是 `Definition`：

```plaintext
Fixpoint even (n : nat) : bool :=
  match n with
  | O => true
  | S O => false
  | S (S n') => even n'
  end.
```

下面我们来定义加法，这是一个多参数的函数：

```plaintext
Fixpoint plus (n : nat) (m : nat) : nat :=
  match n with
  | O => m
  | S n' => S (plus n' m)
  end.
```

加法的归纳思路十分巧妙，将第一个加数的 S 施加到结果上，直到第一个加数为 0。下面是 Coq 计算 `plus 2 3` 的化简流程：

```plaintext
(*      [plus 3 2]
   i.e. [plus (S (S (S O))) (S (S O))]
    ==> [S (plus (S (S O)) (S (S O)))]
          by the second clause of the [match]
    ==> [S (S (plus (S O) (S (S O))))]
          by the second clause of the [match]
    ==> [S (S (S (plus O (S (S O)))))]
          by the second clause of the [match]
    ==> [S (S (S (S (S O))))]
          by the first clause of the [match]
   i.e. [5]  *)
```

函数的多个参数如果类型一样也可以合并了写，例如下面的这个减法的例子

```plaintext
Fixpoint minus (n m : nat) : nat :=
  match n, m with
  | O   , _ => O
  | _   , O => n
  | S n', S m' => minus n', m'
  end.
```

这个减法的思路也十分巧妙：不断将被减数和减数同时减少，直到有一个是 0 为止。

我们可以为加法、减法、乘法 (代码省略) 添加符号：

```plaintext
Notation "x + y" := (plus x y) (at level 50, left associativity) : nat_scope.
Notation "x - y" := (minus x y) (at level 50, left associativity) : nat_scope.
Notation "x * y" := (mult x y) (at level 40, left associativity) : nat_scope.
```

> **More about Notations**
>
> 这里出现了三个新的内容：
>
> * level 后面可以跟一个 0~100 的数值，这个数值规定了该运算的优先级，数字越小优先级越大。上面的例子中 + 和 - 优先级相同，乘法优先级更高，因此 `a + b * c` 等价于 `a + (b * c)`。
>
> * associativity 有 left, right, no 三种，描述了该符号的结合性。例如 + 是左结合的意味着 `a + b + c` 等价于 `(a + b) + c`。
>
> * 每个符号有其 notation scope。Coq 的解释器会根据上下文自动分析符号的作用域。比如在 `S(0x0)` 中 `x` 会被认定为 nat_scope，而在 `bool x bool` 中 `x` 就会被认定为 type_scope。有些情况下我们需要显式地帮助 Coq 解释器确定 scope，我们可以使用 `%` 语法：例如 `(x+y)%nat`。
>
>     除了符号有 notation scope，数字也可以指定 scope，例如 `0%nat` 和 `0%Z` 一个是自然数 0，一个是整数 0，它们来自不同的标准库。

接下来再展示一个给两个数比大小的函数 (在 Coq 中，一切都得自己定义)：

```plaintext
Fixpoint leb (n m : nat) : bool :=
  match n with
  | O => true
  | S n' =>
      match m with
      | O => false
      | S m' => leb n' m'
      end
  end.
```

当然，我们也可以用像减法那样的语法，这里主要想展示的是 match 的嵌套使用。

```plaintext
Notation "x =? y" := (eqb x y) (at level 70) : nat_scope.
Notation "x <=? y" := (leb x y) (at level 70) : nat_scope.
```

这里区分一下 `=` 和 `=?` ：`x = y` 是一个命题 (proposition)，给出这样一个 claim 是需要证明的；而 `x =? y` 是一个表达式，可以直接计算出 true/false。

## Proof by Simplification

我们已经有了一系列定义和函数，接下来我们可以证明一些定理。

```plaintext
Theorem plus_O_n: forall n : nat, 0 + n = n.
Proof.
  intros n.
  simpl. reflexivity.
Qed.
```

我们对定理和证明中的一些元素做一点说明：

* 在 Coq 中，`Example` `Theorem` `Lemma` `Fact` `Remark` 没有本质区别。
* 在定理中我们使用了全称量词 `forall`。联想人类在证明的时候，我们对于这类定理通常会写下“对于任意自然数n...” 从而在接下来的证明中把 n 当作一个“具体”的数使用。在 Coq 中我们可以通过 `intros` 来完成这一步骤。值得注意的是这里 `intros` 后面其实可以使用任意符号，不一定要和命题中的 n 保持一致。
* 在本证明以及上面的所有证明中，用于化简的 `simpl` 其实都是不需要的，因为 `reflexivity` 自带了化简功能，这里有意显式地写出 `simpl` 是为了能更好地看到证明化简的中间过程。值得一提的是 `reflexivity` 的化简功能比 `simpl` 更强大：它还可以将复杂定义展开。

这里的关键词 `intros` `simpl` `reflexivity` 都是 tactic。tactic 指的是证明过程中用于推进证明过程，检验结果正确的一些命令。

## Proof by Rewriting

下面的这个定理比之前的要更加复杂和有趣一些：

```plaintext
Theorem plus_id_example : forall n m : nat,
  n = m -> 
  n + n = m + m
```

这个定理中出现了蕴含关系符。由于 n 和 m 都是任意整数，我们无法直接通过简单的化简来证明最后的等式。但我们注意到在 `n = m` 的假设下我们可以将等式中的 n 都用 m 来代替。在 Coq 中 `rewrite` 这个 tactic 负责进行这种替换。

```plaintext
Proof.
  intros n m.
  intros H.
  rewrite -> H.
  reflexivity.
Qed.
```

第二行的 `intros H` 表示我们将 `n = m` 这条假设加入到上下文中，并给其命名为 H。第三条语句表示利用假设 H 将目标中的等式左侧的内容替换成等式右侧的内容。`rewrite` 中的 `->` 表示用 RHS 替换 LHS，如果写成 `<-` 则是用 LHS 替换 RHS。

`Check` 命令不仅可以检查类型，还可以打印一个定理的内容。下面两个定理是在标准库中证明过的：

```plaintext
Check mult_n_O.
(* ===> forall n : nat, 0 = n * 0 *)
Check mult_n_Sm.
(* ===> forall n m : nat, n * m + n = n * S m *)
```

`rewrite` tactic 除了可以利用命题中的假设等式进行替换，还可以利用已经证明的定理进行替换，例如：

```plaintext
Theorem mult_n_0_m_0 : forall p q : nat,
  (p * 0) + (q * 0) = 0.
Proof.
  intros p q.
  rewrite <- mult_n_O.
  rewrite <- mult_n_O.
  reflexivity. Qed.
```

证明中连续两次利用 `mult_n_O` 定理进行替换，每次执行替换时 Coq 的解释器会自动在目标等式中寻找定理的 instance。

## Proof by Case Analysis

有的时候我们不得不分情况讨论。这里首先给出一个分情况讨论的证明过程的例子：

```plaintext
Theorem negb_involutive : forall b : bool,
  negb (negb b) = b.
Proof.
  intros b. destruct b eqn:E
  - reflexivity.
  - reflexivity.
Qed.
```

命题中的 `b` 由于是一个不确定的变量，无法直接化简，我们需要分情况讨论 `b = true` 和 `b = false`。

* `destruct` 这个 tactic 用于分类讨论，它可以针对所有 inductively 定义的数据类型使用。使用 `destruct` 相当于分别将 b 看作它定义中的第一，第二……个 constructor，作出 assumption 并创建若干个 subgoal。这里对于 bool 类型相当于分别假设 `b = true` 和 `b = false`。作出假设后，`destruct` 会顺手把假设的内容 "rewrite" 进去。
* `eqn:E` 负责给做出的假设命名，之后可以通过 `E` 来使用这个假设。这一步不是必要的，但这是一个好的习惯。
* `-` 称为 bullet，负责划分各个 subgoal 的证明过程。bullet 不是必要的，如果省略，Coq 会将你的按照 subgoal 的顺序将你的证明过程一一代入。但使用 bullet 会使你的证明过程结构更清晰、更可读。

下面是一个稍复杂的例子：

```plaintext
Theorem plus_1_neq_0_firsttry : forall n : nat,
  (n + 1) =? 0 = false.
Proof.
  intros n.
  simpl. (* does nothing! *)
Abort.
```

我们直接使用 `simpl` 无法化简这个式子。这里值的注意的一点是：如果我们要证明的命题是 `(1 + n) =? 0 = false`，那么 `simpl` 就管用了，因为 `1 + n ` 会被计算成 `S n`，`S n =? 0` 可以直接判定为 false。这里无法计算是因为我们定义的加法是针对第一个参数进行递归的，而第一个参数是一个无法确定具体值的 `n`，所以 `simpl` 无法化简。

我们分类讨论的策略是看 `n` 是 `O` 还是 `S n'`。这里给出证明过程：

```plaintext
Theorem plus_1_neg_0 : forall n : nat,
  (n + 1) ?= 0 = false.
Proof.
  intros n. destruct n as [| n'] eqn:E.
  - reflexivity.
  - reflexivity.
Qed.
```

这里新出现了 `[| n']`。和上一个例子不同，自然数的定义的第二个 constructor 是带参数的，因此我们要用一对中括号 `[]` 给出每一个 constructor 的参数列表，不同 constructor 的参数列表之间用 `|` 隔开 (自然数的第一个 constructor 是 `O` 没有参数，因此 `|` 左边留空)。如果没有用 `[]` 显式地说明参数的名称，Coq 会自动分配参数名，但自动分配的参数名可能会很奇怪，从而影响后续的证明。

分类讨论可以嵌套进行，例如下面的证明 and 运算满足交换律的过程：

```plaintext
Theorem andb_commutative: forall b c : bool, andb b c = andb c b.
Proof.
  intros b c. destruct b eqn Eb.
  - destruct c eqn:Ec.
    + reflexivity.
    + reflexivity.
  - destruct c eqn:Ec.
    + reflexivity.
    + reflexivity.
Qed.
```

不同层级必须使用不同的 bullet。可以使用的 bullet 有 `-` `+` `*` 以及它们的重复版本 (如 `--` `***` 等)。除了 bullet 我们还可以使用大括号来框出证明的层次，大括号是可以嵌套的，例如

```plaintext
{ destruct c eqn:Ec.
  { reflexivity. }
  { reflexivity. } }
```

大括号和 bullet 还可以混合使用：

```plaintext
{ destruct c eqn:Ec.
  - reflexivity.
  - reflexivity. }
```

很多时候我们使用完 `intros` 后立刻就要开始分情况讨论，Coq 为我们准备了更加紧凑方便的语法。下面是两个例子：

```plaintext
Theorem plus_1_neq_0' : forall n : nat,
  (n + 1) =? 0  = false.
Proof.
  intros [|n].
  - reflexivity.
  - refiexivity.
Qed.

Theorem andb_commutative'': forall b c : nat, andb b c = andb c b.
Proof.
  intros [] [].
  - reflexivity.
  - reflexivity.
  - reflexivity.
  - reflexivity.
Qed.
```

这种简洁语法的缺点在于我们无法给分情况讨论时做出的假设等式命名 (即 `eqn:E` 的部分)。

## Fixpoints and Structural Recursion (Optional)

回想加法的定义，我们对第一个加数不断 -1 递归。这样的结构递归保证了我们的参数会越来越小，因此不论输入什么参数计算都一定可以终止。

Coq 要求所有 Fixpoint 类型的函数的参数都要不断变小，从而保证函数一定可以终止。事实上，有的时候存在一些可以终止的合法函数不满足这个性质 (比如如果我们以 5 和 6 为基准定义偶数，那么 0, 1, 2, 3, 4 就得往上加，从而 Coq 报错)。

## Exercises

### Exercise: 1 star, standard (nandb)

```plaintext
Definition nandb (b1:bool) (b2:bool) : bool :=
  match b1, b2 with
  | true, true => false
  | true, false => true
  | false, true => true
  | false, false => true
  end.

Example test_nandb1:               (nandb true false) = true.
Proof. simpl. reflexivity. Qed. 
Example test_nandb2:               (nandb false false) = true.
Proof. simpl. reflexivity. Qed.
Example test_nandb3:               (nandb false true) = true.
Proof. simpl. reflexivity. Qed.
Example test_nandb4:               (nandb true true) = false.
Proof. simpl. reflexivity. Qed.
```

### Exercise: 1 star, standard (andb3)

```plaintext
Definition andb3 (b1:bool) (b2:bool) (b3:bool) : bool :=
  (b1 && b2) && b3.

Example test_andb31:                 (andb3 true true true) = true.
Proof. reflexivity. Qed.
Example test_andb32:                 (andb3 false true true) = false.
Proof. reflexivity. Qed.
Example test_andb33:                 (andb3 true false true) = false.
Proof. reflexivity. Qed.
Example test_andb34:                 (andb3 true true false) = false.
Proof. reflexivity. Qed.
```

注：如果 definition 中没有使用 match 语法，则证明过程中使用 `simpl.` 并不能简化证明过程，这时直接使用 `reflexivity.` 即可。

### Exercise: 1 star, standard (factorial)

```plaintext
Fixpoint factorial (n : nat) : nat :=
  match n with
  | O => S O
  | S n' => mult n (factorial n')
  end.

Example test_factorial1:          (factorial 3) = 6.
Proof. simpl. reflexivity. Qed.
Example test_factorial2:          (factorial 5) = (mult 10 12).
Proof. simpl. reflexivity. Qed.
```

### Exercise: 1 star, standard (ltb)

```plaintext
Definition ltb (n m : nat) : bool :=
  negb (m <=? n).

Notation "x <? y" := (ltb x y) (at level 70) : nat_scope.

Example test_ltb1:             (ltb 2 2) = false.
Proof. reflexivity. Qed.
Example test_ltb2:             (ltb 2 4) = true.
Proof. reflexivity. Qed.
Example test_ltb3:             (ltb 4 2) = false.
Proof. reflexivity. Qed.
```

### Exercise: 1 star, standard (plus_id_exercise)

```plaintext
Theorem plus_id_exercise : forall n m o : nat,
  n = m -> m = o -> n + m = m + o.
Proof.
  intros n m o.
  intros H1.
  intros H2.
  rewrite -> H1.
  rewrite -> H2.
  reflexivity.
Qed.
```

这里需要注意第一条假设使用时的替换方向：如果反过来把等式中的 m 用 n 代替，由于没有 n 和 o 的直接关系，证明就被卡住了。

### Exercise: 1 star, standard (mult_n_1)

```plaintext
Theorem mult_n_1 : forall p : nat,
  p * 1 = p.
Proof.
  intros p.
  rewrite <- mult_n_Sm.
  rewrite <- mult_n_O.
  reflexivity.
Qed.
```

### Exercise: 2 stars, standard (andb_true_elim2)

```plaintext
Theorem andb_true_elim2 : forall b c : bool,
  andb b c = true -> c = true.
Proof.
  intros b c. destruct c eqn:Ec.
  - reflexivity.
  - destruct b eqn:Eb.
    + intros H.
      rewrite <- H.
      reflexivity.
    + intros H.
      rewrite <- H.
      reflexivity.
Qed.
```

本题的一个有意思的点在于如何处理 contradiction：我们对 c 分情况讨论的时候会发现如果 `c = false` 那么假设条件不可能成立，会导出 `false = true`。对于人类来说这就已经结束了，但在 Coq 中我们需要利用这条规则继续做 rewrite (`false = true` 的情况下天地大同了)，直到导出等式两边相等。

### Exercise: 1 star, standard (zero_nbeq_plus_1)

```plaintext
Theorem zero_nbeq_plus_1 : forall n : nat,
  0 =? (n + 1) = false.
Proof.
  intros [|n'].
  - reflexivity.
  - reflexivity.
Qed.
```

这里值得一提的是为何在 `n = S n'` 时 reflexivity 可以直接算出结果。此时我们的 subgoal 是证明 `0 =? (S n' + 1) = false`。根据加法的第二条规则，我们有 `S n' + 1 = S (n' + 1)`。根据 `=?` 的规则又有第一个参数为 0 第二个参数满足 `S n` 形式时可以直接判定为 false。因此 `false = false` 就证出来了。

### Exercise: 2 stars, standard, optional (decreasing)

```plaintext
 Fixpoint error_func (n : nat) : nat :=
  match n with
  | O => error_func (S n)
  | S O => S O
  | S n' => S (error_func n')
  end.
```

### Exercise: 1 star, standard (identity_fn_applied_twice)

```plaintext
Theorem identity_fn_applied_twice :
  forall (f : bool -> bool),
  (forall (x : bool), f x = x) ->
  forall (b : bool), f (f b) = b.
Proof.
  intros f.
  intros H.
  intros b.
  rewrite -> H.
  rewrite -> H.
  reflexivity.
Qed.
```

### Exercise: 1 star, standard (negation_fn_applied_twice)

```plaintext
Theorem negation_fn_applied_twice :
  forall (f : bool -> bool),
  (forall (x : bool), f x = negb x) ->
  forall (b : bool), f (f b) = b.
Proof.
  intros f H b.
  rewrite -> H.
  rewrite -> H.
  destruct b eqn:E.
  - reflexivity.
  - reflexivity.
Qed.
```

### Exercise: 3 stars, standard, optional (andb_eq_orb)	

```plaintext
Theorem andb_eq_orb :
  forall (b c : bool),
  (andb b c = orb b c) ->
  b = c.
Proof.
  intros [].
  - simpl.
    intros c H.
    rewrite -> H.
    reflexivity.
  - simpl.
    intros c H.
    rewrite <- H.
    reflexivity.
Qed.
```

本题最暴力的做法就是对 b, c 的四种情况分类讨论。但这样不是很聪明，根据 `andb` 和 `orb` 的定义，一旦 b 的值确定，表达式的结果是可以用 true, false, c 来表示的，因此我们只要对 b 讨论就可以直接得出 c 的值，这样做简洁很多。

### Exercise: 3 stars, standard (binary)

```plaintext
Fixpoint incr (m:bin) : bin :=
  match m with
  | Z => B1 Z
  | B0 m' => B1 m'
  | B1 m' => B0 (incr m')
  end.

Fixpoint bin_to_nat (m : bin) : nat := 
  match m with
  | Z => O
  | B0 m' => (bin_to_nat m') * 2
  | B1 m' => (bin_to_nat m') * 2 + 1
  end.

Example test_bin_incr1 : (incr (B1 Z)) = B0 (B1 Z).
Proof. reflexivity. Qed.

Example test_bin_incr2 : (incr (B0 (B1 Z))) = B1 (B1 Z).
Proof. reflexivity. Qed.

Example test_bin_incr3 : (incr (B1 (B1 Z))) = B0 (B0 (B1 Z)).
Proof. reflexivity. Qed.

Example test_bin_incr4 : bin_to_nat (B0 (B1 Z)) = 2.
Proof. reflexivity. Qed.

Example test_bin_incr5 :
        bin_to_nat (incr (B1 Z)) = 1 + bin_to_nat (B1 Z).
Proof. reflexivity. Qed.

Example test_bin_incr6 :
        bin_to_nat (incr (incr (B1 Z))) = 2 + bin_to_nat (B1 Z).
Proof. reflexivity. Qed.
```

