# Lookup argument

Halo 2 uses the following lookup technique, which allows for lookups in arbitrary sets, and
is arguably simpler than Plookup.

## Note on Language

In addition to the [general notes on language](../../design.md#note-on-language):

- We call the $Z(X)$ polynomial (the grand product argument polynomial for the permutation
  argument) the "permutation product" column.

## Technique Description

For ease of explanation, we'll first describe a simplified version of the argument that
ignores zero knowledge.

We express lookups in terms of a "subset argument" over a table with $2^k$ rows (numbered
from 0), and columns $A$ and $S.$

The goal of the subset argument is to enforce that every cell in $A$ is equal to _some_
cell in $S.$ This means that more than one cell in $A$ can be equal to the _same_ cell in
$S,$ and some cells in $S$ don't need to be equal to any of the cells in $A.$

- $S$ might be fixed, but it doesn't need to be. That is, we can support looking up values
  in either fixed or variable tables (where the latter includes advice columns).
- $A$ and $S$ can contain duplicates. If the sets represented by $A$ and/or $S$ are not
  naturally of size $2^k,$ we extend $S$ with duplicates and $A$ with dummy values known
  to be in $S.$
  - Alternatively we could add a "lookup selector" that controls which elements of the $A$
    column participate in lookups. This would modify the occurrence of $A(X)$ in the
    permutation rule below to replace $A$ with, say, $S_0$ if a lookup is not selected.

Let $\ell_i$ be the Lagrange basis polynomial that evaluates to $1$ at row $i,$ and $0$
otherwise.

We start by allowing the prover to supply permutation columns of $A$ and $S.$ Let's call
these $A'$ and $S',$ respectively. We can enforce that they are permutations using a
permutation argument with product column $Z$ with the rules:

$$
Z(\omega X) \cdot (A'(X) + \beta) \cdot (S'(X) + \gamma) - Z(X) \cdot (A(X) + \beta) \cdot (S(X) + \gamma) = 0
$$$$
\ell_0(X) \cdot (1 - Z(X)) = 0
$$

i.e. provided that division by zero does not occur, we have for all $i \in [0, 2^k)$:

$$
Z_{i+1} = Z_i \cdot \frac{(A_i + \beta) \cdot (S_i + \gamma)}{(A'_i + \beta) \cdot (S'_i + \gamma)}
$$$$
Z_{2^k} = Z_0 = 1.
$$

This is a version of the permutation argument which allows $A'$ and $S'$ to be
permutations of $A$ and $S,$ respectively, but doesn't specify the exact permutations.
$\beta$ and $\gamma$ are separate challenges so that we can combine these two permutation
arguments into one without worrying that they might interfere with each other.

The goal of these permutations is to allow $A'$ and $S'$ to be arranged by the prover in a
particular way:

1. All the cells of column $A'$ are arranged so that like-valued cells are vertically
   adjacent to each other. This could be done by some kind of sorting algorithm, but all
   that matters is that like-valued cells are on consecutive rows in column $A',$ and that
   $A'$ is a permutation of $A.$
2. The first row in a sequence of like values in $A'$ is the row that has the
   corresponding value in $S'.$ Apart from this constraint, $S'$ is any arbitrary
   permutation of $S.$

Now, we'll enforce that either $A'_i = S'_i$ or that $A'_i = A'_{i-1},$ using the rule

$$
(A'(X) - S'(X)) \cdot (A'(X) - A'(\omega^{-1} X)) = 0
$$

In addition, we enforce $A'_0 = S'_0$ using the rule

$$
\ell_0(X) \cdot (A'(X) - S'(X)) = 0
$$

(The $A'(X) - A'(\omega^{-1} X)$ term of the first rule here has no effect at row $0,$ even
though $\omega^{-1} X$ "wraps", because of the second rule.)

Together these constraints effectively force every element in $A'$ (and thus $A$) to equal
at least one element in $S'$ (and thus $S$). Proof: by induction on prefixes of the rows.

## Zero-knowledge adjustment

In order to achieve zero knowledge for the PLONK-based proof system, we will need the last
$t$ rows of each column to be filled with random values. This requires an adjustment to the
lookup argument, because these random values would not satisfy the constraints described
above.

We limit the number of usable rows to $u = 2^k - t - 1.$ We add two selectors:

* $q_\mathit{blind}$ is set to $1$ on the last $t$ rows, and $0$ elsewhere;
* $q_\mathit{last}$ is set to $1$ only on row $u,$ and $0$ elsewhere (i.e. it is set on the
  row in between the usable rows and the blinding rows).

We enable the constraints from above only for the usable rows:

$$
\big(1 - (q_\mathit{last}(X) + q_\mathit{blind}(X))\big) \cdot \big(Z(\omega X) \cdot (A'(X) + \beta) \cdot (S'(X) + \gamma) - Z(X) \cdot (A(X) + \beta) \cdot (S(X) + \gamma)\big) = 0
$$$$
\big(1 - (q_\mathit{last}(X) + q_\mathit{blind}(X))\big) \cdot (A'(X) - S'(X)) \cdot (A'(X) - A'(\omega^{-1} X)) = 0
$$

The rules that are enabled on row $0$ remain the same:

$$
\ell_0(X) \cdot (A'(X) - S'(X)) = 0
$$$$
\ell_0(X) \cdot (1 - Z(X)) = 0
$$

Since we can no longer rely on the wraparound to ensure that the product $Z$ becomes $1$
again at $\omega^{2^k},$ we would instead need to constrain $Z(\omega^u)$ to $1.$ However,
there is a potential difficulty: if any of the values $A_i + \beta$ or $S_i + \gamma$ are
zero for $i \in [0, u),$ then it might not be possible to satisfy the permutation argument.
This occurs with negligible probability over choices of $\beta$ and $\gamma,$ but is an
obstacle to achieving *perfect* zero knowledge (because an adversary can rule out witnesses
that would cause this situation), as well as perfect completeness.

To ensure both perfect completeness and perfect zero knowledge, we allow $Z(\omega^u)$
to be either zero or one:

$$
q_\mathit{last}(X) \cdot (Z(X)^2 - Z(X)) = 0
$$

Now if $A_i + \beta$ or $S_i + \gamma$ are zero for some $i,$ we can set $Z_j = 0$ for
$i < j \leq u,$ satisfying the constraint system.

Note that the challenges $\beta$ and $\gamma$ are chosen after committing to $A$ and $S$
(and to $A'$ and $S'$), so the prover cannot force the case where some $A_i + \beta$ or
$S_i + \gamma$ is zero to occur. Since this case occurs with negligible probability,
soundness is not affected.

## Cost

* There is the original column $A$ and the fixed column $S.$
* There is a permutation product column $Z.$
* There are the two permutations $A'$ and $S'.$
* The gates are all of low degree.

## Generalizations

Halo 2's lookup argument implementation generalizes the above technique in the following
ways:

- $A$ and $S$ can be extended to multiple columns, combined using a random challenge. $A'$
  and $S'$ stay as single columns.
  - The commitments to the columns of $S$ can be precomputed, then combined cheaply once
    the challenge is known by taking advantage of the homomorphic property of Pedersen
    commitments.
  - The columns of $A$ can be given as arbitrary polynomial expressions using relative
    references. These will be substituted into the product column constraint, subject to
    the maximum degree bound. This potentially saves one or more advice columns.
- Then, a lookup argument for an arbitrary-width relation can be implemented in terms of a
  subset argument, i.e. to constrain $\mathcal{R}(x, y, ...)$ in each row, consider
  $\mathcal{R}$ as a set of tuples $S$ (using the method of the previous point), and check
  that $(x, y, ...) \in \mathcal{R}.$
  - In the case where $\mathcal{R}$ represents a function, this implicitly also checks
    that the inputs are in the domain. This is typically what we want, and often saves an
    additional range check.
- We can support multiple tables in the same circuit, by combining them into a single
  table that includes a tag column to identify the original table.
  - The tag column could be merged with the "lookup selector" mentioned earlier, if this
    were implemented.

These generalizations are similar to those in sections 4 and 5 of the
[Plookup paper](https://eprint.iacr.org/2020/315.pdf). That is, the differences from
Plookup are in the subset argument. This argument can then be used in all the same ways;
for instance, the optimized range check technique in section 5 of the Plookup paper can
also be used with this subset argument.

# 查找论证

Halo 2 使用以下查找技术，该技术允许在任意集合中进行查找，并且可以说比 Plookup 更简单。

## 语言说明

除了[语言的一般说明](../../design.md#note-on-language)外：

- 我们将 $Z(X)$ 多项式（置换参数的大乘积参数多项式）称为“置换乘积”列。

## 技术描述

为了便于解释，我们首先描述一个简化版本的参数，该版本忽略零知识。

我们通过在具有 $2^k$ 行（编号从 0 开始）的表中表达查找，表中有列 $A$ 和 $S$。

子集参数的目标是强制 $A$ 中的每个单元格等于 $S$ 中的某个单元格。这意味着 $A$ 中的多个单元格可以等于 $S$ 中的同一个单元格，并且 $S$ 中的某些单元格不需要等于 $A$ 中的任何单元格。

- $S$ 可能是固定的，但不必如此。也就是说，我们可以支持在固定表或变量表中查找值（后者包括建议列）。
- $A$ 和 $S$ 可以包含重复项。如果 $A$ 和/或 $S$ 表示的集合的大小不是 $2^k$，我们用重复项扩展 $S$，并用已知在 $S$ 中的虚拟值扩展 $A$。
  - 或者，我们可以添加一个“查找选择器”来控制 $A$ 列的哪些元素参与查找。这将修改下面置换规则中 $A(X)$ 的出现，例如，如果未选择查找，则将 $A$ 替换为 $S_0$。

设 $\ell_i$ 为在行 $i$ 处求值为 $1$，否则为 $0$ 的拉格朗日基多项式。

我们首先允许证明者提供 $A$ 和 $S$ 的置换列。我们分别称这些为 $A'$ 和 $S'$。我们可以使用具有乘积列 $Z$ 的置换参数来强制它们是置换，规则如下：

$$
Z(\omega X) \cdot (A'(X) + \beta) \cdot (S'(X) + \gamma) - Z(X) \cdot (A(X) + \beta) \cdot (S(X) + \gamma) = 0
$$$$
\ell_0(X) \cdot (1 - Z(X)) = 0
$$

即，假设不发生除以零的情况，对于所有 $i \in [0, 2^k)$，我们有：

$$
Z_{i+1} = Z_i \cdot \frac{(A_i + \beta) \cdot (S_i + \gamma)}{(A'_i + \beta) \cdot (S'_i + \gamma)}
$$$$
Z_{2^k} = Z_0 = 1.
$$

这是置换参数的一个版本，它允许 $A'$ 和 $S'$ 分别是 $A$ 和 $S$ 的置换，但不指定确切的置换。$\beta$ 和 $\gamma$ 是单独的挑战，因此我们可以将这两个置换参数组合在一起，而不必担心它们可能会相互干扰。

这些置换的目标是允许证明者以特定方式排列 $A'$ 和 $S'$：

1. $A'$ 列的所有单元格排列为具有相同值的单元格在垂直方向上相邻。这可以通过某种排序算法完成，但重要的是具有相同值的单元格在 $A'$ 列中位于连续行上，并且 $A'$ 是 $A$ 的置换。
2. $A'$ 中具有相同值的序列的第一行是在 $S'$ 中具有相应值的行。除了此约束外，$S'$ 是 $S$ 的任意置换。

现在，我们将强制 $A'_i = S'_i$ 或 $A'_i = A'_{i-1}$，使用以下规则：

$$
(A'(X) - S'(X)) \cdot (A'(X) - A'(\omega^{-1} X)) = 0
$$

此外，我们使用以下规则强制 $A'_0 = S'_0$：

$$
\ell_0(X) \cdot (A'(X) - S'(X)) = 0
$$

（此处第一个规则中的 $A'(X) - A'(\omega^{-1} X)$ 项在行 $0$ 处没有效果，即使 $\omega^{-1} X$ “回绕”，因为第二个规则。）

这些约束共同有效地强制 $A'$（以及 $A$）中的每个元素等于 $S'$（以及 $S$）中的至少一个元素。证明：通过对行的前缀进行归纳。

## 零知识调整

为了实现基于 PLONK 的证明系统的零知识，我们需要将每列的最后 $t$ 行填充为随机值。这需要对查找参数进行调整，因为这些随机值不会满足上述约束。

我们将可用行的数量限制为 $u = 2^k - t - 1$。我们添加两个选择器：

* $q_\mathit{blind}$ 在最后 $t$ 行设置为 $1$，在其他地方设置为 $0$；
* $q_\mathit{last}$ 仅在行 $u$ 设置为 $1$，在其他地方设置为 $0$（即，它设置在可用行和盲行之间的行上）。

我们仅为可用行启用上述约束：

$$
\big(1 - (q_\mathit{last}(X) + q_\mathit{blind}(X))\big) \cdot \big(Z(\omega X) \cdot (A'(X) + \beta) \cdot (S'(X) + \gamma) - Z(X) \cdot (A(X) + \beta) \cdot (S(X) + \gamma)\big) = 0
$$$$
\big(1 - (q_\mathit{last}(X) + q_\mathit{blind}(X))\big) \cdot (A'(X) - S'(X)) \cdot (A'(X) - A'(\omega^{-1} X)) = 0
$$

在行 $0$ 上启用的规则保持不变：

$$
\ell_0(X) \cdot (A'(X) - S'(X)) = 0
$$$$
\ell_0(X) \cdot (1 - Z(X)) = 0
$$

由于我们不能再依赖回绕来确保乘积 $Z$ 在 $\omega^{2^k}$ 处再次变为 $1$，我们需要将 $Z(\omega^u)$ 约束为 $1$。然而，存在一个潜在的困难：如果对于 $i \in [0, u)$，任何值 $A_i + \beta$ 或 $S_i + \gamma$ 为零，则可能无法满足置换参数。这种情况在选择 $\beta$ 和 $\gamma$ 时发生的概率可以忽略不计，但这是实现*完美*零知识（因为对手可以排除会导致这种情况的见证）以及完美完备性的障碍。

为了确保完美完备性和完美零知识，我们允许 $Z(\omega^u)$ 为零或一：

$$
q_\mathit{last}(X) \cdot (Z(X)^2 - Z(X)) = 0
$$

现在，如果对于某些 $i$，$A_i + \beta$ 或 $S_i + \gamma$ 为零，我们可以为 $i < j \leq u$ 设置 $Z_j = 0$，从而满足约束系统。

注意，挑战 $\beta$ 和 $\gamma$ 是在提交 $A$ 和 $S$（以及 $A'$ 和 $S'$）之后选择的，因此证明者无法强制 $A_i + \beta$ 或 $S_i + \gamma$ 为零的情况发生。由于这种情况发生的概率可以忽略不计，因此不会影响健全性。

## 成本

* 有原始列 $A$ 和固定列 $S$。
* 有一个置换乘积列 $Z$。
* 有两个置换 $A'$ 和 $S'$。
* 所有门的度数都很低。

## 泛化

Halo 2 的查找参数实现通过以下方式泛化了上述技术：

- $A$ 和 $S$ 可以扩展到多列，使用随机挑战组合。$A'$ 和 $S'$ 保持为单列。
  - $S$ 的列的承诺可以预先计算，然后在知道挑战后利用 Pedersen 承诺的同态属性廉价组合。
  - $A$ 的列可以作为使用相对引用的任意多项式表达式给出。这些将被代入乘积列约束，受最大度数限制。这可能会节省一个或多个建议列。
- 然后，任意宽度关系的查找参数可以实现为子集参数，即为了在每行中约束 $\mathcal{R}(x, y, ...)$，将 $\mathcal{R}$ 视为元组集合 $S$（使用前一点的方法），并检查 $(x, y, ...) \in \mathcal{R}$。
  - 在 $\mathcal{R}$ 表示函数的情况下，这隐含地还检查输入是否在域中。这通常是我们想要的，并且通常可以节省额外的范围检查。
- 我们可以在同一电路中支持多个表，通过将它们组合成一个包含标签列以标识原始表的表。
  - 标签列可以与前面提到的“查找选择器”合并，如果实现了这一点。

这些泛化与 [Plookup 论文](https://eprint.iacr.org/2020/315.pdf) 的第 4 节和第 5 节中的泛化类似。也就是说，与 Plookup 的区别在于子集参数。然后，该参数可以以所有相同的方式使用；例如，Plookup 论文第 5 节中的优化范围检查技术也可以与此子集参数一起使用。