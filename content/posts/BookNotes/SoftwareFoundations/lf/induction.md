---
title: "Proof by Indution"
linktitle: "02: Induction"
type: docs
draft: false
featured: false

math: true

menu:
    sf-lf:
        parent: Chapters
        weight: 2
---

- [Separate Compilation](#separate-compilation)
- [Proof by Induction](#proof-by-induction)
- [Proofs Within Proofs](#proofs-within-proofs)
- [Formal vs. Informal Proof](#formal-vs-informal-proof)
- [Exercises](#exercises)
  - [Exercise: 2 stars, standard, especially useful (basic_induction)](#exercise-2-stars-standard-especially-useful-basic_induction)
  - [Exercise: 2 stars, standard (double_plus)](#exercise-2-stars-standard-double_plus)
  - [Exercise: 2 stars, standard (eqb_refl)](#exercise-2-stars-standard-eqb_refl)
  - [Exercise: 2 stars, standard, optional (even_S)](#exercise-2-stars-standard-optional-even_s)
  - [Exercise: 3 stars, standard, especially useful (mul_comm)](#exercise-3-stars-standard-especially-useful-mul_comm)
  - [Exercise: 2 stars, standard, optional (plus_leb_compat_l)](#exercise-2-stars-standard-optional-plus_leb_compat_l)
  - [Exercise: 3 stars, standard, optional (more_exercises)](#exercise-3-stars-standard-optional-more_exercises)
  - [Exercise: 2 stars, standard, optional (add_shuffle3')](#exercise-2-stars-standard-optional-add_shuffle3)
  - [Exercise: 3 stars, standard, especially useful (binary_commute)](#exercise-3-stars-standard-especially-useful-binary_commute)
  - [Exercise: 3 stars, standard (nat_bin_nat)](#exercise-3-stars-standard-nat_bin_nat)
  - [Exercise: 2 stars, advanced (double_bin)](#exercise-2-stars-advanced-double_bin)
  - [Exercise: 4 stars, advanced (bin_nat_bin)](#exercise-4-stars-advanced-bin_nat_bin)

## Separate Compilation

如果想在新的 `.v` 文件中使用别的文件中的所有定义和定理 (比如 `Basics.v`)，可以使用如下语句：

```plaintext
From LF Require Export Basics.
```

使用这条语句的前置条件是 Coq 可以在有 "LF" 前缀的目录下找到编译过的 `Basics.vo`。`.vo` 文件之于 `.v` 文件就好比 `.o` 文件之于 `.c` 文件。我们应当在当前的工作目录下建立 `_CoqProject` 文件，在其中写上

```plaintext
-Q . LF
```

这会将当前目录绑定到 "LF" 前缀上。之后我们可以用

```bash
coq_makefile -f _CoqProject *.v -o Makefile
```

来自动生成用于编译工作目录下所有源文件的 Makefile，并通过

```bash
make Basics.vo
```

来编译生成 `.vo` 文件。

## Proof by Induction

我们之前证明了 0 是 "+" 的 neutral element，但我们只证明了 0 在左侧的情况，即 `forall n : nat, 0 + n = n.`。如果我们要证明 `n + 0 = n`，事情便变得复杂起来：因为我们的加法是通过对第一个加数的递归定义的，未知大小的整数 `n` 无法递归。在这里我们必须使用归纳法 (induction)。

我们先直接给出归纳法证明 `n + 0 = n` 的步骤：

```plaintext
Theorem add_0_r: forall n : nat, n + 0 = n.
Proof.
  intros n. induction n as [| n' IHn'].
  - reflexivity.
  - simpl.
    rewrite -> IHn'.
    reflexivity.
Qed.
```

`induction` 和 `destruct` 这个 tactic 一样，也可以在后面跟一个 as。第一种情况是令 `n = 0`，这种情况没有参数，所以 `|` 左侧没有东西；第二种情况是令 `n = S n'`，基于的假设是 `n' + 0 = n'` ，我们给这个假设命名为 `IHn'`。

## Proofs Within Proofs

我们进行大的证明的过程中时常需要一些小的结论。我们可以在证明大定理之前把小结论先证明好，然后在大定理中 rewrite。但有时候有一些杂项的、trivial 的结论单独拎出来给证明会使证明过程变得很繁杂，所以 Coq 提供了在证明内部证明另一个命题的语法。下面是一个例子：

```plaintext
Theorem mult_0_plus' forall n m : nat, (n + 0 + 0) * m = n * m.
Proof.
  intros n m.
  assert (H: n + 0 + 0 = n).
    { rewrite add_comm. simpl. rewrite add_comm. reflexivity. }
  rewrite -> H.
  reflexivity.
Qed.
```

`assert` 这个 tactic 会引入两个 subgoal：

* 第一个 subgoal 是括号内写出的命题。上述的写法给这个命题命名为 H。我们也可以使用 `assert (n + 0 + 0 = n) as H.` 的语法来书写这一行。这个命题的证明用一对 `{}` 框起来。 
* 第二个 subgoal 和 assert 之前的一刻的 goal 相同，但多了 assert 证明的结论作为一条假设。

下面展示另一个需要使用 `assert` 的有趣的场景：

```plaintext
Theorem plus_rearrange: forall n m p q : nat, (n + m) + (p + q) = (m + n) + (p + q).
Proof.
  intros n m p q.
  assert (H: n + m = m + n)
    { rewrite add_comm. reflexivity. }
  rewrite H.
  reflexivity.
Qed.
```

可以看到要证明的命题中只有第一个括号内两个加数的顺序不同。但可惜的是我们不能直接利用加法的交换律来 `rewrite`，因为 Coq 根据加法交换律在 goal 中寻找实例时会优先找到外层括号的实例，即会把两个括号的内容整体调换。在这里我们不利用加法交换律 rewrite，而是先证明 `n + m = m + n` 然后直接 rewrite。

## Formal vs. Informal Proof

Formal proof 指的是写给 Coq 等证明工具看的证明过程；Informal proof 指的是用自然语言写给人看的证明过程。一个好的 formal proof 应当通过适当的注释和缩进来使其对人类也比较友好。通常来说，formal proof 在细节处会写得比 informal proof 更详细 (例如 reflexivity)，informal proof 会通过一些语言让读者更好地了解当前地证明状态 (这些信息只有使用 Coq 执行代码时才会显示出来)。

## Exercises

### Exercise: 2 stars, standard, especially useful (basic_induction)

```plaintext
Theorem mul_0_r : forall n:nat,
  n * 0 = 0.
Proof.
  intros n. induction n as [| n' IHn'].
  - reflexivity.
  - simpl.
    rewrite -> IHn'.
    reflexivity.
Qed.
```

```plaintext
Theorem plus_n_Sm : forall n m : nat,
  S (n + m) = n + (S m).
Proof.
  intros n m. induction n as [| n' IHn'].
  - reflexivity.
  - simpl.
    rewrite -> IHn'.
    reflexivity.
Qed.
```

```plaintext
Theorem add_comm : forall n m : nat,
  n + m = m + n.
Proof.
  intros n m. induction n as [| n' IHn'].
  - rewrite -> add_0_r. 
    reflexivity.
  - rewrite <- plus_n_Sm.
    rewrite <- IHn'.
    reflexivity.
Qed.
```

这个证明稍微复杂一些，归纳基础和归纳步骤都需要使用之前证明过的定理。

```plaintext
Theorem add_assoc : forall n m p : nat,
  n + (m + p) = (n + m) + p.
Proof.
  intros n m p. induction n as [| n' IHn'].
  - reflexivity.
  - simpl. 
    rewrite -> IHn'.
    reflexivity.
Qed.
```

### Exercise: 2 stars, standard (double_plus)

```plaintext
Lemma double_plus : forall n, double n = n + n .
Proof.
  intros n. induction n as [| n' IHn'].
  - reflexivity.
  - simpl.
    rewrite -> IHn'.
    rewrite <- plus_n_Sm.
    reflexivity.
Qed.
```

### Exercise: 2 stars, standard (eqb_refl)

```plaintext
Theorem eqb_refl : forall n : nat,
  (n =? n) = true.
Proof.
  intros n. induction n as [| n' IHn'].
  - reflexivity.
  - simpl.
    rewrite -> IHn'.
    reflexivity.
Qed.
```

### Exercise: 2 stars, standard, optional (even_S)

```plaintext
Theorem even_S : forall n : nat,
  even (S n) = negb (even n).
Proof.
  intros n. induction n as [| n' IHn'].
  - reflexivity.
  - rewrite -> IHn'.
    simpl.
    rewrite -> negb_involutive.
    reflexivity.
Qed.
```

本题需要注意的点是：归纳步骤中如果直接上 `simpl` 化简，会将等式右侧的 `even n'` 的定义展开，所以这里笔者先 `rewrite` 再 `simpl`。

### Exercise: 3 stars, standard, especially useful (mul_comm)

```plaintext
Theorem add_shuffle3 : forall n m p : nat,
  n + (m + p) = m + (n + p).
Proof.
  intros n m p.
  rewrite add_assoc.
  rewrite add_assoc.
  assert (H: n + m = m + n).
    { rewrite add_comm. reflexivity. }
  rewrite H.
  reflexivity.
Qed.
```

```plaintext
Lemma mult_n_0: forall n : nat, n * 0 = 0.
Proof.
  induction n as [| n' IHn'].
  - reflexivity.
  - simpl. rewrite IHn'. reflexivity.
Qed.

Lemma mult_n_Sm: forall n m : nat,
  n * (1 + m) = n + n * m.
Proof.
  intros n m.
  induction n as [| n' IHn'].
  - reflexivity.
  - assert (H1: S n' * (1 + m) = S m + n' * (1 + m)). { reflexivity. }
    rewrite H1.
    rewrite IHn'.
    assert (H2: S n' * m = m + n' * m). { reflexivity. }
    rewrite H2.
    simpl.
    rewrite add_shuffle3.
    reflexivity.
Qed.


Theorem mul_comm : forall m n : nat,
  m * n = n * m.
Proof.
  intros n m.
  induction n as [| n' IHn'].
  - rewrite mult_n_0. reflexivity.
  - simpl.
    rewrite mult_n_Sm.
    rewrite IHn'.
    reflexivity.
Qed.
```

这题相当有难度，需要先证明 `n * S m = n + n * m` 这个引理，中间反复的在局部使用乘法定义/加法交换律结合律比较搞心态。

### Exercise: 2 stars, standard, optional (plus_leb_compat_l)

```plaintext
Theorem plus_leb_compat_l : forall n m p : nat,
  n <=? m = true -> (p + n) <=? (p + m) = true.
Proof.
  intros n m p H.
  induction p as [| p' IHp'].
  - simpl. rewrite H. reflexivity.
  - simpl. rewrite IHp'. reflexivity.
Qed.
```

### Exercise: 3 stars, standard, optional (more_exercises)

```plaintext
Theorem leb_refl : forall n:nat,
  (n <=? n) = true.
Proof.
  intros n.
  induction n as [| n' IHn'].
  - reflexivity.
  - simpl. rewrite IHn'. reflexivity.
Qed.

Theorem zero_neqb_S : forall n:nat,
  0 =? (S n) = false.
Proof.
  intros n.
  reflexivity.
Qed.

Theorem andb_false_r : forall b : bool,
  andb b false = false.
Proof.
  intros b.
  rewrite andb_commutative.
  reflexivity.
Qed

Theorem S_neqb_0 : forall n:nat,
  (S n) =? 0 = false.
Proof.
  intros n.
  reflexivity.
Qed.

Theorem mult_1_l : forall n:nat, 1 * n = n.
Proof.
  intros n.
  simpl.
  rewrite add_comm.
  reflexivity.
Qed.

Theorem all3_spec : forall b c : bool,
  orb
    (andb b c)
    (orb (negb b)
         (negb c))
  = true.
Proof.
  intros [] [].
  - reflexivity.
  - reflexivity.
  - reflexivity.
  - reflexivity.
Qed.

Theorem mult_plus_distr_r : forall n m p : nat,
  (n + m) * p = (n * p) + (m * p).
Proof.
  intros.
  rewrite mul_comm.
  induction p as [| p' IHp'].
  - rewrite mult_n_0. rewrite mult_n_0. reflexivity.
  - simpl.
    rewrite mult_n_Sm.
    rewrite mult_n_Sm.
    rewrite IHp'.
    rewrite add_assoc.
    rewrite add_assoc.
    assert (H1: n + m + n * p' = n + (m + n * p')). { rewrite add_assoc. reflexivity. }
    rewrite H1.
    assert (H2: m + n * p' = n * p' + m). { rewrite add_comm. reflexivity. }
    rewrite H2.
    rewrite add_assoc.
    reflexivity.
Qed.

Theorem mult_assoc : forall n m p : nat,
  n * (m * p) = (n * m) * p.
Proof.
  intros.
  induction n as [| n' IHn'].
  - reflexivity.
  - simpl.
    rewrite IHn'.
    rewrite mult_plus_distr_r.
    reflexivity.
Qed.
```

注：乘法分配律的证明由于涉及过多繁琐的加法交换律/结合律，不能保证过程的简洁性。

### Exercise: 2 stars, standard, optional (add_shuffle3')

```plaintext
Theorem add_shuffle3' : forall n m p : nat,
  n + (m + p) = m + (n + p).
Proof.
  intros.
  rewrite add_assoc.
  rewrite add_assoc.
  replace (n + m) with (m + n).
  - reflexivity.
  - rewrite add_comm. reflexivity.
Qed.
```

使用 `replace` 这个 tactic 可以避免每次 assert 之后再 rewrite 新假设的繁琐过程。

### Exercise: 3 stars, standard, especially useful (binary_commute)

```plaintext
Theorem bin_to_nat_pres_incr : forall b : bin,
  bin_to_nat (incr b) = 1 + bin_to_nat b.
Proof.
  intros b. induction b as [| b' | b' IHb']. 
  - reflexivity.
  - simpl. rewrite add_comm. reflexivity.
  - simpl. rewrite IHb'. simpl. rewrite add_comm. reflexivity.
Qed.
```

这里值得一提的是：Coq 中只有基于结构的归纳法，只要变量类型是以 inductive 的形式定义的，我们都可以使用归纳法进行证明。之前的归纳法都针对自然数，这是因为自然数也是归纳定义的。这里的 `bin` 有三个归纳分支，写起来和自然数大同小异。

### Exercise: 3 stars, standard (nat_bin_nat)

```plaintext
Fixpoint nat_to_bin (n:nat) : bin :=
  match n with
  | O => Z
  | S n' => incr(nat_to_bin(n'))
  end.
```

```plaintext
Theorem nat_bin_nat : forall n, bin_to_nat (nat_to_bin n) = n.
Proof.
  intros. induction n as [| n' IHn'].
  - reflexivity.
  - simpl. 
    rewrite bin_to_nat_pres_incr.
    rewrite IHn'.
    reflexivity.
Qed.
```

### Exercise: 2 stars, advanced (double_bin)

```plaintext
Lemma double_incr : forall n : nat, double (S n) = S (S (double n)).
Proof. reflexivity. Qed.
```

```plaintext
Definition double_bin (b:bin) : bin :=
  match b with
  | Z => Z
  | _ => B0 b
  end.
```

```plaintext
Example double_bin_zero : double_bin Z = Z.
Proof. reflexivity. Qed.
```

```plaintext
Lemma double_incr_bin : forall b,
    double_bin (incr b) = incr (incr (double_bin b)).
Proof.
  intros b.
  induction b as [| b' | b' IHb'].
  - reflexivity.
  - reflexivity.
  - reflexivity.
Qed.
```

### Exercise: 4 stars, advanced (bin_nat_bin)

```plaintext
Fixpoint normalize (b:bin) : bin :=
  match b with
  | Z => Z
  | B0 b' =>
    match bin_to_nat b' with
    | 0 => Z
    | S n' => B0 (normalize b')
    end
  | B1 b' => B1 (normalize b')
  end.

Example normalize_example : normalize (B1 (B0 Z)) = B1 Z.
Proof. reflexivity. Qed.
```

书中提到的 "equivalent" bin 实质上指的就是前导 0 的问题，normalize 的主要任务是去除掉 `B0 (B0 ... Z)` 的情况，因此对 `B0 b'` 的情况进行特殊判断。

这里利用了 `bin_to_nat b'` 的结果判断是否是 0 而没有直接用 `normalize b'` 的结果，是为了下面证明的方便。下面的证明中对 `bin_to_nat b'` 的结果进行了分类讨论，这样定义可以让 `simpl` 直接拆解定义。

```plaintext
Lemma mult_2_plus_1_bin : forall n : nat, nat_to_bin(n * 2 + 1) = B1 (nat_to_bin n).
Proof.
  intros n.
  induction n as [| n' IHn'].
  - reflexivity.
  - simpl.
    rewrite IHn'.
    reflexivity.
Qed.

Lemma mult_2_eq_double : forall n : nat, nat_to_bin (n * 2) = double_bin (nat_to_bin n).
Proof.
  intros n.
  induction n as [| n' IHn'].
  - reflexivity.
  - simpl.
    rewrite IHn'.
    rewrite double_incr_bin.
    reflexivity.
Qed.

Lemma double_bin_eq_B0 : forall b : bin, double_bin (incr b) = B0 (incr b).
Proof.
  intros b.
  destruct b as [| b' | b'] eqn:B.
  - reflexivity.
  - reflexivity.
  - reflexivity.
Qed.

Lemma mult_2_bin : forall n : nat, nat_to_bin (S n * 2) = B0 (nat_to_bin (S n)).
Proof.
  intros n.
  simpl.
  rewrite mult_2_eq_double.
  rewrite <- double_incr_bin.
  rewrite double_bin_eq_B0.
  reflexivity.
Qed.

Theorem bin_nat_bin : forall b, nat_to_bin (bin_to_nat b) = normalize b.
Proof.
  intros b. induction b as [| b' | b' IHb'].
  - reflexivity.
  - destruct (bin_to_nat b') as [| n'] eqn:N.
    + simpl. rewrite N. reflexivity.
    + simpl. rewrite N. 
      rewrite <- IHb'.
      rewrite mult_2_bin.
      reflexivity.
  - simpl.
    rewrite <- IHb'.
    rewrite mult_2_plus_1_bin.
    reflexivity.
Qed.
```

为了证明最终的定理，笔者先证明了 3 个引理，主要围绕着 `*2`/`*2+1` 和在 bin前面添加 B0/B1 的一致性讨论。最终定理的证明中比较精妙的部分是关于 `bin_to_nat b'` 的讨论。在归纳步骤中等式右边是 `normalize (B0 b')`，如果我们不能确定 `b'` 不是 Z，那么括号里的 B0 就提不出来，可如果对着 b' 的结构直接进行讨论，又容易陷入循环论证 (如果 `b'= B0 b''`，仍然说明不了问题)，因此在这里选择对 b' 转化成自然数后的结果进行讨论。
