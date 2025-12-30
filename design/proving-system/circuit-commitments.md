# Circuit commitments

## Committing to the circuit assignments

At the start of proof creation, the prover has a table of cell assignments that it claims
satisfy the constraint system. The table has $n = 2^k$ rows, and is broken into advice,
instance, and fixed columns. We define $F_{i,j}$ as the assignment in the $j$th row of
the $i$th fixed column. Without loss of generality, we'll similarly define $A_{i,j}$ to
represent the advice and instance assignments.

> We separate fixed columns here because they are provided by the verifier, whereas the
> advice and instance columns are provided by the prover. In practice, the commitments to
> instance and fixed columns are computed by both the prover and verifier, and only the
> advice commitments are stored in the proof.

To commit to these assignments, we construct Lagrange polynomials of degree $n - 1$ for
each column, over an evaluation domain of size $n$ (where $\omega$ is the $n$th primitive
root of unity):

- $a_i(X)$ interpolates such that $a_i(\omega^j) = A_{i,j}$.
- $f_i(X)$ interpolates such that $f_i(\omega^j) = F_{i,j}$.

We then create a blinding commitment to the polynomial for each column:

$$\mathbf{A} = [\text{Commit}(a_0(X)), \dots, \text{Commit}(a_i(X))]$$
$$\mathbf{F} = [\text{Commit}(f_0(X)), \dots, \text{Commit}(f_i(X))]$$

$\mathbf{F}$ is constructed as part of key generation, using a blinding factor of $1$.
$\mathbf{A}$ is constructed by the prover and sent to the verifier.

## Committing to the lookup permutations

The verifier starts by sampling $\theta$, which is used to keep individual columns within
lookups independent. Then, the prover commits to the permutations for each lookup as
follows:

- Given a lookup with input column polynomials $[A_0(X), \dots, A_{m-1}(X)]$ and table
  column polynomials $[S_0(X), \dots, S_{m-1}(X)]$, the prover constructs two compressed
  polynomials

  $$A_\text{compressed}(X) = \theta^{m-1} A_0(X) + \theta^{m-2} A_1(X) + \dots + \theta A_{m-2}(X) + A_{m-1}(X)$$
  $$S_\text{compressed}(X) = \theta^{m-1} S_0(X) + \theta^{m-2} S_1(X) + \dots + \theta S_{m-2}(X) + S_{m-1}(X)$$

- The prover then permutes $A_\text{compressed}(X)$ and $S_\text{compressed}(X)$ according
  to the [rules of the lookup argument](lookup.md), obtaining $A'(X)$ and $S'(X)$.

The prover creates blinding commitments for all of the lookups

$$\mathbf{L} = \left[ (\text{Commit}(A'(X))), \text{Commit}(S'(X))), \dots \right]$$

and sends them to the verifier.

After the verifier receives $\mathbf{A}$, $\mathbf{F}$, and $\mathbf{L}$, it samples
challenges $\beta$ and $\gamma$ that will be used in the permutation argument and the
remainder of the lookup argument below. (These challenges can be reused because the
arguments are independent.)

## Committing to the equality constraint permutation

Let $c$ be the number of columns that are enabled for equality constraints.

