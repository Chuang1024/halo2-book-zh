# Selector combining

Heavy use of custom gates can lead to a circuit defining many binary selectors, which
would increase proof size and verification time.

This section describes an optimization, applied automatically by halo2, that combines
binary selector columns into fewer fixed columns.

The basic idea is that if we have $\ell$ binary selectors labelled $1, \ldots, \ell$ that
are enabled on disjoint sets of rows, then under some additional conditions we can combine
them into a single fixed column, say $q$, such that:
$$
q = \begin{cases}
  k, &\text{if the selector labelled } k \text{ is } 1 \\
  0, &\text{if all these selectors are } 0.
\end{cases}
$$

However, the devil is in the detail.

The halo2 API allows defining some selectors to be "simple selectors", subject to the
following condition:

> Every polynomial constraint involving a simple selector $s$ must be of the form
> $s \cdot t = 0,$ where $t$ is a polynomial involving *no* simple selectors.

Suppose that $s$ has label $k$ in some set of $\ell$ simple selectors that are combined
into $q$ as above. Then this condition ensures that replacing $s$ by
$q \cdot \prod_{1 \leq h \leq \ell,\,h \neq k}\; (h - q)$ will not change the meaning of
any constraints.

> It would be possible to relax this condition by ensuring that every use of a binary
> selector is substituted by a precise interpolation of its value from the corresponding
> combined selector. However,
>
> * the restriction simplifies the implementation, developer tooling, and human
>   understanding and debugging of the resulting constraint system;
> * the scope to apply the optimization is not impeded very much by this restriction for
>   typical circuits.

Note that replacing $s$ by $q \cdot \prod_{1 \leq h \leq \ell,\,h \neq k}\; (h - q)$ will
increase the degree of constraints selected by $s$ by $\ell-1$, and so we must choose the
selectors that are combined in such a way that the maximum degree bound is not exceeded.

## Identifying selectors that can be combined

We need a partition of the overall set of selectors $s_0, \ldots, s_{m-1}$ into subsets
(called "combinations"), such that no two selectors in a combination are enabled on the
same row.

Labels must be unique within a combination, but they are not unique across combinations.
Do not confuse a selector's index with its label.

Suppose that we are given $\mathsf{max\_degree}$, the degree bound of the circuit.

We use the following algorithm:

1. Leave nonsimple selectors unoptimized, i.e. map each of them to a separate fixed
   column.
2. Check (or ensure by construction) that all polynomial constraints involving each simple
   selector $s_i$ are of the form $s_i \cdot t_{i,j} = 0$ where $t_{i,j}$ do not involve
   any simple selectors. For each $i$, record the maximum degree of any $t_{i,j}$ as
   $d^\mathsf{max}_i$.
3. Compute a binary "exclusion matrix" $X$ such that $X_{j,i}$ is $1$ whenever $i \neq j$
   and $s_i$ and $s_j$ are enabled on the same row; and $0$ otherwise.
   > Since $X$ is symmetric and is zero on the diagonal, we can represent it by either its
   > upper or lower triangular entries. The rest of the algorithm is guaranteed only to
   > access only the entries $X_{j,i}$ where $j > i$.
4. Initialize a boolean array $\mathsf{added}_{0..{k-1}}$ to all $\mathsf{false}$.
   > $\mathsf{added}_i$ will record whether $s_i$ has been included in any combination.
