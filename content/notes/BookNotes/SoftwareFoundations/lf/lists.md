---
title: "Working with Structured Data"
linktitle: "03: Lists"
type: docs
draft: false
featured: false

math: true

menu:
    sf-lf:
        parent: Chapters
        weight: 3
---

- [Pairs of Numbers](#pairs-of-numbers)
- [Lists of Numbers](#lists-of-numbers)
  - [Repeat](#repeat)
  - [Length](#length)
  - [Append](#append)
  - [Head ant Tail](#head-ant-tail)
  - [Bags via Lists](#bags-via-lists)
- [Reasoning About Lists](#reasoning-about-lists)
  - [Induction on Lists](#induction-on-lists)
  - [Reversing a List](#reversing-a-list)
  - [Search](#search)
- [Options](#options)
- [Partial Maps](#partial-maps)
- [Exercises](#exercises)
  - [Exercise: 1 star, standard (snd_fst_is_swap)](#exercise-1-star-standard-snd_fst_is_swap)
  - [Exercise: 1 star, standard, optional (fst_swap_is_snd)](#exercise-1-star-standard-optional-fst_swap_is_snd)
  - [Exercise: 2 stars, standard, especially useful (list_funs)](#exercise-2-stars-standard-especially-useful-list_funs)
  - [Exercise: 3 stars, advanced (alternate)](#exercise-3-stars-advanced-alternate)
  - [Exercise: 3 stars, standard, especially useful (bag_functions)](#exercise-3-stars-standard-especially-useful-bag_functions)
  - [Exercise: 3 stars, standard, optional (bag_more_functions)](#exercise-3-stars-standard-optional-bag_more_functions)
  - [Exercise: 2 stars, standard, especially useful (add_inc_count)](#exercise-2-stars-standard-especially-useful-add_inc_count)
  - [Exercise: 3 stars, standard (list_exercises)](#exercise-3-stars-standard-list_exercises)
  - [Exercise: 2 stars, standard (eqblist)](#exercise-2-stars-standard-eqblist)
  - [Exercise: 1 star, standard (count_member_nonzero)](#exercise-1-star-standard-count_member_nonzero)
  - [Exercise: 3 stars, advanced (remove_does_not_increase_count)](#exercise-3-stars-advanced-remove_does_not_increase_count)
  - [Exercise: 3 stars, standard, optional (bag_count_sum)](#exercise-3-stars-standard-optional-bag_count_sum)
  - [Exercise: 3 stars, advanced (involution_injective)](#exercise-3-stars-advanced-involution_injective)
  - [Exercise: 2 stars, advanced (rev_injective)](#exercise-2-stars-advanced-rev_injective)
  - [Exercise: 2 stars, standard (hd_error)](#exercise-2-stars-standard-hd_error)
  - [Exercise: 1 star, standard, optional (option_elim_hd)](#exercise-1-star-standard-optional-option_elim_hd)
  - [Exercise: 1 star, standard (eqb_id_refl)](#exercise-1-star-standard-eqb_id_refl)
  - [Exercise: 1 star, standard (update_eq)](#exercise-1-star-standard-update_eq)
  - [Exercise: 1 star, standard (update_neq)](#exercise-1-star-standard-update_neq)

## Pairs of Numbers

我们可以定义一个 pair 类型：

```plaintext
Inductive natprod : Type := 
  | pair (n1 n2 : nat).
Notation "( x, y )" := (pair x y).
```

以及一些辅助函数：

```plaintext
Definition fst (p : natprod) : nat :=
  match p with
  | (x, y) => x
  end.
Definition snd (p : natprod) : nat :=
  match p with
  | (x, y) => y
  end.
Definition swap_pair(p : natprod) : nat :=
  match p with
  | (x, y) => (y, x)
  end.
```

这里值得注意的是在 pattern matching 中的 `(x, y)` 和 `x, y` 是不一样的：前者匹配的是一个 pair 类型，而后者匹配了两个东西。

下面看一个有意思的证明题：

```plaintext
Theorem surjective_pairing : forall p : natprod, p = (fst p, snd p).
```

我们无法通过 `simpl` 直接证明该题，因为 p 的具体内容不知道，`fst p` `snd p` 无法化简。一个好的方法是使用 `destruct`：

```plaintext
Proof.
  intros p. destruct p as [n m].
  simpl. reflexivity.
Qed.
```

这里的 destruct 不是用来分类讨论，而是其字面意思：给 `p` 一个形式，从而解构它。

## Lists of Numbers

我们定义一个链表类型：

```plaintext
Inductive natlist : Type :=
  | nil
  | cons (n : nat) (l : natlist).
```

简单来说，每一个 natlist 类型的变量要么是 nil，要么是一个自然数和另一个 natlist。下面的一些 notation 让表示链表变得简单：

```plaintext
Notation "x :: l" := (cons x l)
                     (at level 60, right associativity).
Notation "[ ]" := nil.
Notation "[ x ; .. ; y]" := (cons x .. (cons y nil) ..).
```

这里值得注意两点：

* `::` 应当是右结合的，即 `1 :: 2 :: 3` 等价于 `1 :: (2 :: 3)`。
* `::` 的优先级低于加减法，即 `1 + 2 :: 3` 等价于 `(1 + 2) :: 3`。

### Repeat

`repeat` 函数接收两个参数 `n` 和 `count`，返回一个长度为 `count`，每个节点都是 `n` 的链表：

```plaintext
Fixpoint repeat (n count : nat) -> natlist :=
  match count with
  | O => nil
  | S count' => n :: (repeat n count')
  end.
```

### Length

`length` 函数接收一个参数 `l`，返回链表 `l` 的长度。

```plaintext
Fixpoint length (l : natlist) -> nat :=
  match l with
  | nil => 0
  | h :: t => 1 + (length t)
  end.
```

### Append

`app` 函数用于把两个链表连接起来。

```plaintext
Fixpoint app (l1 l2 : natlist) -> natlist :=
  match l1 with
  | nil => l2
  | h :: t => h :: (app t l2)
  end.

Notation "x ++ y" := (app x y)
                     (right associativity, at level 60).
```

### Head ant Tail

`hd` 函数返回链表的第一个元素，`tl` 函数返回链条除了第一个元素以外的后面的部分。

```plaintext
Definition hd (default : nat) (l : natlist) -> nat :=
  match hd with
  | nil => default
  | h :: t => h
  end.
Definition tl (l : natlist) -> natlist :=
  match tl with
  | nil => nil
  | h :: t => t
  end.
```

### Bags via Lists

这里的 bag 指的是 multiset。我们可以用链表模拟 multiset。

## Reasoning About Lists

List 的 match 定义让我们可以对于很多比较简单的命题直接使用 `reflexivity` 或者分类讨论解决。但稍复杂的问题还是要借助归纳法。

### Induction on Lists

对于链表的归纳法和对于自然数的归纳法没有本质区别。只要使用 `Inductive` 的方式定义的类型都可以使用归纳法进行证明。下面是 `++` 具有结合律的一段证明：

```plaintext
Theorem app_assoc : forall l1 l2 l3 : natlist,
  (l1 ++ l2) ++ l3 = l1 ++ (l2 ++ l3).
Proof.
  intros l1 l2 l3. induction l1 as [| n l1' IHl'].
  - (* l1 = nil *)
    reflexivity.
  - (* l1 = cons n l1' *)
    simpl. rewrite IHl'. reflexivity.
Qed.
```

### Reversing a List

考虑如下的翻转链表的函数：

```plaintext
Fixpoint rev (l : natlist) : natlist :=
  match l with
  | nil => nil
  | h :: t => (rev t) ++ [h]
  end.
```

下面我们尝试证明链表翻转后长度不变：

```plaintext
Theorem rev_length : forall l : natlist, length (rev l) = length l.
```

如果直接使用数学归纳法证明，在 successor case 中，我们会被 `length(rev l' ++ [n]) = S (length (rev l'))` 这个结论卡住。因此我们需要先证明一个引理：链表结合后的长度等于结合前两个链表的长度之和。

```plaintext
Theorem app_length : forall l1 l2 : natlist, length (l1 ++ l2) = (length l1) + (length l2).
Proof.
  intros l1 l2. induction l1 as [| n l1' IHl1'].
  - reflexivity.
  - simpl. rewrite IHl1'. reflexivity.
Qed.

Theorem rev_length : forall l : natlist, length (rev l) = length l.
Proof.
  intros l. induction l as [| n l' IHl'].
  - reflexivity.
  - simpl. rewrite app_length.
    simpl. rewrite IHl'. rewrite add_comm.
    reflexivity.
Qed.
```

### Search

我们在证明的时候时常要使用之前证明过的引理，但使用这些引理需要知道它们的名字，如果现场不记得名字，可以使用 `Search name` 的方式来查找，Coq 会输出所有名字中带有 name 的引理。

`Search` tactic 还支持更加丰富的查找功能，例如

```plaintext
Search (_ + _ = _ + _) inside Induction. 
```

会在 Induction 模块中寻找符合上述形式的定理。

```plaintext
Search (?x + ?y = ?y + ?x).
```

几乎可以直接锁定到加法交换律。这里变量前加一个 `?` 表示这是搜索时使用的 pattern，而不是在当前环境中要求存在的变量。

## Options

这一章节讲述了 error handling 的重要性。假设我们要写一个函数返回链表的第 n 个元素，我们必须要考虑如果链表的长度小于 n 应当返回什么，如果我们这样写代码：

```plaintext
Fixpoint nth_bad (l : natlist) (n : nat) : nat :=
  match l with
  | nil => 42
  | a :: l' => match n with
               | 0 ⇒ a
               | S n' ⇒ nth_bad l' n'
               end
  end.
```

那么如果函数返回了 42，我们必须再检查一遍链表才能确定究竟是链表太短了还是第 n 个元素正好是 42。一个好的方法是重新定义返回类型，在 error 时返回一个特殊的东西：

```plaintext
Inductive natoption : Type :=
  | Some (n : nat)
  | None.
```

```plaintext
Fixpoint nth_error (l : natlist) (n : nat) : natoption :=
  match l with
  | nil => None
  | a :: l' => match n with
               | O ⇒ Some a
               | S n' ⇒ nth_error l' n'
               end
  end.
```

之后我们还可以根据具体的情境来选择不同的不可能出现的值作为返回值：

```plaintext
Definition option_elim (d : nat) (o : natoption) : nat :=
  match o with
  | Some n' => n'
  | None => d
  end.
```

## Partial Maps

这一章介绍了用 Coq 定义 map 这一数据结构。map 本质上就是 key-value pair 的集合。这里我们先为 key 定义一个单独的类型 (没有什么特殊的作用，只是为了封装一下好看)：

```plaintext
Inductive id : Type :=
  | Id (n : nat)
```

接下来我们定义 partial map:

```plaintext
Inductive partial_map : Type :=
  | empty
  | record (i : id) (v : nat) (m : partial_map)
```

该定义的意思是：partial map 要么是 empty (空集)，要么是一个 key-value pair 接上另一个 partial map。

下面的 update 函数可以完成向 partial map 中添加一个 key-value pair 的功能：

```plaintext
Definition update (d : partial_map) (x : id) (value : nat) -> partial_map :=
  record x value d.
```

find 函数可以查询 partial map 中是否存在某个 id，并返回对应的 value：

```plaintext
Fixpoint find (x : id) (d : partial_map) : natoption :=
  match d with
  | empty => None.
  | y v d' => if eqb_id x y
              then Some v
              else find x d'
  end.s
```



## Exercises

### Exercise: 1 star, standard (snd_fst_is_swap)

```plaintext
Theorem snd_fst_is_swap : forall (p : natprod),
  (snd p, fst p) = swap_pair p.
Proof.
  intros p. destruct p as [n m].
  reflexivity.
Qed.
```

### Exercise: 1 star, standard, optional (fst_swap_is_snd)

```plaintext
Theorem fst_swap_is_snd : forall (p : natprod),
  fst (swap_pair p) = snd p.
Proof.
  intros p. destruct p as [n m].
  reflexivity.
Qed.
```

### Exercise: 2 stars, standard, especially useful (list_funs)

```plaintext
Fixpoint nonzeros (l:natlist) : natlist :=
  match l with
  | nil => nil
  | O :: t => nonzeros t
  | S n :: t => S n :: (nonzeros t)
  end.

Example test_nonzeros:
  nonzeros [0;1;0;2;3;0;0] = [1;2;3].
Proof. reflexivity. Qed.
```

```plaintext
Fixpoint oddmembers (l:natlist) : natlist :=
  match l with
  | nil => nil
  | h :: t => if (even h) then (oddmembers t) else (h :: (oddmembers t))
  end.

Example test_oddmembers:
  oddmembers [0;1;0;2;3;0;0] = [1;3].
Proof. reflexivity. Qed.
```

```plaintext
Definition countoddmembers (l:natlist) : nat :=
  length (oddmembers l).

Example test_countoddmembers1:
  countoddmembers [1;0;3;1;4;5] = 4.
Proof. reflexivity. Qed.

Example test_countoddmembers2:
  countoddmembers [0;2;4] = 0.
Proof. reflexivity. Qed.

Example test_countoddmembers3:
  countoddmembers nil = 0.
Proof. reflexivity. Qed.
```

### Exercise: 3 stars, advanced (alternate)

```plaintext
Fixpoint alternate (l1 l2 : natlist) : natlist :=
  match l1, l2 with
  | nil, nil => nil
  | nil, _ => l2
  | _, nil => l1
  | h1 :: t1, h2 :: t2 => h1 :: h2 :: (alternate t1 t2)
  end.

Example test_alternate1:
  alternate [1;2;3] [4;5;6] = [1;4;2;5;3;6].
Proof. reflexivity. Qed.

Example test_alternate2:
  alternate [1] [4;5;6] = [1;4;5;6].
Proof. reflexivity. Qed.

Example test_alternate3:
  alternate [1;2;3] [4] = [1;4;2;3].
Proof. reflexivity. Qed.

Example test_alternate4:
  alternate [] [20;30] = [20;30].
Proof. reflexivity. Qed.
```

### Exercise: 3 stars, standard, especially useful (bag_functions)

```plaintext
Fixpoint count (v : nat) (s : bag) : nat :=
  match s with
  | nil => 0
  | h :: t => if (v =? h) then 1 + (count v t) else (count v t)
  end.

Example test_count1:              count 1 [1;2;3;1;4;1] = 3.
Proof. reflexivity. Qed.
Example test_count2:              count 6 [1;2;3;1;4;1] = 0.
Proof. reflexivity. Qed.
```

```plaintext
Definition sum : bag -> bag -> bag := app.

Example test_sum1:              count 1 (sum [1;2;3] [1;4;1]) = 3.
Proof. reflexivity. Qed.
```

这里可以注意一下不显式给出参数名称的定义函数的方法。在这种情况下，我们只能把一个参数类型和返回值类型相同的函数赋给当前函数。

```plaintext
Definition add (v : nat) (s : bag) : bag := v :: s.

Example test_add1:                count 1 (add 1 [1;4;1]) = 3.
Proof. reflexivity. Qed.
Example test_add2:                count 5 (add 1 [1;4;1]) = 0.
Proof. reflexivity. Qed.
```

```plaintext
Fixpoint member (v : nat) (s : bag) : bool :=
  match s with
  | nil => false
  | h :: t => if (v =? h) then true else (member v t)
  end.

Example test_member1:             member 1 [1;4;1] = true.
Proof. reflexivity. Qed.
Example test_member2:             member 2 [1;4;1] = false.
Proof. reflexivity. Qed.
```

### Exercise: 3 stars, standard, optional (bag_more_functions)

```plaintext
Fixpoint remove_one (v : nat) (s : bag) : bag :=
  match s with
  | nil => nil
  | h :: t => if (h =? v) then t else (h :: (remove_one v t))
  end.

Example test_remove_one1:
  count 5 (remove_one 5 [2;1;5;4;1]) = 0.
Proof. reflexivity. Qed.

Example test_remove_one2:
  count 5 (remove_one 5 [2;1;4;1]) = 0.
Proof. reflexivity. Qed.

Example test_remove_one3:
  count 4 (remove_one 5 [2;1;4;5;1;4]) = 2.
Proof. reflexivity. Qed.

Example test_remove_one4:
  count 5 (remove_one 5 [2;1;5;4;5;1;4]) = 1.
Proof. reflexivity. Qed.
```

```plaintext
Fixpoint remove_all (v:nat) (s:bag) : bag :=
  match s with
  | nil => nil
  | h :: t => if (h =? v) then (remove_all v t) else (h :: (remove_all v t))
  end.

Example test_remove_all1:  count 5 (remove_all 5 [2;1;5;4;1]) = 0.
Proof. reflexivity. Qed.
Example test_remove_all2:  count 5 (remove_all 5 [2;1;4;1]) = 0.
Proof. reflexivity. Qed.
Example test_remove_all3:  count 4 (remove_all 5 [2;1;4;5;1;4]) = 2.
Proof. reflexivity. Qed.
Example test_remove_all4:  count 5 (remove_all 5 [2;1;5;4;5;1;4;5;1;4]) = 0.
Proof. reflexivity. Qed.
```

```plaintext
Fixpoint included (s1 : bag) (s2 : bag) : bool :=
  match s1 with
  | nil => true
  | h :: t => if ((count h s2) =? 0) then
                false
              else
                included t (remove_one h s2)
  end.

Example test_included1:              included [1;2] [2;1;4;1] = true.
Proof. reflexivity. Qed.
Example test_included2:              included [1;2;2] [2;1;4;1] = false.
Proof. reflexivity. Qed.
```

### Exercise: 2 stars, standard, especially useful (add_inc_count)

```plaintext
Theorem add_inc_count : forall (v : nat) (s : bag),
  count v (add v s) = 1 + count v s.
Proof.
  intros. 
  simpl.
  rewrite eqb_refl. (* n =? n = true *)
  reflexivity.
Qed.
```

### Exercise: 3 stars, standard (list_exercises)

```plaintext
Theorem app_nil_r : forall l : natlist,
  l ++ [] = l.
Proof.
  intros l. induction l as [| n l' IHl'].
  - reflexivity.
  - simpl. rewrite IHl'. reflexivity.
Qed.
```

```plaintext
Theorem rev_app_distr: forall l1 l2 : natlist,
  rev (l1 ++ l2) = rev l2 ++ rev l1.
Proof.
  intros l1 l2.
  induction l1 as [|n l1' IHl1'].
  - rewrite app_nil_r. reflexivity.
  - simpl. 
    rewrite IHl1'. 
    rewrite app_assoc. 
    reflexivity.
Qed.
```

```plaintext
Theorem rev_involutive : forall l : natlist,
  rev (rev l) = l.
Proof.
  intros l.
  induction l as [| n l' IHl'].
  - reflexivity.
  - simpl. 
    rewrite rev_app_distr.
    rewrite IHl'.
    reflexivity.
Qed.
```

```plaintext
Theorem app_assoc4 : forall l1 l2 l3 l4 : natlist,
  l1 ++ (l2 ++ (l3 ++ l4)) = ((l1 ++ l2) ++ l3) ++ l4.
Proof.
  intros l1 l2 l3 l4.
  rewrite app_assoc.
  rewrite app_assoc.
  reflexivity.
Qed.
```

```plaintext
Lemma nonzeros_app : forall l1 l2 : natlist,
  nonzeros (l1 ++ l2) = (nonzeros l1) ++ (nonzeros l2).
Proof.
  intros l1 l2.
  induction l1 as [| n l1' IHl1'].
  - reflexivity.
  - destruct n as [| n'] eqn:N.
    + simpl. rewrite IHl1'. reflexivity.
    + simpl. rewrite IHl1'. reflexivity.
Qed.
```

### Exercise: 2 stars, standard (eqblist)

```plaintext
Fixpoint eqblist (l1 l2 : natlist) : bool :=
  match l1, l2 with
  | nil, nil => true
  | nil, _ => false
  | _, nil => false
  | h1 :: t1, h2 :: t2 =>
    if (h1 =? h2) then (eqblist t1 t2) else false
  end.

Example test_eqblist1 :
  (eqblist nil nil = true).
Proof. reflexivity. Qed.

Example test_eqblist2 :
  eqblist [1;2;3] [1;2;3] = true.
Proof. reflexivity. Qed.

Example test_eqblist3 :
  eqblist [1;2;3] [1;2;4] = false.
Proof. reflexivity. Qed.
```

```plaintext
Theorem eqblist_refl : forall l:natlist,
  true = eqblist l l.
Proof.
  intros l.
  induction l as [| n l' IHl'].
  - (* l = nil *)
    reflexivity.
  - (* l = con n l' *)
    simpl. 
    rewrite eqb_refl. (* n =? n = true *)
    rewrite IHl'.
    reflexivity.
Qed.
```

### Exercise: 1 star, standard (count_member_nonzero)

```plaintext
Theorem count_member_nonzero : forall (s : bag),
  1 <=? (count 1 (1 :: s)) = true.
Proof.
  intros s.
  reflexivity.
Qed.
```

### Exercise: 3 stars, advanced (remove_does_not_increase_count)

```plaintext
Theorem remove_does_not_increase_count: forall (s : bag),
  (count 0 (remove_one 0 s)) <=? (count 0 s) = true.
Proof.
  intros s.
  induction s as [| n s' IHs'].
  - reflexivity.
  - destruct n as [| n'] eqn:N.
    + simpl.
      rewrite leb_n_Sn.
      reflexivity.
    + simpl.
      rewrite IHs'.
      reflexivity.
Qed.
```

### Exercise: 3 stars, standard, optional (bag_count_sum)

```plaintext
Theorem count_sum : forall (v : nat) (l1 l2 : bag),
  count v l1 + count v l2 = count v (sum l1 l2).
Proof.
  intros v l1 l2.
  induction l1 as [| n l1' IHl1'].
  - reflexivity.
  - simpl.
    destruct (v =? n) as [].
    + simpl. rewrite IHl1'. reflexivity.
    + rewrite IHl1'. reflexivity.
Qed.
```

这题一个值得注意的细节是：我们不方便利用 destruct 直接去讨论 n 是否等于 v，但我们可以通过讨论 `v =? n` 是 true 还是 false 来实现这一需求。

### Exercise: 3 stars, advanced (involution_injective)

```plaintext
Theorem involution_injective : forall (f : nat -> nat),
    (forall n : nat, n = f (f n)) -> (forall n1 n2 : nat, f n1 = f n2 -> n1 = n2).
Proof.
  intros f H1 n1 n2 H2.
  rewrite H1.
  rewrite <- H2.
  rewrite <- H1.
  reflexivity.
Qed.
```

### Exercise: 2 stars, advanced (rev_injective)

```plaintext
Theorem rev_injective : forall (l1 l2 : natlist),
  rev l1 = rev l2 -> l1 = l2.
Proof.
  intros l1 l2 H.
  rewrite <- rev_involutive.
  rewrite <- H.
  rewrite rev_involutive.
  reflexivity.
Qed.
```

该定理的证明具有一定的技巧性。如果直接对某个链表使用归纳法会比较繁琐。这里利用 involutive 定理 `l = rev (rev l)`先把 `l2` 换成 `rev (rev l2)`，再利用条件把 `rev l2` 换成 `rev l1`，最后换一个方向利用 involutive 完成证明。

### Exercise: 2 stars, standard (hd_error)

```plaintext
Definition hd_error (l : natlist) : natoption :=
  match l with
  | nil => None
  | h :: t => (Some h)
  end.

Example test_hd_error1 : hd_error [] = None.
Proof. reflexivity. Qed.

Example test_hd_error2 : hd_error [1] = Some 1.
Proof. reflexivity. Qed.

Example test_hd_error3 : hd_error [5;6] = Some 5.
Proof. reflexivity. Qed.
```

### Exercise: 1 star, standard, optional (option_elim_hd)

```plaintext
Theorem option_elim_hd : forall (l:natlist) (default:nat),
  hd default l = option_elim default (hd_error l).
Proof.
  intros l d.
  destruct l as [| h l'] eqn:L.
  - reflexivity.
  - reflexivity.
Qed.
```

### Exercise: 1 star, standard (eqb_id_refl)

```plaintext
Theorem eqb_id_refl : forall x, eqb_id x x = true.
Proof.
  intros x.
  destruct x as [n].
  simpl.
  rewrite eqb_refl.
  reflexivity.
Qed.
```

### Exercise: 1 star, standard (update_eq)

```plaintext
Theorem update_eq :
  forall (d : partial_map) (x : id) (v: nat),
    find x (update d x v) = Some v.
Proof.
  intros d x v.
  simpl.
  destruct x as [x'].
  simpl.
  rewrite eqb_refl.
  reflexivity.
Qed.
```

### Exercise: 1 star, standard (update_neq)

```plaintext
Theorem update_neq :
  forall (d : partial_map) (x y : id) (o: nat),
    eqb_id x y = false -> find x (update d y o) = find x d.
Proof.
  intros d x y o H.
  simpl.
  rewrite H.
  reflexivity.
Qed.
```