Let $m$ be the maximum number of columns that can be accommodated by a
[column set](permutation.md#spanning-a-large-number-of-columns) without exceeding
the PLONK configuration's maximum constraint degree.

Let $u$ be the number of “usable” rows as defined in the
[Permutation argument](permutation.md#zero-knowledge-adjustment) section.

Let $b = \mathsf{ceiling}(c/m).$

The prover constructs a vector $\mathbf{P}$ of length $bu$ such that for each
column set $0 \leq a < b$ and each row $0 \leq j < u,$

$$
\mathbf{P}_{au + j} = \prod\limits_{i=am}^{\min(c, (a+1)m)-1} \frac{v_i(\omega^j) + \beta \cdot \delta^i \cdot \omega^j + \gamma}{v_i(\omega^j) + \beta \cdot s_i(\omega^j) + \gamma}.
$$

The prover then computes a running product of $\mathbf{P}$, starting at $1$,
and a vector of polynomials $Z_{P,0..b-1}$ that each have a Lagrange basis
representation corresponding to a $u$-sized slice of this running product, as
described in the [Permutation argument](permutation.md#argument-specification)
section.

The prover creates blinding commitments to each $Z_{P,a}$ polynomial:

$$\mathbf{Z_P} = \left[\text{Commit}(Z_{P,0}(X)), \dots, \text{Commit}(Z_{P,b-1}(X))\right]$$

and sends them to the verifier.

## Committing to the lookup permutation product columns

In addition to committing to the individual permuted lookups, for each lookup,
the prover needs to commit to the permutation product column:

- The prover constructs a vector $P$:

$$
P_j = \frac{(A_\text{compressed}(\omega^j) + \beta)(S_\text{compressed}(\omega^j) + \gamma)}{(A'(\omega^j) + \beta)(S'(\omega^j) + \gamma)}
$$

- The prover constructs a polynomial $Z_L$ which has a Lagrange basis representation
  corresponding to a running product of $P$, starting at $Z_L(1) = 1$.

$\beta$ and $\gamma$ are used to combine the permutation arguments for $A'(X)$ and $S'(X)$
while keeping them independent. The important thing here is that the verifier samples
$\beta$ and $\gamma$ after the prover has created $\mathbf{A}$, $\mathbf{F}$, and
$\mathbf{L}$ (and thus committed to all the cell values used in lookup columns, as well
as $A'(X)$ and $S'(X)$ for each lookup).

As before, the prover creates blinding commitments to each $Z_L$ polynomial:

$$\mathbf{Z_L} = \left[\text{Commit}(Z_L(X)), \dots \right]$$

and sends them to the verifier.

# 电路承诺

## 对电路赋值进行承诺

在证明创建的初始阶段，证明者有一个单元格赋值的表格，声称这些赋值满足约束系统。该表格有 $n = 2^k$ 行，并分为建议列（advice）、实例列（instance）和固定列（fixed）。我们将 $F_{i,j}$ 定义为第 $i$ 个固定列的第 $j$ 行的赋值。不失一般性，我们类似地定义 $A_{i,j}$ 来表示建议列和实例列的赋值。

> 这里我们将固定列单独列出，因为它们由验证者提供，而建议列和实例列由证明者提供。实际上，实例列和固定列的承诺由证明者和验证者共同计算，只有建议列的承诺存储在证明中。

为了对这些赋值进行承诺，我们为每一列构造一个次数为 $n - 1$ 的拉格朗日多项式，定义在大小为 $n$ 的求值域上（其中 $\omega$ 是第 $n$ 个单位原根）：

- $a_i(X)$ 插值满足 $a_i(\omega^j) = A_{i,j}$。
- $f_i(X)$ 插值满足 $f_i(\omega^j) = F_{i,j}$。

然后，我们对每一列的多项式创建一个盲化承诺：

$$\mathbf{A} = [\text{Commit}(a_0(X)), \dots, \text{Commit}(a_i(X))]$$
$$\mathbf{F} = [\text{Commit}(f_0(X)), \dots, \text{Commit}(f_i(X))]$$

$\mathbf{F}$ 在密钥生成阶段构造，使用盲化因子 $1$。$\mathbf{A}$ 由证明者构造并发送给验证者。

## 对查找置换进行承诺

验证者首先采样 $\theta$，用于保持查找中各个列的独立性。然后，证明者对每个查找的置换进行如下承诺：

- 给定一个查找，其输入列多项式为 $[A_0(X), \dots, A_{m-1}(X)]$，表格列多项式为 $[S_0(X), \dots, S_{m-1}(X)]$，证明者构造两个压缩多项式：

  $$A_\text{压缩}(X) = \theta^{m-1} A_0(X) + \theta^{m-2} A_1(X) + \dots + \theta A_{m-2}(X) + A_{m-1}(X)$$
  $$S_\text{压缩}(X) = \theta^{m-1} S_0(X) + \theta^{m-2} S_1(X) + \dots + \theta S_{m-2}(X) + S_{m-1}(X)$$

- 证明者根据[查找参数的规则](lookup.md)对 $A_\text{压缩}(X)$ 和 $S_\text{压缩}(X)$ 进行置换，得到 $A'(X)$ 和 $S'(X)$。

证明者为所有查找创建盲化承诺：

$$\mathbf{L} = \left[ (\text{Commit}(A'(X))), \text{Commit}(S'(X))), \dots \right]$$

并将其发送给验证者。

验证者在收到 $\mathbf{A}$、$\mathbf{F}$ 和 $\mathbf{L}$ 后，采样挑战值 $\beta$ 和 $\gamma$，这些值将用于置换参数和后续的查找参数。（这些挑战值可以复用，因为这些参数是独立的。）

## 对等式约束置换进行承诺

设 $c$ 为启用等式约束的列数。

设 $m$ 为在不超出 PLONK 配置的最大约束次数的情况下，一个[列集](permutation.md#spanning-a-large-number-of-columns)可以容纳的最大列数。

设 $u$ 为[置换参数](permutation.md#zero-knowledge-adjustment)部分定义的“可用”行数。

设 $b = \lceil c/m \rceil$。

证明者构造一个长度为 $bu$ 的向量 $\mathbf{P}$，使得对于每个列集 $0 \leq a < b$ 和每行 $0 \leq j < u$，

$$
\mathbf{P}_{au + j} = \prod\limits_{i=am}^{\min(c, (a+1)m)-1} \frac{v_i(\omega^j) + \beta \cdot \delta^i \cdot \omega^j + \gamma}{v_i(\omega^j) + \beta \cdot s_i(\omega^j) + \gamma}.
$$

证明者然后计算 $\mathbf{P}$ 的累乘积，从 $1$ 开始，并构造一个多项式向量 $Z_{P,0..b-1}$，每个多项式的拉格朗日基表示对应于这个累乘积的一个大小为 $u$ 的切片，如[置换参数](permutation.md#argument-specification)部分所述。

证明者为每个 $Z_{P,a}$ 多项式创建盲化承诺：

$$\mathbf{Z_P} = \left[\text{Commit}(Z_{P,0}(X)), \dots, \text{Commit}(Z_{P,b-1}(X))\right]$$

并将其发送给验证者。

## 对查找置换乘积列进行承诺

除了对单个置换查找进行承诺外，对于每个查找，证明者还需要对置换乘积列进行承诺：

- 证明者构造一个向量 $P$：

$$
P_j = \frac{(A_\text{压缩}(\omega^j) + \beta)(S_\text{压缩}(\omega^j) + \gamma)}{(A'(\omega^j) + \beta)(S'(\omega^j) + \gamma)}
$$

- 证明者构造一个多项式 $Z_L$，其拉格朗日基表示对应于 $P$ 的累乘积，从 $Z_L(1) = 1$ 开始。

$\beta$ 和 $\gamma$ 用于将 $A'(X)$ 和 $S'(X)$ 的置换参数结合起来，同时保持它们的独立性。这里的关键在于，验证者在证明者构造完 $\mathbf{A}$、$\mathbf{F}$ 和 $\mathbf{L}$（即已对所有查找列中的单元格值以及每个查找的 $A'(X)$ 和 $S'(X)$ 进行承诺）之后才采样 $\beta$ 和 $\gamma$。

与之前类似，证明者为每个 $Z_L$ 多项式创建盲化承诺：

$$\mathbf{Z_L} = \left[\text{Commit}(Z_L(X)), \dots \right]$$

并将其发送给验证者。
