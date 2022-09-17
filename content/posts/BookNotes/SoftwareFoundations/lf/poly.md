---
title: "Polymorphism and Higher-Order Functions"
linktitle: "04: Poly"
type: docs
draft: false
featured: false

math: true

menu:
    sf-lf:
        parent: Chapters
        weight: 4
---

- [Polymorphism](#polymorphism)
  - [Polymorphic Lists](#polymorphic-lists)
    - [Type Annotation Inference](#type-annotation-inference)
    - [Type Argument Synthesis](#type-argument-synthesis)
    - [Implicit Arguments](#implicit-arguments)
    - [Supplying Type Arguments Explicitly](#supplying-type-arguments-explicitly)
  - [Polymorphic Pairs](#polymorphic-pairs)
  - [Polymorphic Options](#polymorphic-options)
- [Functions as Data](#functions-as-data)
  - [Higher-Order Functions](#higher-order-functions)
  - [Filter](#filter)
  - [Anonymous Functions](#anonymous-functions)
  - [Maps](#maps)
  - [Fold](#fold)
  - [Functions That Construct Functions](#functions-that-construct-functions)
- [Exercises](#exercises)
  - [Exercise: 2 stars, standard, optional (mumble_grumble)](#exercise-2-stars-standard-optional-mumble_grumble)
  - [Exercise: 1 star, standard, optional (combine_checks)](#exercise-1-star-standard-optional-combine_checks)
  - [Exercise: 2 stars, standard, especially useful (split)](#exercise-2-stars-standard-especially-useful-split)
  - [Exercise: 1 star, standard, optional (hd_error_poly)](#exercise-1-star-standard-optional-hd_error_poly)
  - [Exercise: 2 stars, standard (filter_even_gt7)](#exercise-2-stars-standard-filter_even_gt7)
  - [Exercise: 3 stars, standard (partition)](#exercise-3-stars-standard-partition)
  - [Exercise: 3 stars, standard (map_rev)](#exercise-3-stars-standard-map_rev)
  - [Exercise: 2 stars, standard, especially useful (flat_map)](#exercise-2-stars-standard-especially-useful-flat_map)
  - [Exercise: 1 star, standard, optional (fold_types_different)](#exercise-1-star-standard-optional-fold_types_different)
  - [Exercise: 2 stars, standard (fold_length)](#exercise-2-stars-standard-fold_length)
  - [Exercise: 3 stars, standard (fold_map)](#exercise-3-stars-standard-fold_map)
  - [Exercise: 2 stars, advanced (currying)](#exercise-2-stars-advanced-currying)
  - [Church Numerals (Advanced)](#church-numerals-advanced)
    - [Exercise: 2 stars, advanced (church_scc)](#exercise-2-stars-advanced-church_scc)
    - [Exercise: 3 stars, advanced (church_plus)](#exercise-3-stars-advanced-church_plus)
    - [Exercise: 3 stars, advanced (church_mult)](#exercise-3-stars-advanced-church_mult)
    - [Exercise: 3 stars, advanced (church_exp)](#exercise-3-stars-advanced-church_exp)

这一章节我们主要关注函数式编程的两个基本概念：多态 (polymorphism)，即对函数操作的对象的类型进行抽象；高阶函数，即将函数本身看作数据。

## Polymorphism

### Polymorphic Lists

上一章节中我们定义的列表只能处理自然数。在真正的程序中列表中的元素类型是丰富的，甚至可以有 list of lists。我们不可能对每种类型都定义一种列表——因为我们不仅要定义类型，还要将列表相关的所有定理对每种类型都证明一遍，这几乎是不可能的。

基于以上需求，Coq 支持了 polymorphic 的 inductive type definition。比如下面是一个多态的 list 类型：

```plaintext
Inductive list (X : Type) : Type :=
  | nil
  | cons (x : X) (l : list X).
```

多态的本质很简单：我们之前定义的类型以归纳的方式描述了一个集合，现在 list 是一个 Type -> Type 的函数：

```plaintext
Check list: Type -> Type
```

它接收一个类型 X 作为参数，返回 list X 这个类型，“list X” 类型是一个包含了所有由 X 类型元素构成的链表的集合。

list 定义中的类型 X 会自动成为 constructor 的参数——即现在 nil 和 con 都是多态 constructor，以后使用它们时需要在后面写上类型。例如：

```plaintext
Check (cons nat 3 (nil nat)) : list nat.
```

nil 和 cons 的类型也都是从 Type 映射到 list X 的“函数”：

```plaintext
Check nil : forall X : Type, list X.
Check cons: forall X : Type, X -> list X -> list X.
```

有了多态的列表定义后，我们可以把之前写的函数也改成多态的。下面以 repeat 为例：

```plaintext
Fixpoint repeat (X : Type) (x : X) (count : nat) : list X :=
  match count with
  | 0 => nil X
  | S count' => cons X x (repeat X x count')
  end.
```

#### Type Annotation Inference

如果我们在书写函数时不显式声明参数的类型，例如

```plaintext
Fixpoint repeat' X x count : list X :=
  match count with
  | 0 => nil X
  | S count' => cons X x (repeat X x count')
  end.
```

我们会发现 `Check repeat` 和 `Check repeat'` 的结果是完全一样的。这是因为 Coq 可以自动进行 type inference，例如根据 count match 的方法可以看出 count 一定是 nat 类型等等。

#### Type Argument Synthesis

synthesis 和 inference 的原理类似：在某些地方你可以直接用 `_` 来代替类型名，Coq 会结合所有的上下文信息自动分析出这里该填写什么。例如下面两句代码在功能上是等价的：

```plaintext
Definition list123 := cons nat 1 (cons nat 2 (cons nat 3 (nil nat))).
Definition list123' := cons _ 1 (cons _ 2 (cons _ 3 (nil _))).
```

#### Implicit Arguments

利用 implicit argument 技术，我们甚至可以将 `_` 省略掉。

第一种方法是使用 Arguments 语法：Arguments 可以指定一个函数/constructor的名字，以及需要被处理为 implicit 的类型名，以后再使用该函数/constructor时就不需要再显式地给出类型名或类型名的占位符。例如：

```plaintext
Arguments nil {X}.
Arguments cons {X}.
Arguments repeat {X}.
```

这时我们定义 list123 就可以直接写：

```plaintext
Definition list123'' := cons 1 (cons 2 (cons 3 nil)).
```

第二种更方便的语法是：我们在定义 Type/函数的时候，可以用 `{}` 代替 `()` 把用于多态的类型名框起来，这样以后自动省略类型名参数。下面我们用这种语法把 list 的其他几个重要函数重写一下，可以看到使用了 implicit arguments 后，函数体内部的代码就和之前的没有区别了：

```plaintext
Fixpoint app {X : Type} (l1 l2 : list X) : list X :=
  match l1 with
  | nil => l2
  | cons h t => cons h (app t l2)  (* 注意我们还没有对多态的list定义::等notation，所以现在还不能
                                      用 h::t 的方式书写，一会儿定义了之后就可以了*)
  end.

Fixpoint rev {X : Type} (l : list X) : list X :=
  match l with
  | nil => nil
  | cons h t => app (rev t) (cons h nil)
  end.

Fixpoint length {X : Type} (l : list X) : list X :=
  match l with
  | nil => 0
  | cons h t => 1 + (length t)
  end.
```

#### Supplying Type Arguments Explicitly

将参数定义为隐式的一个坏处在于，某些特殊情况下 Coq 可能无法自动识别参数类型。这种情况下我们需要临时地显式给出参数类型。例如：

```plaintext
Fail Definition mynil := nil.
```

（注：这里的 Fail 命令表示如果后面的语句执行失败，Coq 会在交互区打印出错误信息，但程序仍然可以继续执行。）

这样的定义是不成立的，因为完全没有上下文信息可以看出 nil X 的 X 是什么类型。我们有两种解决方法，一种是显式地指定 mynil 的类型：

```plaintext
Definition mynil : list nat := nil.
```

第二种是使用 `@` 让 nil 临时显式地接收一个类型参数：

```plaintext
Definition mynil' := @nil nat.
```

我们定义的多态 list 和之前的 list 本质上是两种类型，因此所有的 Notation 我们也要重新定义一遍：

```plaintext
Notation "x :: y" := (cons x y)
                     (at level 60, right associativity).
Notation "[ ]" := nil.
Notation "[ x ; .. ; y ]" := (cons x .. (cons y []) ..).
Notation "x ++ y" := (app x y)
                     (at level 60, right associativity).
```

### Polymorphic Pairs

在上一章节中我们定义的两个自然数的 pair 也可以拓展成多态的 pair，这就是离散数学中的笛卡尔积：

```plaintext
Inductive prod (X Y : Type) : Type :=
  | pair (x : X) (y : Y).
Argument pair {X} {Y}.

Notation "( x, y )" := (pair x y).
Notation "X * Y" := (prod X Y) : type_scope.
```

这里的 "type_scope" 表明了笛卡尔积只作用在两个类型 (集合) 上，避免和自然数上的乘法混淆。

`(x, y)` 和 `X * Y` 在初期容易混淆。一个好的理解方式是：`X * Y` 是一个类型，即一个集合。如果 `x` 的类型是 `X`，`y` 的类型是 `Y`，那么 `(x, y)` 是类型 `X * Y` 的一个实例 (集合 `X * Y` 中的一个元素)。

下面的函数 combine 接收两个 list，返回一个把对应 index 元素打包成 pair 的 list。这个函数在通常的编程语言中称为 `zip()`：

```plaintext
Fixpoint combine {X, Y : Type} (l1 : list X) (l2 : list Y) -> list (X * Y) :=
  match lx, ly with
  | nil, _ => nil
  | _, nil => nil
  | x :: tx, y :: ty => (x, y) :: (combine tx ty)
  end.
```

### Polymorphic Options

对于之前定义的 natoption，我们也将其转化为多态：

```plaintext
Inductive option (X : Type) : Type :=
  | Some (x : X)
  | None.

Arguments Some {X}.
Arguments None {X}.
```

然后我们可以将 `nth_error` 函数也转化为多态：

```plaintext
Fixpoint nth_error {X : Type} (l : list X) (n : nat) : option X :=
  match l with
  | nil => None
  | a :: l' => match n with
               | 0 => a
               | S n' => nth_error l' n'
               end
  end.
```

## Functions as Data

和多数函数式编程语言一样，Coq 将函数视为 first-class citizen，即函数可以作为函数参数，可以作为函数返回值，可以作为数据存储在数据结构中。

### Higher-Order Functions

下面是一个简单的接受函数作为参数的高阶函数：

```plaintext
Definition doit3times {X : Type} (f : X -> X) (n : X) : X :=
  f (f (f n)).
```

参数 `f` 是一个从 X 映射到 X 的函数，我们可以检查 `doit3times` 的类型：

```plaintext
Check @doit3times : forall X : Type, (X -> X) -> X -> X.
```

### Filter

下面展示一个更有意思的高阶函数：`filter` 函数接收一个函数 test 和一个 list，返回一个新的 list，其中包含原 list 中通过 test 测试的元素：

```plaintext
Fixpoint filter {X : Type} (test : X -> bool) (l : list X) : list X :=
  match l with
  | nil => nil
  | h :: t =>
    if test h then h :: (filter test t)
    else filter test t
  end.
```

多态的 list 可以接收任何类型的元素，甚至是 list 自己，形成 list of lists，下面是一个例子：

```plaintext
Definition length_is_1 {X : Type} (l : list X) : bool :=
  (length l) =? 1.

Example test_filter :
  filter length_is_1 [ [1; 2]; [3]; [4]; [5; 6; 7]; []; [8] ] = [ [3]; [4]; [8]; ].
Proof. reflexivity. Qed.
```

### Anonymous Functions

有的时候，必须给函数命名以方便使用是一件令人沮丧的事情，因为很多时候我们只是想临时创建一个函数丢给高阶函数当参数使用，比如上面例子中的 `length_is_1`。Coq 为我们提供了一种临时创建函数的方式，这种方式类似于 Python 中的 lambda 表达式，不需要给函数命名，该函数也不会出现在 top level environment 中。下面是一个例子：

```plaintext
Example test_filter' :
  filter (fun l => (length l) =? 1)
         [ [1; 2]; [3]; [4]; [5;6;7]; []; [8] ]
  = [ [3]; [4]; [8] ].
```

其中 `(fun l => (length l) =? 1)` 的意思是该函数接收一个参数 `l`，返回 `(length l) =? 1` (我们可以不指明参数和返回值的类型，Coq 会自己做 type inference)。

### Maps

map 也是许多函数式语言支持的常用高阶函数。它接收一个转换函数 f 和一个列表 l，返回一个新列表，其内容是把 f 作用到 l 里的每个元素后的结果。

```plaintext
Fixpoint map {X Y : Type} (f : X -> Y) (l : list X) : list Y :=
  match l with
  | nil => nil
  | h :: t => (f h) :: (map f t)
  end.
```

值得注意的是传入的列表的元素类型和返回的列表的元素类型不一定一样。

我们不仅可以对 list 做 map，也可以对其他的数据类型做 map，比如下面的例子对 option 做 map：

```plaintext
Definition option_map {X Y : Type} (f : X -> Y) (xo : option X) : option Y :=
  match xo with
  | None => None
  | Some x => Some (f x)
  end.
```

### Fold

一个更加强大的高阶函数是 fold，它是 Google map-reduced 框架的 reduce 函数的灵感来源：

```plaintext
Fixpoint fold {X Y : Type} (f : X -> Y -> Y) (l : list X) (b : Y) : Y :=
  match l with
  | nil => b
  | h :: t => f h (fold f t b)
  end.
```

从直观上来理解，该函数接收函数 f，列表 l 和初始值 b。然后从初始值 b 开始，把列表中的元素按照从右往左的顺序依次与 type Y 的结果做函数 f 操作。比如 `fold plus [1; 2; 3; 4] 0` 就意味着 `1+(2+(3+(4+0)))`。

### Functions That Construct Functions

我们之前讨论的高阶函数都是将函数作为参数传入，我们在这里看一些将函数作为函数返回值的例子。下面是一个比较 dummy 的例子，constfun 返回的函数不论接收什么都会输出固定值：

```plaintext
Definition constfun {X : Type} (x : X) : nat -> X :=
  fun (k : nat) => x.
```

事实上，函数式编程本质上不支持多参数的函数，我们之前看到的多参数的函数都是通过 currying 完成的。以 plus 为例：

```plaintext
Check plus : nat -> nat -> nat.
```

这里的 `->` 可以理解为作用在 type 类型元素上的一种二元运算符，它是 right associative 的，即 plus 的类型实际上是 `nat -> (nat -> nat)`。我们使用 plus 时都给它传两个参数，但实际上传一个也是可以的，按照 `nat -> (nat -> nat)`，plus 可以被理解为 “这是一个函数，它接收一个加数，返回另一个函数，这个返回的函数接收另一个加数作为参数，返回加法的结果”。

```plaintext
Definition plus3 := plus 3
Check plus3 : nat -> nat.

Example test_plus3 : plus3 4 = 7.
Proof. reflexivity. Qed.
```

这种给 currying 的函数传一个参数的手法称为 partial application。

## Exercises

### Exercise: 2 stars, standard, optional (mumble_grumble)

```plaintext
d (b a 5)           (* NO, the first argument of d should be a type *)
d mumble (b a 5)    (* YES, grumble mumble *)
d bool (b a 5)      (* YES, grumble bool   *)
e bool true         (* YES, grumble bool   *)
e mumble (b c 0)    (* YES, grumble mumble *)
e bool (b c 0)      (* NO, the second argument shoule be type bool *)
c                   (* YES, mumble *)
```

### Exercise: 1 star, standard, optional (combine_checks)

```plaintext
Check @combine : forall X Y : Type, list X -> list Y -> list (X * Y).
Compute (combine [1; 2] [false; false; true; true]).
  (* [(1, false); (2, false)] *)
```

### Exercise: 2 stars, standard, especially useful (split)

```plaintext
Fixpoint split {X Y : Type} (l : list (X*Y)) : (list X) * (list Y) :=
  match l with
  | nil => (nil, nil)
  | (x, y) :: l' => (x :: fst (split l'), y :: snd (split l'))
  end.

Example test_split:
  split [(1,false);(2,false)] = ([1;2],[false;false]).
Proof. reflexivity. Qed.
```

### Exercise: 1 star, standard, optional (hd_error_poly)

```plaintext
Definition hd_error {X : Type} (l : list X) : option X :=
  match l with
  | nil => None
  | h :: t => Some h
  end.

Check @hd_error : forall X : Type, list X -> option X.

Example test_hd_error1 : hd_error [1;2] = Some 1.
Proof. reflexivity. Qed.
Example test_hd_error2 : hd_error  [[1];[2]]  = Some [1].
Proof. reflexivity. Qed.
```

### Exercise: 2 stars, standard (filter_even_gt7)

```plaintext
Definition filter_even_gt7 (l : list nat) : list nat :=
  filter (fun n => (even n) && (8 <=? n)) l.

Example test_filter_even_gt7_1 :
  filter_even_gt7 [1;2;6;9;10;3;12;8] = [10;12;8].
Proof. reflexivity. Qed.

Example test_filter_even_gt7_2 :
  filter_even_gt7 [5;2;6;19;129] = [].
Proof. reflexivity. Qed.
```

### Exercise: 3 stars, standard (partition)	

```plaintext
Definition partition {X : Type}
                     (test : X -> bool)
                     (l : list X)
                   : list X * list X :=
  ((filter test l), (filter (fun x => negb(test x)) l)).

Example test_partition1: partition odd [1;2;3;4;5] = ([1;3;5], [2;4]).
Proof. reflexivity. Qed.
Example test_partition2: partition (fun x => false) [5;9;0] = ([], [5;9;0]).
Proof. reflexivity. Qed.
```

### Exercise: 3 stars, standard (map_rev)

```plaintext
Lemma app_map : forall (X Y : Type) (f : X -> Y) (l1 l2 : list X),
  map f (l1 ++ l2) = (map f l1) ++ (map f l2).
Proof.
  intros X Y f l1 l2.
  induction l1 as [| n l1' IHl1'].
  - reflexivity.
  - simpl. rewrite IHl1'. reflexivity.
Qed.

Theorem map_rev : forall (X Y : Type) (f : X -> Y) (l : list X),
  map f (rev l) = rev (map f l).
Proof.
  intros X Y f l.
  induction l as [| n l' IHl'].
  - reflexivity.
  - simpl. 
    rewrite <- IHl'.
    rewrite app_map.
    reflexivity.
Qed.
```

在证明主要定理之前，需要先证明一个和列表 append 相关的引理。

### Exercise: 2 stars, standard, especially useful (flat_map)

```plaintext
Fixpoint flat_map {X Y: Type} (f: X -> list Y) (l: list X)
                   : list Y :=
  match l with
  | nil => nil
  | h :: t => (f h) ++ (flat_map f t)
  end.

Example test_flat_map1:
  flat_map (fun n => [n;n;n]) [1;5;4]
  = [1; 1; 1; 5; 5; 5; 4; 4; 4].
Proof. reflexivity. Qed.
```

### Exercise: 1 star, standard, optional (fold_types_different)

fold 参数中的函数 f 的类型是 `f : X -> Y -> Y`，X 和 Y 不同的情况也是很有用的，比如传入的 list 是一系列动作 (上下左右)，b 是一个初始位置，那么使用 fold 函数就可以得出做完这一系列移动后的最终位置。

### Exercise: 2 stars, standard (fold_length)

```plaintext
Theorem fold_length_correct : forall X (l : list X),
  fold_length l = length l.
Proof.
  intros X l.
  induction l as [|n l' IHl'].
  - reflexivity.
  - simpl.
    rewrite <- IHl'.
    reflexivity.
Qed.
```

这段证明有一个值得关注的细节：在 successor case 的最后一步之前，证明的 goal 是

```plaintext
fold_length (n :: l') = S (fold_length l')
```

在这一步使用 `simpl.` 无法化简，但我们知道根据 fold_length 的定义这一步是可以化简的，事实上使用 `reflexivity.` 就可以直接解决问题。reflexivity 使用了更加 aggressive 的化简方法，有时卡壳时可以一试。

### Exercise: 3 stars, standard (fold_map)

```plaintext
Definition fold_map {X Y: Type} (f: X -> Y) (l: list X) : list Y :=
  fold (fun x l => [f x] ++ l) l nil.

Theorem fold_map_correct : forall (X Y : Type) (f : X -> Y) (l : list X),
  fold_map f l = map f l.
Proof.
  intros X Y f l.
  induction l as [| n l' IHl'].
  - reflexivity.
  - simpl.
    rewrite <- IHl'.
    reflexivity.
Qed.
```

### Exercise: 2 stars, advanced (currying)

将一个 `(X * Y) -> Z` 类型的函数转换为 `X -> (Y -> Z)` 类型的函数的过程称为 currying，反过来称为 uncurrying。

```plaintext
Definition prod_uncurry {X Y Z : Type}
  (f : X -> Y -> Z) (p : X * Y) : Z := (f (fst p))(snd p).
```

```plaintext
Theorem uncurry_curry : forall (X Y Z : Type)
                        (f : X -> Y -> Z)
                        x y,
  prod_curry (prod_uncurry f) x y = f x y.
Proof.
  intros X Y Z f x y.
  reflexivity.
Qed.

Theorem curry_uncurry : forall (X Y Z : Type)
                        (f : (X * Y) -> Z) (p : X * Y),
  prod_uncurry (prod_curry f) p = f p.
Proof.
  intros X Y Z f p.
  destruct p as [n m].
  reflexivity.
Qed.
```

`prod_uncurry f` 接收了一个 `X -> Y -> Z` 类型的函数参数，返回了一个 `(X * Y) -> Z ` 类型的参数。`prod_curry` 正好相反。

注意第二个证明中，由于我们定义 `prod_uncurry` 时使用了 `fst` 和 `snd`，所以使用 reflexivity 之前要先用 destruct 把 pair p 给展开。

### Church Numerals (Advanced)

#### Exercise: 2 stars, advanced (church_scc)

```plaintext
Definition scc (n : cnat) : cnat :=
  fun (X : Type) (f : X -> X) (x : X) => n X f (f x).
Example scc_1 : scc zero = one.
Proof. reflexivity. Qed.

Example scc_2 : scc one = two.
Proof. reflexivity. Qed.

Example scc_3 : scc two = three.
Proof. reflexivity. Qed.
```

#### Exercise: 3 stars, advanced (church_plus)

```plaintext
Definition plus (n m : cnat) : cnat :=
  fun (X : Type) (f : X -> X) (x : X) => m X f (n X f x).

Example plus_1 : plus zero one = one.
Proof. reflexivity. Qed.

Example plus_2 : plus two three = plus three two.
Proof. reflexivity. Qed.

Example plus_3 :
  plus (plus two two) three = plus one (plus three three).
Proof. reflexivity. Qed.
```

#### Exercise: 3 stars, advanced (church_mult)

```plaintext
Definition mult (n m : cnat) : cnat :=
  fun (X : Type) (f : X -> X) (x : X) => n X (m X f) x. 

Example mult_1 : mult one one = one.
Proof. reflexivity. Qed.

Example mult_2 : mult zero (plus three three) = zero.
Proof. reflexivity. Qed.

Example mult_3 : mult two three = plus three three.
Proof. reflexivity. Qed.
```

#### Exercise: 3 stars, advanced (church_exp)

```plaintext
Definition exp (n m : cnat) : cnat :=
  fun (X : Type) (f : X -> X) (x : X) => (m (X -> X) (n X) f) x.

Example exp_1 : exp two two = plus two two.
Proof. reflexivity. Qed.

Example exp_2 : exp three zero = one.
Proof. reflexivity. Qed.

Example exp_3 : exp three two = plus (mult two (mult two two)) one.
Proof. reflexivity. Qed.
```

关于 church numeral 可以参考 UCB CS61A 的作业。这里额外再对乘方函数做一点解读，因为其结构比较复杂。

`n X` 的类型是 `(X -> X) -> X -> X`，根据 `->` right associativity 的性质也可以看作 `(X -> X) -> (X -> X)`，即传给它一个 `X -> X` 的函数，它会返回另一个 `X -> X` 的函数。这里我们将 `f` 作为底层参数传给 m，可以认为整个操作提高了一阶，我们不是将一个 `X -> X` 的函数作用在元素 `X` 上 m 次，而是将一个 `(X -> X) -> (X -> X)` 的函数作用在函数 `f` 上 m 次，这个作用函数的语义又恰好是将接收的函数参数重复 n 次，所以就形成了 $1 \to n \to n^2 \to \cdots \to n^m $ 的格局。