6. Iterate over the $s_i$ that have not yet been added to any combination:
   * a. Add $s_i$ to a fresh combination $c$, and set $\mathsf{added}_i = \mathsf{true}$.
   * b. Let mut $d := d^\mathsf{max}_i - 1$.
     > $d$ is used to keep track of the largest degree, *excluding* the selector
     > expression, of any gate involved in the combination $c$ so far.
   * c. Iterate over all the selectors $s_j$ for $j > i$ that can potentially join $c$,
     i.e. for which $\mathsf{added}_j$ is false:
     * i. (Optimization) If $d + \mathsf{len}(c) = \mathsf{max\_degree}$, break to the
       outer loop, since no more selectors can be added to $c$.
     * ii. Let $d^\mathsf{new} = \mathsf{max}(d, d^\mathsf{max}_j-1)$.
     * iii. If $X_{j,i'}$ is $\mathsf{true}$ for any $i'$ in $c$, or if
       $d^\mathsf{new} + (\mathsf{len}(c) + 1) > \mathsf{max\_degree}$, break to the outer
       loop.
       > $d^\mathsf{new} + (\mathsf{len}(c) + 1)$ is the maximum degree, *including* the
       > selector expression, of any constraint that would result from adding $s_j$ to the
       > combination $c$.
     * iv. Set $d := d^\mathsf{new}$.
     * v. Add $s_j$ to $c$ and set $\mathsf{added}_j := \mathsf{true}$.
   * d. Allocate a fixed column $q_c$, initialized to all-zeroes.
   * e. For each selector $s' \in c$:
     * i. Label $s'$ with a distinct index $k$ where $1 \leq k \leq \mathsf{len}(c)$.
     * ii. Record that $s'$ should be substituted with
       $q_c \cdot \prod_{1 \leq h \leq \mathsf{len}(c),\,h \neq k} (h-q_c)$ in all gate
       constraints.
     * iii. For each row $r$ such that $s'$ is enabled at $r$, assign the value $k$ to
       $q_c$ at row $r$.

The above algorithm is implemented in
[halo2_proofs/src/plonk/circuit/compress_selectors.rs](https://github.com/zcash/halo2/blob/main/halo2_proofs/src/plonk/circuit/compress_selectors.rs).
This is used by the `compress_selectors` function of
[halo2_proofs/src/plonk/circuit.rs](https://github.com/zcash/halo2/blob/main/halo2_proofs/src/plonk/circuit.rs)
which does the actual substitutions.

## Writing circuits to take best advantage of selector combining

For this optimization it is beneficial for a circuit to use simple selectors as far as
possible, rather than fixed columns. It is usually not beneficial to do manual combining
of selectors, because the resulting fixed columns cannot take part in the automatic
combining. That means that to get comparable results you would need to do a global
optimization manually, which would interfere with writing composable gadgets.

Whether two selectors are enabled on the same row (and so are inhibited from being
combined) depends on how regions are laid out by the floor planner. The currently
implemented floor planners do not attempt to take this into account. We suggest not
worrying about it too much — the gains that can be obtained by cajoling a floor planner to
shuffle around regions in order to improve combining are likely to be relatively small.


# 选择器组合

大量使用自定义门可能会导致电路定义许多二进制选择器，从而增加证明大小和验证时间。

本节描述了 halo2 自动应用的一种优化，将二进制选择器列合并为更少的固定列。

基本思想是，如果我们有 $\ell$ 个二进制选择器，标记为 $1, \ldots, \ell$，它们在不相交的行集上启用，那么在某些附加条件下，我们可以将它们合并为一个固定列，例如 $q$，使得：
$$
q = \begin{cases}
  k, &\text{如果标记为 } k \text{ 的选择器为 } 1 \\
  0, &\text{如果所有这些选择器都为 } 0。
\end{cases}
$$

然而，细节决定成败。

halo2 API 允许定义一些选择器为“简单选择器”，满足以下条件：

> 每个涉及简单选择器 $s$ 的多项式约束必须为 $s \cdot t = 0$ 的形式，其中 $t$ 是不涉及*任何*简单选择器的多项式。

假设 $s$ 在某个 $\ell$ 个简单选择器的集合中标记为 $k$，并且这些选择器按照上述方式合并为 $q$。那么这一条件确保将 $s$ 替换为 $q \cdot \prod_{1 \leq h \leq \ell,\,h \neq k}\; (h - q)$ 不会改变任何约束的含义。

> 可以通过确保每个二进制选择器的使用都被相应的组合选择器的精确插值替换来放宽这一条件。然而，
>
> * 这一限制简化了实现、开发者工具以及对结果约束系统的人类理解和调试；
> * 对于典型电路，这一限制不会显著影响优化的应用范围。

请注意，将 $s$ 替换为 $q \cdot \prod_{1 \leq h \leq \ell,\,h \neq k}\; (h - q)$ 会使由 $s$ 选择的约束的度数增加 $\ell-1$，因此我们必须以不超出最大度数限制的方式选择要合并的选择器。

## 识别可以合并的选择器

我们需要将整个选择器集合 $s_0, \ldots, s_{m-1}$ 划分为子集（称为“组合”），使得组合中的任何两个选择器不会在同一行上启用。

标签在组合内必须是唯一的，但在组合之间不必唯一。不要将选择器的索引与其标签混淆。

假设我们给定 $\mathsf{max\_degree}$，即电路的最大度数限制。

我们使用以下算法：

1. 保留非简单选择器不进行优化，即将它们各自映射到一个单独的固定列。
2. 检查（或通过构造确保）每个简单选择器 $s_i$ 涉及的所有多项式约束均为 $s_i \cdot t_{i,j} = 0$ 的形式，其中 $t_{i,j}$ 不涉及任何简单选择器。对于每个 $i$，记录任何 $t_{i,j}$ 的最大度数为 $d^\mathsf{max}_i$。
3. 计算一个二元“排除矩阵” $X$，使得当 $i \neq j$ 且 $s_i$ 和 $s_j$ 在同一行上启用时，$X_{j,i}$ 为 $1$；否则为 $0$。
   > 由于 $X$ 是对称的且对角线为零，我们可以通过其上三角或下三角条目来表示它。算法的其余部分保证只访问 $X_{j,i}$，其中 $j > i$。
4. 初始化一个布尔数组 $\mathsf{added}_{0..{k-1}}$，全部为 $\mathsf{false}$。
   > $\mathsf{added}_i$ 将记录 $s_i$ 是否已包含在任何组合中。
6. 遍历尚未添加到任何组合的 $s_i$：
   * a. 将 $s_i$ 添加到一个新的组合 $c$ 中，并设置 $\mathsf{added}_i = \mathsf{true}$。
   * b. 令 $d := d^\mathsf{max}_i - 1$。
     > $d$ 用于跟踪组合 $c$ 中任何门的最大度数，*不包括*选择器表达式。
   * c. 遍历所有可能加入 $c$ 的选择器 $s_j$，即 $j > i$ 且 $\mathsf{added}_j$ 为 false：
     * i. （优化）如果 $d + \mathsf{len}(c) = \mathsf{max\_degree}$，则跳出外部循环，因为不能再向 $c$ 添加更多选择器。
     * ii. 令 $d^\mathsf{new} = \mathsf{max}(d, d^\mathsf{max}_j-1)$。
     * iii. 如果 $X_{j,i'}$ 对于 $c$ 中的任何 $i'$ 为 $\mathsf{true}$，或者如果 $d^\mathsf{new} + (\mathsf{len}(c) + 1) > \mathsf{max\_degree}$，则跳出外部循环。
       > $d^\mathsf{new} + (\mathsf{len}(c) + 1)$ 是将 $s_j$ 添加到组合 $c$ 后，任何约束的最大度数，*包括*选择器表达式。
     * iv. 设置 $d := d^\mathsf{new}$。
     * v. 将 $s_j$ 添加到 $c$ 中，并设置 $\mathsf{added}_j := \mathsf{true}$。
   * d. 分配一个固定列 $q_c$，初始化为全零。
   * e. 对于每个选择器 $s' \in c$：
     * i. 将 $s'$ 标记为一个唯一的索引 $k$，其中 $1 \leq k \leq \mathsf{len}(c)$。
     * ii. 记录在所有门约束中，$s'$ 应替换为 $q_c \cdot \prod_{1 \leq h \leq \mathsf{len}(c),\,h \neq k} (h-q_c)$。
     * iii. 对于每个 $s'$ 启用的行 $r$，将 $q_c$ 在行 $r$ 处的值赋为 $k$。

上述算法在 [halo2_proofs/src/plonk/circuit/compress_selectors.rs](https://github.com/zcash/halo2/blob/main/halo2_proofs/src/plonk/circuit/compress_selectors.rs) 中实现。该算法由 [halo2_proofs/src/plonk/circuit.rs](https://github.com/zcash/halo2/blob/main/halo2_proofs/src/plonk/circuit.rs) 中的 `compress_selectors` 函数使用，该函数执行实际的替换。

## 编写电路以充分利用选择器组合

为了充分利用此优化，电路应尽可能使用简单选择器，而不是固定列。通常不建议手动组合选择器，因为生成的固定列无法参与自动组合。这意味着要获得类似的结果，您需要手动进行全局优化，这会影响编写可组合的小工具。

两个选择器是否在同一行上启用（从而阻止它们被合并）取决于区域布局规划器如何布局区域。当前实现的区域布局规划器不会考虑这一点。我们建议不要过于担心这一点——通过调整区域布局规划器以重新排列区域来改善组合所获得的收益可能相对较小。
