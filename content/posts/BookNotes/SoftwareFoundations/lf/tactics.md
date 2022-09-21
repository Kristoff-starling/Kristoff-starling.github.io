---
title: "More Basic Tactics"
linktitle: "05: Tactics"
type: docs
draft: false
featured: false

math: true

menu:
    sf-lf:
        parent: Chapters
        weight: 5
---

- [The `apply` Tactic](#the-apply-tactic)
- [The `apply with` Tactic](#the-apply-with-tactic)
- [The `injection` and `discriminate` Tactics](#the-injection-and-discriminate-tactics)
- [Using Tactics on Hypotheses](#using-tactics-on-hypotheses)
- [Varying the Induction Hypothesis](#varying-the-induction-hypothesis)
- [Unfolding Definitions](#unfolding-definitions)
- [Using `destruct` on Compound Expressions](#using-destruct-on-compound-expressions)
- [Exercises (tbd)](#exercises-tbd)
  - [Exercise: 2 stars, standard, optional (silly_ex)](#exercise-2-stars-standard-optional-silly_ex)
  - [Exercise: 2 stars, standard (apply_exercise1)](#exercise-2-stars-standard-apply_exercise1)
  - [Exercise: 3 stars, standard, optional (trans_eq_exercise)](#exercise-3-stars-standard-optional-trans_eq_exercise)
  - [Exercise: 3 stars, standard (injection_ex3)](#exercise-3-stars-standard-injection_ex3)
  - [Exercise: 1 star, standard (discriminate_ex3)](#exercise-1-star-standard-discriminate_ex3)
  - [Exercise: 2 stars, standard (eqb_true)](#exercise-2-stars-standard-eqb_true)
  - [Exercise: 3 stars, standard, especially useful (plus_n_n_injective)](#exercise-3-stars-standard-especially-useful-plus_n_n_injective)
  - [Exercise: 3 stars, standard, especially useful (gen_dep_practice)](#exercise-3-stars-standard-especially-useful-gen_dep_practice)
  - [Exercise: 3 stars, standard (combine_split)](#exercise-3-stars-standard-combine_split)
  - [Exercise: 2 stars, standard (destruct_eqn_practice)](#exercise-2-stars-standard-destruct_eqn_practice)
  - [Exercise: 3 stars, standard (eqb_sym)](#exercise-3-stars-standard-eqb_sym)
  - [Exercise: 3 stars, standard, optional (eqb_trans)](#exercise-3-stars-standard-optional-eqb_trans)

## The `apply` Tactic

我们在证明过程中经常遇到某个 goal 正好是假设/某个之前已经证明过的定理的情况。这时候写一步 `rewrite` 再写一步 `reflexivity` 显得略为繁琐，我们可以用 `apply` 这个 tactic 来一步到位：

```plaintext
Theorem silly1 : forall n m : nat, n = m -> n = m.
Proof.
  intros n m eq.
  apply eq.
Qed.
```

当 `apply` 应用于一个蕴含式时，Coq 会把蕴含式的条件设置成一个新的 goal。如果 `apply` 后面跟的表达式中有全称量词，Coq 可以自动将其实例化：

```plaintext
Theorem silly2a : forall (n m : nat),
  (n,n) = (m,m) ->
  (forall (q r : nat), (q,q) = (r,r) -> [q] = [r]) ->
  [n] = [m].
Proof.
  intros n m eq1 eq2.
  apply eq2. apply eq1. Qed.
```

在使用 `apply` 时，被 applied 的表达式必须和 goal 完全匹配，比如 `n = m` 和 `m = n` 就不算完全匹配。但这种情况我们可以通过 `symmetry` 这个 tactic 来调整等号左右的位置：

```plaintext
Theorem silly3 : forall n m : nat, n = m -> m = n.
Proof.
  intros n m H.
  symmetry.
  apply H.
Qed.
```

## The `apply with` Tactic

考虑如下例子：假设我们已经有了这样一个引理：

```plaintext
Lemma trans_eq : forall (X : Type) (n m o : X),
  n = m -> m = o -> n = o.
Proof.
  intros X n m o eq1 eq2.
  rewrite eq1.
  rewrite eq2.
  reflexivity.
Qed.
```

现在我们希望证明如下命题：

```plaintext
Example trans_eq_example : forall (a b c d e f : nat),
  [a;b] = [c;d] ->
  [c;d] = [e;f] ->
  [a;b] = [e;f].
```

如果我们想要用 `apply` 直接使用之前的引理，Coq 会根据 goal 自动进行实例化，从而得到 `X` 是 `list nat`，`n` 是 `[a;b]`，`o` 是 `[e;f]`，但 `apply` 无法从 goal 推断出 `m` 应该取什么。所以我们要用 `apply with` tactic 显式地把 `m` 的值提供给 Coq：

```plaintext
Proof.
  intros a b c d e f eq1 eq2.
  apply trans_wq with (m:=[c;d]).
  apply eq1. apply eq2.
Qed.
```

事实上我们可以省略 `m`，即直接写 `apply trans_eq with [c;d].`Coq 可以智能地判定我们提供的值是谁的。

除此之外，Coq 有一个名叫 `transtivity` 的 tactic 专门负责做这项工作，所以实际上我们不需要证明 `trans_eq` 这个引理，直接用这个 tactic，同时显式地给出中间变量的取值即可：

```plaintext
Proof.
  intros a b c d e f eq1 eq2.
  transtivity [c;d].
  apply eq1. apply eq2.
Qed.
```

## The `injection` and `discriminate` Tactics

考虑自然数的定义：

```plaintext
Inductive nat : Type :=
  | O
  | S (n : nat).
```

从这个定义中我们除了知道自然数唯二两种可能的形式，还可以看出两条性质：

* constructor `S` 是 injective (one-to-one) 的，即若 `S n = S m`，则有 `n = m`。
* `O` 和 `S` 这两个 constructor 是 disjoint 的，即对于任意 n，`O` 不等于 `S n`。

上面两条性质其实可以推广到任何归纳定义的类型上，即一个类型中所有的 constructor 都是 injective 的；不同的 constructor 产生的 value 不可能相等。

我们先考虑前者，`forall n m : nat, S n = S m -> n = m.` 这个定理可以通过归纳法证明，事实上所有 constructor 的 injectivity 都可以通过类似的归纳法证明，因此 Coq 提供了 `injection` 这个 tactic。我们看一个使用 tactic 的例子 (注意：`injection` 只能用在 constructor 的 injectivity 上，不能用于一般的单射！)：

```plaintext
Theorem injection_ex : forall (n m o : nat), [n;m] = [o;o] -> n = m.
Proof.
  intros n m o H.
  injective H as H1 H2.
  rewrite H1. rewrite H2. reflexivity.
Qed.
```

`injection H as H1 H2.` 表示将 `H` 这个表达式中所有可以通过 injectivity 得到的结论全部做成当前上下文的 hypothesis，分别命名为 `H1` `H2`。这里 `H : [n;m] = [o;o]`，完全展开就是 `cons n (cons m nil) = cons o (cons o nil)`。根据 injectivity 我们可以知道 `n = o` 和 `cons m nil = cons o nil`，再次利用 injectivity 可知 `m = o`，因此 Coq 的 `injection` 可以帮我们自动分析出两条结论。

我们再来讨论后者。disjointness 告诉我们如果我们的 hypothesis 中出现了一个定义中的两个不同的 constructor 的 value 相等，那么就可以由条件假推出一切真，这也被称为 principle of explosion。在 Coq 中我们可以使用 `discrimate` 这个 tactic 来完成这件事，只要我们指定一个确定是错误的假设 (不一定是 disjointness 的这种，任何错误的表达式都可以)，就可以直接证完当前结论。下面是一个例子：

```plaintext
Theorem discriminate_ex : forall (n : nat), S n = O -> 2 + 2 = 5.
Proof.
  intros n contra.
  discriminate contra.
Qed.
```

injectivity 告诉我们 `S n = S m -> n = m`。这件事情反过来也是成立的，即如下定理：

```plaintext
Theorem f_equal : forall (A B : Type) (f : A -> B) (x y : A),
  x = y -> f x = f y.
```

该定理用 `rewrite` 很好证。事实上 Coq 为我们提供了 `f_equal` 这个 tactic，我们不需要证明该定理就可以使用这个结论，下面是一个例子：

```plaintext
Theorem eq_implies_succ_equal : forall (n m : nat),
  n = m -> S n = S m.
Proof.
  intros n m H.
  f_equal.
  apply H.
Qed.
```

如果当前的 goal 形如 `f a1 a2 ... an = g b1 b2 ... bn`，则 `f_goal` 会自动将其转化为 `f = g` `a1 = b1` ... `an = bn` 这些 subgoal，如果其中有 trivial 的 (比如两个函数相同，对应位置的两个参数相同) `f_equal` 还会自动过滤到它们。

## Using Tactics on Hypotheses

默认情况下，大部分的 tactic 都是在保持 context 不变的情况下针对 goal 进行的。事实上 Coq 提供了 "in" 的语法，可以让 tactic 作用在 context 中的 hypothesis 上，例如 `simpl`：

```plaintext
Theorem S_inj : forall (n m : nat) (b : bool),
  ((S n) =? (S m)) = b  ->
  (n =? m) = b.
Proof.
  intros n m b H. simpl in H. apply H.  Qed.
```

再比如 `symmetry` 和  `apply`：

```plaintext
Theorem silly4 : forall (n m p q : nat),
  (n = m -> p = q) ->
  m = n ->
  q = p.
Proof.
  intros n m p q EQ H.
  symmetry in H. apply EQ in H.
  symmetry in H. apply H.
Qed.
```

值得注意的是 `apply` 和 `apply in` 的区别：

* `apply in` 采用的是 forward reasoning，即 `apply L in H` 的功能是：如果 L 是蕴含式 `X -> Y`，H 是 `X`，则我们可以转而去证明 `Y`。
* `apply` 采用的是 backward reasoning，即 `apply H` 的功能是：如果 H 是蕴含式 `X -> Y`，当前的结论是 `Y`，则我们只要能证明 `X` 就足已完成证明。

人类的 informal proof 倾向于 forward reasoning，但 Coq 等证明工具通常更多采用 backward reasoning。

## Varying the Induction Hypothesis

该章节介绍了两段具有一定技巧性的证明。有时候，不管三七二十一把所有的变量和假设全部都 `intros` 了反而会使证明卡住。考虑如下证明 `double()` 是单射的定理：

```plaintext
Theorem double_injective : forall n m,
  double n = double m -> n = m.
```

如果我们先把 `n` `m` 都 `intros` 了再对 `n` 进行归纳，则会得到

```plaintext
Proof.
  intros n m. induction n as [| n' IHn'].
  - (* n = 0 *) simpl. intros eq.
    destruct m as [| m'] eqn:E.
    + reflexivity.
    + discriminate eq.
  - (* n = S n' *) intros eq.
    destruct m as [| m'] eqn:E.
    + discriminate eq.
    + apply f_equal.
Abort.
```

(注：两次 `discriminate` 的等式 `eq` 均为 `double (S n') = double 0`。这里可以根据 disjointment 直接 pass 的原因是我们可以根据 double 的定义将式子化简为 `S (S (double n')) = 0`，在自然数的两个 constructor 下触发 disjointment。)

在 successor case 中 `m = S m'` 的情况下，我们需要证明 `n' = m'`，而我们的归纳假设是 `double n' = double (S m') -> n' = S m'`，这和我们要证明的结论没什么联系，从而证明卡住。出现这个问题的原因是：在归纳假设和归纳目标的式子中我们的 `m` 需要取到不同的值，但我们在开始归纳之前过早地 `intros m` 导致 `m` 只能是一个具体的值。正确的证明过程应当先对 `n` 进行归纳，保证归纳假设中是 `forall m` 而不是一个具体的 `m`，这样在 successor case 时我们就可以让归纳假设中的 `m` 实例化为具体需要的变量，从而完成证明。

```plaintext
Theorem double_injective : forall n m,
  double n = double m -> n = m.
Proof.
  intros n. induction n as [| n' IHn'].
  - (* n = 0 *) simpl. intros m eq.
    destruct m as [| m'] eqn:E.
    + reflexivity.
    + discriminate eq.
  - intros m eq.
    destruct m as [| m'] eqn:E.
    + discriminate eq.
    + apply f_equal.
      apply IHn'    (* IHn'中是forall m，apply，可以完成关键的实例化 *)
      simpl in eq.
      injection eq as goal.
      apply goal.
Qed.
```

刚才的例子展示了 fewer intros 的技巧，但有的时候我们发现我们不得不对全称量词中的变量顺序进行调换。比如刚才的定理，如果我们想对 `m` 进行归纳就会遇到一些困难：

```plaintext
Proof.
  intros m. induction m as [| m' IHm'].
```

我们会发现 `intros m.` 时变量 `n` 也被实例化了，因为在 forall 中 `n` 出现在 `m` 前面。在不修改定理描述的前提下，我们可以使用 `generalize dependent` 这个 tactic 来将某个变量“去实例化”：

```plaintext
Proof.
  intros n m. generalize dependent n.
  ...
```

执行完 `generalize dependent n` 后，变量 `n` 又会回到 forall 中。

## Unfolding Definitions

有的时候我们需要手动将某个函数的定义展开，从而将证明推进下去。比如我们定义了平方函数：

```plaintext
Definition square n := n * n.
```

并且试图证明定理

```plaintext
Lemma square_mult : forall n m, square (n * m) = (square n) * (square m)
Proof.
  intros n m.
  simpl.  (* get stuck! *)
```

`simpl.` 并不能帮助我们化简。这时候我们可以使用 `unfold square` 把定义展开，这样再使用乘法的交换律、结合律就可以完成证明。

这里我们把 `unfold` 和 `simpl` `reflexivity` 的定义展开功能做一个对比。`simpl` 和 `reflexivity` 也有一定的定义展开功能，但不是很强。例如：

```plaintext
Definition foo (x : nat) := 5.
Fact silly_fact_1 : forall m, foo m + 1 = foo (m + 1) + 1.
Proof.
  intros m.
  reflexivity.
Qed.
```

`reflexivity` 可以帮我们自动把 foo 这个常值函数展开，直接完成证明。但如果函数的结构稍微复杂一点 `simpl` 和 `reflexivity` 就无能为力了：

```plaintext
Definition bar x :=
  match x with
  | O => 5
  | S _ => 5
  end.
Fact silly_fact_2_FAILED : forall m, bar m + 1 = bar (m + 1) + 1.
Proof.
  intros m.
  simpl.
Abort.
```

虽然我们人眼可以看出来不管自然数 x 走哪个分支 `bar x` 的结果都是 5，但 Coq 没有这么强的分析能力。这种情况我们只能老老实实对着 x 做 `destruct`。

## Using `destruct` on Compound Expressions

在 Coq 中，`destruct` 的对象可以是任意的表达式，只要该表达式的结果是归纳定义的，`destruct` 就会对其每种 constructor 分类讨论。

另外值得注意的一点是：当我们对一个表达式做 `destruct` 时，通常 `eqn` 是必不可少的，因为表达式的取值应当作为 context 的一部分参与证明，如果只是让 Coq 做纯代入，我们可能会丢失必要的信息。

## Exercises (tbd)

### Exercise: 2 stars, standard, optional (silly_ex)

```plaintext
Theorem silly_ex : forall p,
  (forall n, even n = true -> even (S n) = false) ->
  (forall n, even n = false -> odd n = true) ->
  even p = true ->
  odd (S p) = true.
Proof.
  intros p H1 H2 H3.
  apply H2.
  apply H1.
  apply H3.
Qed.
```

### Exercise: 2 stars, standard (apply_exercise1)

```plaintext
Theorem rev_exercise1 : forall (l l' : list nat),
  l = rev l' ->
  l' = rev l.
Proof.
  intros l l' H.
  replace l' with (rev (rev l')).
  - rewrite H. reflexivity.
  - apply rev_involutive.
Qed.
```

### Exercise: 3 stars, standard, optional (trans_eq_exercise)

```plaintext
Example trans_eq_exercise : forall (n m o p : nat),
     m = (minustwo o) ->
     (n + p) = m ->
     (n + p) = (minustwo o).
Proof.
  intros n m o p eq1 eq2.
  transitivity m.
  apply eq2.
  apply eq1.
Qed.
```

### Exercise: 3 stars, standard (injection_ex3)

```plaintext
Example injection_ex3 : forall (X : Type) (x y z : X) (l j : list X),
  x :: y :: l = z :: j ->
  j = z :: l ->
  x = y.
Proof.
  intros X x y z l j H1.
  injection H1 as eq1.
  rewrite <- H.
  intros H2.
  injection H2 as eq2.
  rewrite eq1. rewrite eq2. reflexivity.
Qed.
```

### Exercise: 1 star, standard (discriminate_ex3)

```plaintext
Example discriminate_ex3 :
  forall (X : Type) (x y z : X) (l j : list X),
    x :: y :: l = [] ->
    x = z.
Proof.
  intros X x y z l eq1 eq2.
  discriminate eq2.
Qed.
```

(注：不知为何 Coq 自动把 goal 修改为了 `list X -> ... -> x = z`，导致多了一个条件。)

### Exercise: 2 stars, standard (eqb_true)

```plaintext
Theorem eqb_true : forall n m,
  n =? m = true -> n = m.
Proof.
  intros n. induction n as [| n' IHn'].
  - destruct m as [| m'] eqn:E.
    + reflexivity.
    + discriminate.
  - destruct m as [| m'] eqn:E.
    + discriminate.
    + intros H. apply IHn' in H.
      apply f_equal.
      apply H.
Qed.
```

### Exercise: 3 stars, standard, especially useful (plus_n_n_injective)

```plaintext
Theorem plus_n_n_injective : forall n m,
  n + n = m + m ->
  n = m.
Proof.
  intros n. induction n as [| n' IHn'].
  - destruct m as [| m'] eqn:E.
    + reflexivity.
    + discriminate.
  - destruct m as [| m'] eqn:E.
    + discriminate.
    + intros H. 
      simpl in H. injection H as H'.
      rewrite <- plus_n_Sm in H'.
      rewrite <- plus_n_Sm in H'.
      injection H' as H''.
      apply IHn' in H''.
      apply f_equal.
      apply H''.
Qed.
```

### Exercise: 3 stars, standard, especially useful (gen_dep_practice)

```plaintext
Theorem nth_error_after_last: forall (n : nat) (X : Type) (l : list X),
  length l = n ->
  nth_error l n = None.
Proof.
  intros n X l.
  generalize dependent n.
  induction l as [| h l' IHl'].
  - reflexivity.
  - destruct n as [| n'] eqn:E.
    + discriminate.
    + simpl. intros H.
      apply IHl'.
      injection H as goal.
      apply goal.
Qed.
```

### Exercise: 3 stars, standard (combine_split)

```plaintext
Theorem combine_split : forall X Y (l : list (X * Y)) l1 l2,
  split l = (l1, l2) ->
  combine l1 l2 = l.
Proof.
  intros X Y l.
  induction l as [| (x, y) l' IHl'].
  - simpl. intros l1 l2 H.
    injection H as H1 H2.
    rewrite <- H1. rewrite <- H2.
    reflexivity.
  - intros l1 l2 H.
    simpl in H. destruct (split l') as [lx ly].
    destruct l1 as [| h1 l1'].
    + discriminate.
    + destruct l2 as [| h2 l2'].
      * discriminate.
      * simpl.
        injection H as eq1 eq2 eq3 eq4.
        rewrite eq1. rewrite eq3.
        apply f_equal. apply IHl'.
        rewrite eq2. rewrite eq4.
        reflexivity.
Qed.
```

### Exercise: 2 stars, standard (destruct_eqn_practice)

```plaintext
Theorem bool_fn_applied_thrice :
  forall (f : bool -> bool) (b : bool),
  f (f (f b)) = f b.
Proof.
  intros f b.
  destruct b.
  - destruct (f true) eqn:ft.
    + rewrite ft. apply ft.
    + destruct (f false) eqn:ff.
      * apply ft.
      * apply ff.
  - destruct (f false) eqn:ff.
    + destruct (f true) eqn:ft.
      * apply ft.
      * apply ff.
    + rewrite ff. apply ff.
Qed.
```

### Exercise: 3 stars, standard (eqb_sym)

```plaintext
Theorem eqb_sym : forall (n m : nat),
  (n =? m) = (m =? n).
Proof.
  intros n. induction n as [| n' IHn'].
  - destruct m as [| m'].
    + reflexivity.
    + reflexivity.
  - intros m. destruct m as [| m'].
    + reflexivity.
    + simpl. apply IHn'.
Qed.
```

### Exercise: 3 stars, standard, optional (eqb_trans)

```plaintext
Theorem eqb_trans : forall n m p,
  n =? m = true ->
  m =? p = true ->
  n =? p = true.
Proof.
  intros n m p H1 H2.
  generalize dependent n.
  generalize dependent p.
  induction m as [| m' IHm'].
  - destruct n as [| n'].
    + destruct p as [| p'].
      * reflexivity.
      * discriminate.
    + discriminate.
  - intros n Hn p Hp.
    destruct n as [| n'].
    + discriminate.
    + destruct p as [| p'].
      * discriminate.
      * simpl. rewrite eqb_sym.
        simpl in Hn. rewrite eqb_sym in Hn.
        simpl in Hp. rewrite eqb_sym in Hp.
        apply IHm'.
        apply Hp. apply Hn.
Qed.
```