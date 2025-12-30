# Permutation argument

Given that gates in halo2 circuits operate "locally" (on cells in the current row or
defined relative rows), it is common to need to copy a value from some arbitrary cell into
the current row for use in a gate. This is performed with an equality constraint, which
enforces that the source and destination cells contain the same value.

We implement these equality constraints by constructing a permutation that represents the
constraints, and then using a permutation argument within the proof to enforce them.

## Notation

A permutation is a one-to-one and onto mapping of a set onto itself. A permutation can be
factored uniquely into a composition of cycles (up to ordering of cycles, and rotation of
each cycle).

We sometimes use [cycle notation](https://en.wikipedia.org/wiki/Permutation#Cycle_notation)
to write permutations. Let $(a\ b\ c)$ denote a cycle where $a$ maps to $b,$ $b$ maps to
$c,$ and $c$ maps to $a$ (with the obvious generalization to arbitrary-sized cycles).
Writing two or more cycles next to each other denotes a composition of the corresponding
permutations. For example, $(a\ b)\ (c\ d)$ denotes the permutation that maps $a$ to $b,$
$b$ to $a,$ $c$ to $d,$ and $d$ to $c.$

## Constructing the permutation

### Goal

We want to construct a permutation in which each subset of variables that are in a
equality-constraint set form a cycle. For example, suppose that we have a circuit that
defines the following equality constraints:

- $a \equiv b$
- $a \equiv c$
- $d \equiv e$

From this we have the equality-constraint sets $\{a, b, c\}$ and $\{d, e\}.$ We want to
construct the permutation:

$$(a\ b\ c)\ (d\ e)$$

which defines the mapping of $[a, b, c, d, e]$ to $[b, c, a, e, d].$

### Algorithm

We need to keep track of the set of cycles, which is a
[set of disjoint sets](https://en.wikipedia.org/wiki/Disjoint-set_data_structure).
Efficient data structures for this problem are known; for the sake of simplicity we choose
one that is not asymptotically optimal but is easy to implement.

We represent the current state as:

- an array $\mathsf{mapping}$ for the permutation itself;
- an auxiliary array $\mathsf{aux}$ that keeps track of a distinguished element of each
  cycle;
- another array $\mathsf{sizes}$ that keeps track of the size of each cycle.

We have the invariant that for each element $x$ in a given cycle $C,$ $\mathsf{aux}(x)$
points to the same element $c \in C.$ This allows us to quickly decide whether two given
elements $x$ and $y$ are in the same cycle, by checking whether
$\mathsf{aux}(x) = \mathsf{aux}(y).$ Also, $\mathsf{sizes}(\mathsf{aux}(x))$ gives the
size of the cycle containing $x.$ (This is guaranteed only for
$\mathsf{sizes}(\mathsf{aux}(x)),$ not for $\mathsf{sizes}(x).$)

The algorithm starts with a representation of the identity permutation:
for all $x,$ we set $\mathsf{mapping}(x) = x,$ $\mathsf{aux}(x) = x,$ and
$\mathsf{sizes}(x) = 1.$

To add an equality constraint $\mathit{left} \equiv \mathit{right}$:

1. Check whether $\mathit{left}$ and $\mathit{right}$ are already in the same cycle, i.e.
   whether $\mathsf{aux}(\mathit{left}) = \mathsf{aux}(\mathit{right}).$ If so, there is
   nothing to do.
2. Otherwise, $\mathit{left}$ and $\mathit{right}$ belong to different cycles. Make
   $\mathit{left}$ the larger cycle and $\mathit{right}$ the smaller one, by swapping them
   iff $\mathsf{sizes}(\mathsf{aux}(\mathit{left})) < \mathsf{sizes}(\mathsf{aux}(\mathit{right})).$
3. Set $\mathsf{sizes}(\mathsf{aux}(\mathit{left})) :=
        \mathsf{sizes}(\mathsf{aux}(\mathit{left})) + \mathsf{sizes}(\mathsf{aux}(\mathit{right})).$
4. Following the mapping around the right (smaller) cycle, for each element $x$ set
   $\mathsf{aux}(x) := \mathsf{aux}(\mathit{left}).$
5. Splice the smaller cycle into the larger one by swapping $\mathsf{mapping}(\mathit{left})$
   with $\mathsf{mapping}(\mathit{right}).$

For example, given two disjoint cycles $(A\ B\ C\ D)$ and $(E\ F\ G\ H)$:

```plaintext
A +---> B
^       +
|       |
+       v
D <---+ C       E +---> F
                ^       +
                |       |
                +       v
                H <---+ G
```

After adding constraint $B \equiv E$ the above algorithm produces the cycle:

```plaintext
A +---> B +-------------+
^                       |
|                       |
+                       v
D <---+ C <---+ E       F
                ^       +
                |       |
                +       v
                H <---+ G
```

### Broken alternatives

If we did not check whether $\mathit{left}$ and $\mathit{right}$ were already in the same
cycle, then we could end up undoing an equality constraint. For example, if we have the
following constraints:

- $a \equiv b$
- $b \equiv c$
- $c \equiv d$
- $b \equiv d$

and we tried to implement adding an equality constraint just using step 5 of the above
algorithm, then we would end up constructing the cycle $(a\ b)\ (c\ d),$ rather than the
correct $(a\ b\ c\ d).$

## Argument specification

We need to check a permutation of cells in $m$ columns, represented in Lagrange basis by
polynomials $v_0, \ldots, v_{m-1}.$

We will label *each cell* in those $m$ columns with a unique element of $\mathbb{F}^\times.$

Suppose that we have a permutation on these labels,
$$
\sigma(\mathsf{column}: i, \mathsf{row}: j) = (\mathsf{column}: i', \mathsf{row}: j').
$$
in which the cycles correspond to equality-constraint sets.

> If we consider the set of pairs $\{(\mathit{label}, \mathit{value})\}$, then the values within
> each cycle are equal if and only if permuting the label in each pair by $\sigma$ yields the
> same set:
>
> ![An example for a cycle (A B C D). The set before permuting the labels is {(A, 7), (B, 7), (C, 7), (D, 7)}, and the set after is {(D, 7), (A, 7), (B, 7), (C, 7)} which is the same. If one of the 7s is replaced by 3, then the set after permuting labels is not the same.](./permutation-diagram.png)
>
> Since the labels are distinct, set equality is the same as multiset equality, which we can
> check using a product argument.

Let $\omega$ be a $2^k$ root of unity and let $\delta$ be a $T$ root of unity, where
${T \cdot 2^S + 1 = p}$ with $T$ odd and ${k \leq S.}$
We will use $\delta^i \cdot \omega^j \in \mathbb{F}^\times$ as the label for the
cell in the $j$th row of the $i$th column of the permutation argument.

We represent $\sigma$ by a vector of $m$ polynomials $s_i(X)$ such that
$s_i(\omega^j) = \delta^{i'} \cdot \omega^{j'}.$

Notice that the identity permutation can be represented by the vector of $m$ polynomials
$\mathsf{ID}_i(\omega^j)$ such that $\mathsf{ID}_i(\omega^j) = \delta^i \cdot \omega^j.$

We will use a challenge $\beta$ to compress each ${(\mathit{label}, \mathit{value})}$ pair
to $\mathit{value} + \beta \cdot \mathit{label}.$ Just as in the product argument we used
for [lookups](lookup.md), we also use a challenge $\gamma$ to randomize each term of the
product.

Now given our permutation represented by $s_0, \ldots, s_{m-1}$ over columns represented by
$v_0, \ldots, v_{m-1},$ we want to ensure that:
$$
\prod\limits_{i=0}^{m-1} \prod\limits_{j=0}^{n-1} \left(\frac{v_i(\omega^j) + \beta \cdot \delta^i \cdot \omega^j + \gamma}{v_i(\omega^j) + \beta \cdot s_i(\omega^j) + \gamma}\right) = 1
$$

> Here ${v_i(\omega^j) + \beta \cdot \delta^i \cdot \omega^j}$ represents the unpermuted
> $(\mathit{label}, value)$ pair, and ${v_i(\omega^j) + \beta \cdot s_i(\omega^j)}$
> represents the permuted $(\sigma(\mathit{label}), value)$ pair.

Let $Z_P$ be such that $Z_P(\omega^0) = Z_P(\omega^n) = 1$ and for $0 \leq j < n$:
$$\begin{array}{rl}
Z_P(\omega^{j+1}) &= \prod\limits_{h=0}^{j} \prod\limits_{i=0}^{m-1} \frac{v_i(\omega^h) + \beta \cdot \delta^i \cdot \omega^h + \gamma}{v_i(\omega^h) + \beta \cdot s_i(\omega^h) + \gamma} \\
                  &= Z_P(\omega^j) \prod\limits_{i=0}^{m-1} \frac{v_i(\omega^j) + \beta \cdot \delta^i \cdot \omega^j + \gamma}{v_i(\omega^j) + \beta \cdot s_i(\omega^j) + \gamma}
\end{array}$$

Then it is sufficient to enforce the rules:
$$
Z_P(\omega X) \cdot \prod\limits_{i=0}^{m-1} \left(v_i(X) + \beta \cdot s_i(X) + \gamma\right) - Z_P(X) \cdot \prod\limits_{i=0}^{m-1} \left(v_i(X) + \beta \cdot \delta^i \cdot X + \gamma\right) = 0 \\
\ell_0 \cdot (1 - Z_P(X)) = 0
$$

This assumes that the number of columns $m$ is such that the polynomial in the first
rule above fits within the degree bound of the PLONK configuration. We will see
[below](#spanning-a-large-number-of-columns) how to handle a larger number of columns.

> The optimization used to obtain the simple representation of the identity permutation was suggested
> by Vitalik Buterin for PLONK, and is described at the end of section 8 of the PLONK paper. Note that
> the $\delta^i$ are all distinct quadratic non-residues, provided that the number of columns that
> are enabled for equality is no more than $T$, which always holds in practice for the curves used in
> Halo 2.

## Zero-knowledge adjustment

Similarly to the [lookup argument](lookup.md#zero-knowledge-adjustment), we need an
adjustment to the above argument to account for the last $t$ rows of each column being
filled with random values.

We limit the number of usable rows to $u = 2^k - t - 1.$ We add two selectors,
defined in the same way as for the lookup argument:

* $q_\mathit{blind}$ is set to $1$ on the last $t$ rows, and $0$ elsewhere;
* $q_\mathit{last}$ is set to $1$ only on row $u,$ and $0$ elsewhere (i.e. it is set on
  the row in between the usable rows and the blinding rows).

We enable the product rule from above only for the usable rows:

$\big(1 - (q_\mathit{last}(X) + q_\mathit{blind}(X))\big) \cdot$
$\hspace{1em}\left(Z_P(\omega X) \cdot \prod\limits_{i=0}^{m-1} \left(v_i(X) + \beta \cdot s_i(X) + \gamma\right) - Z_P(X) \cdot \prod\limits_{i=0}^{m-1} \left(v_i(X) + \beta \cdot \delta^i \cdot X + \gamma\right)\right) = 0$

The rule that is enabled on row $0$ remains the same:

$$
\ell_0(X) \cdot (1 - Z_P(X)) = 0
$$

Since we can no longer rely on the wraparound to ensure that each product $Z_P$ becomes
$1$ again at $\omega^{2^k},$ we would instead need to constrain $Z(\omega^u) = 1.$ This
raises the same problem that was described for the lookup argument. So we allow
$Z(\omega^u)$ to be either zero or one:

$$
q_\mathit{last}(X) \cdot (Z_P(X)^2 - Z_P(X)) = 0
$$

which gives perfect completeness and zero knowledge.

## Spanning a large number of columns

The halo2 implementation does not in practice limit the number of columns for which
equality constraints can be enabled. Therefore, it must solve the problem that the
above approach might yield a product rule with a polynomial that exceeds the PLONK
configuration's degree bound. The degree bound could be raised, but this would be
inefficient if no other rules require a larger degree.

Instead, we split the product across $b$ sets of $m$ columns, using product columns
$Z_{P,0}, \ldots Z_{P,b-1},$ and we use another rule to copy the product from the end
of one column set to the beginning of the next.

That is, for $0 \leq a < b$ we have:

$\big(1 - (q_\mathit{last}(X) + q_\mathit{blind}(X))\big) \cdot$
$\hspace{1em}\left(Z_{P,a}(\omega X) \cdot \!\prod\limits_{i=am}^{(a+1)m-1}\! \left(v_i(X) + \beta \cdot s_i(X) + \gamma\right) - Z_P(X) \cdot \!\prod\limits_{i=am}^{(a+1)m-1}\! \left(v_i(X) + \beta \cdot \delta^i \cdot X + \gamma\right)\right)$
$\hspace{2em}= 0$

> For simplicity this is written assuming that the number of columns enabled for
> equality constraints is a multiple of $m$; if not then the products for the last
> column set will have fewer than $m$ terms.

For the first column set we have:

$$
\ell_0 \cdot (1 - Z_{P,0}(X)) = 0
$$

For each subsequent column set, $0 < a < b,$ we use the following rule to copy
$Z_{P,a-1}(\omega^u)$ to the start of the next column set, $Z_{P,a}(\omega^0)$:

$$
\ell_0 \cdot \left(Z_{P,a}(X) - Z_{P,a-1}(\omega^u X)\right) = 0
$$

For the last column set, we allow $Z_{P,b-1}(\omega^u)$ to be either zero or one:

$$
q_\mathit{last}(X) \cdot \left(Z_{P,b-1}(X)^2 - Z_{P,b-1}(X)\right) = 0
$$

which gives perfect completeness and zero knowledge as before.


# 置换论证

由于 Halo2 电路中的门操作是“局部”的（在当前行或定义的相对行中的单元格上），通常需要将某个任意单元格中的值复制到当前行以供门使用。这是通过一个等式约束来实现的，该约束强制源单元格和目标单元格包含相同的值。

我们通过构建一个表示这些约束的置换来实现这些等式约束，然后在证明中使用置换论证来强制执行它们。

## 符号表示

置换是一个集合到自身的一一映射。置换可以唯一地分解为循环的组合（循环的顺序和每个循环的旋转不影响结果）。

我们有时使用[循环表示法](https://en.wikipedia.org/wiki/Permutation#Cycle_notation)来书写置换。令 $(a\ b\ c)$ 表示一个循环，其中 $a$ 映射到 $b$，$b$ 映射到 $c$，$c$ 映射到 $a$（对于任意大小的循环也有类似的表示）。将两个或多个循环并列表示相应的置换组合。例如，$(a\ b)\ (c\ d)$ 表示将 $a$ 映射到 $b$，$b$ 映射到 $a$，$c$ 映射到 $d$，$d$ 映射到 $c$ 的置换。

## 构建置换

### 目标

我们希望构建一个置换，其中每个等式约束集中的变量形成一个循环。例如，假设我们有一个电路定义了以下等式约束：

• $a \equiv b$
• $a \equiv c$
• $d \equiv e$

由此我们得到等式约束集 $\{a, b, c\}$ 和 $\{d, e\}$。我们希望构建置换：

$$(a\ b\ c)\ (d\ e)$$

该置换将 $[a, b, c, d, e]$ 映射到 $[b, c, a, e, d]$。

### 算法

我们需要跟踪循环集，这是一个[不相交集](https://en.wikipedia.org/wiki/Disjoint-set_data_structure)。对于这个问题，已知有高效的数据结构；为了简单起见，我们选择一种不是渐近最优但易于实现的结构。

我们将当前状态表示为：

• 一个数组 $\mathsf{mapping}$ 用于置换本身；
• 一个辅助数组 $\mathsf{aux}$，用于跟踪每个循环的一个特指元素；
• 另一个数组 $\mathsf{sizes}$，用于跟踪每个循环的大小。

我们有一个不变量：对于给定循环 $C$ 中的每个元素 $x$，$\mathsf{aux}(x)$ 指向同一个元素 $c \∈ C$。这使我们能够快速判断两个给定元素 $x$ 和 $y$ 是否在同一个循环中，只需检查 $\mathsf{aux}(x) = \mathsf{aux}(y)$。此外，$\mathsf{sizes}(\mathsf{aux}(x))$ 给出了包含 $x$ 的循环的大小。（这仅对 $\mathsf{sizes}(\mathsf{aux}(x))$ 保证，而不是对 $\mathsf{sizes}(x)$ 保证。）

算法从表示恒等置换开始：对于所有 $x$，我们设置 $\mathsf{mapping}(x) = x$，$\mathsf{aux}(x) = x$，以及 $\mathsf{sizes}(x) = 1$。

要添加一个等式约束 $\mathit{left} \equiv \mathit{right}$：

1. 检查 $\mathit{left}$ 和 $\mathit{right}$ 是否已经在同一个循环中，即 $\mathsf{aux}(\mathit{left}) = \mathsf{aux}(\mathit{right})$。如果是，则无需做任何操作。
2. 否则，$\mathit{left}$ 和 $\mathit{right}$ 属于不同的循环。通过交换使 $\mathit{left}$ 成为较大的循环，$\mathit{right}$ 成为较小的循环，条件是 $\mathsf{sizes}(\mathsf{aux}(\mathit{left})) < \mathsf{sizes}(\mathsf{aux}(\mathit{right}))$。
3. 设置 $\mathsf{sizes}(\mathsf{aux}(\mathit{left})) :=
        \mathsf{sizes}(\mathsf{aux}(\mathit{left})) + \mathsf{sizes}(\mathsf{aux}(\mathit{right}))$。
4. 沿着右（较小）循环的映射，对于每个元素 $x$，设置 $\mathsf{aux}(x) := \mathsf{aux}(\mathit{left})$。
5. 通过交换 $\mathsf{mapping}(\mathit{left})$ 和 $\mathsf{mapping}(\mathit{right})$ 将较小的循环拼接到较大的循环中。

例如，给定两个不相交的循环 $(A\ B\ C\ D)$ 和 $(E\ F\ G\ H)$：

```plaintext
A +---> B
^       +
|       |
+       v
D <---+ C       E +---> F
                ^       +
                |       |
                +       v
                H <---+ G
```

在添加约束 $B \equiv E$ 后，上述算法生成循环：

```plaintext
A +---> B +-------------+
^                       |
|                       |
+                       v
D <---+ C <---+ E       F
                ^       +
                |       |
                +       v
                H <---+ G
```

### 错误的替代方案

如果我们不检查 $\mathit{left}$ 和 $\mathit{right}$ 是否已经在同一个循环中，那么我们可能会撤销一个等式约束。例如，如果我们有以下约束：

• $a \equiv b$
• $b \equiv c$
• $c \equiv d$
• $b \equiv d$

并且我们尝试仅使用上述算法的第 5 步来添加等式约束，那么我们最终会构建循环 $(a\ b)\ (c\ d)$，而不是正确的 $(a\ b\ c\ d)$。


# 置换论证规范

我们需要检查由拉格朗日基多项式 $v_0, \ldots, v_{m-1}$ 表示的 $m$ 列单元格的置换。

## 单元格标记

我们将这 $m$ 列中的每个单元格标记为 $\mathbb{F}^\times$ 的唯一元素。假设有一个关于这些标签的置换：

$$
\sigma(\mathsf{列}: i, \mathsf{行}: j) = (\mathsf{列}: i', \mathsf{行}: j')
$$

其中循环对应于等式约束集。

> 如果我们考虑标签-值对集合 $\{(\mathit{标签}, \mathit{值})\}$，那么当且仅当通过 $\sigma$ 置换每个对的标签后得到的集合相同时，每个循环中的值才相等：
>
> ![循环示例 (A B C D)。置换前集合为 {(A,7),(B,7),(C,7),(D,7)}，置换后为 {(D,7),(A,7),(B,7),(C,7)} 相同。如果将其中一个7替换为3，置换后的集合将不同。](./permutation-diagram.png)
>
> 由于标签唯一，集合相等性等同于多重集相等性，我们可以使用乘积论证来验证。

## 数学构造

设 $\omega$ 是 $2^k$ 单位根，$\delta$ 是 $T$ 单位根，其中 ${T \cdot 2^S + 1 = p}$，$T$ 为奇数且 ${k \leq S}$。我们使用 $\delta^i \cdot \omega^j \in \mathbb{F}^\times$ 作为置换论证中第 $i$ 列第 $j$ 行单元格的标签。

置换 $\sigma$ 由 $m$ 个多项式 $s_i(X)$ 表示，满足 $s_i(\omega^j) = \delta^{i'} \cdot \omega^{j'}$。

恒等置换由多项式 $\mathsf{ID}_i(\omega^j) = \delta^i \cdot \omega^j$ 表示。

## 乘积验证

使用挑战 $\beta$ 压缩标签-值对为 $\mathit{值} + \beta \cdot \mathit{标签}$，并使用挑战 $\gamma$ 随机化乘积项：

$$
\prod\limits_{i=0}^{m-1} \prod\limits_{j=0}^{n-1} \left(\frac{v_i(\omega^j) + \beta \cdot \delta^i \cdot \omega^j + \gamma}{v_i(\omega^j) + \beta \cdot s_i(\omega^j) + \gamma}\right) = 1
$$

定义累积多项式 $Z_P$：

$$
Z_P(\omega^{j+1}) = Z_P(\omega^j) \prod\limits_{i=0}^{m-1} \frac{v_i(\omega^j) + \beta \cdot \delta^i \cdot \omega^j + \gamma}{v_i(\omega^j) + \beta \cdot s_i(\omega^j) + \gamma}
$$

## 约束规则

强制执行以下约束：

1. 主乘积约束：
   $$
   Z_P(\omega X) \cdot \prod\limits_{i=0}^{m-1} \left(v_i(X) + \beta \cdot s_i(X) + \gamma\right) - Z_P(X) \cdot \prod\limits_{i=0}^{m-1} \left(v_i(X) + \beta \cdot \delta^i \cdot X + \gamma\right) = 0
   $$

2. 初始约束：
   $$
   \ell_0 \cdot (1 - Z_P(X)) = 0
   $$

## 零知识调整

为处理最后 $t$ 行的随机值：

1. 定义选择器：
   - $q_\mathit{blind}$：最后 $t$ 行为1，其余为0
   - $q_\mathit{last}$：仅第 $u$ 行为1（$u = 2^k - t - 1$）

2. 修改后的乘积约束：
   $$
   \big(1 - (q_\mathit{last}(X) + q_\mathit{blind}(X))\big) \cdot \left[Z_P(\omega X) \cdot \prod(\cdots) - Z_P(X) \cdot \prod(\cdots)\right] = 0
   $$

3. 终止约束：
   $$
   q_\mathit{last}(X) \cdot (Z_P(X)^2 - Z_P(X)) = 0
   $$

# 处理大量列

Halo2 的实现实际上并不限制可以启用等式约束的列数。因此，它必须解决上述方法可能导致的乘积规则多项式超出 PLONK 配置的度数限制的问题。虽然可以提高度数限制，但如果其他规则不需要更大的度数，这将是不高效的。

相反，我们将乘积跨 $b$ 组 $m$ 列拆分，使用乘积列 $Z_{P,0}, \ldots Z_{P,b-1}$，并使用另一个规则将乘积从一组列的末尾复制到下一组列的开头。

即，对于 $0 \leq a < b$，我们有：

$\big(1 - (q_\mathit{last}(X) + q_\mathit{blind}(X))\big) \cdot$
$\hspace{1em}\left(Z_{P,a}(\omega X) \cdot \!\prod\limits_{i=am}^{(a+1)m-1}\! \left(v_i(X) + \beta \cdot s_i(X) + \gamma\right) - Z_P(X) \cdot \!\prod\limits_{i=am}^{(a+1)m-1}\! \left{v_i(X) + \beta \cdot \delta^i \cdot X + \gamma\right)\right)$
$\hspace{2em}= 0$

> 为简单起见，这里假设启用了等式约束的列数是 $m$ 的倍数；如果不是，则最后一组列的乘积将少于 $m$ 项。

对于第一组列，我们有：

$$
\ell_0 \cdot (1 - Z_{P,0}(X)) = 0
$$

对于每组后续列，$0 < a < b$，我们使用以下规则将 $Z_{P,a-1}(\omega^u)$ 复制到下一组列的开头，$Z_{P,a}(\omega^0)$：

$$
\ell_0 \cdot \left(Z_{P,a}(X) - Z_{P,a-1}(\omega^u X)\right) = 0
$$

对于最后一组列，我们允许 $Z_{P,b-1}(\omega^u)$ 为零或一：

$$
q_\mathit{last}(X) \cdot \left(Z_{P,b-1}(X)^2 - Z_{P,b-1}(X)\right) = 0
$$

这提供了完美的完备性和零知识，如前所述。