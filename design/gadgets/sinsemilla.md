# Sinsemilla

## Overview
Sinsemilla is a collision-resistant hash function and commitment scheme designed to be efficient in algebraic circuit models that support [lookups](https://zcash.github.io/halo2/design/proving-system/lookup.html), such as PLONK or Halo 2.

The security properties of Sinsemilla are similar to Pedersen hashes; it is **not** designed to be used where a random oracle, PRF, or preimage-resistant hash is required. **The only claimed security property of the hash function is collision-resistance for fixed-length inputs.**

Sinsemilla is roughly 4 times less efficient than the algebraic hashes Rescue and Poseidon inside a circuit, but around 19 times more efficient than Rescue outside a circuit. Unlike either of these hashes, the collision resistance property of Sinsemilla can be proven based on cryptographic assumptions that have been well-established for at least 20 years. Sinsemilla can also be used as a computationally binding and perfectly hiding commitment scheme.

The general approach is to split the message into $k$-bit pieces, and for each piece, select from a table of $2^k$ bases in our cryptographic group. We combine the selected bases using a double-and-add algorithm. This ends up being provably as secure as a vector Pedersen hash, and makes advantageous use of the lookup facility supported by Halo 2.

## Description

This section is an outline of how Sinsemilla works: for the normative specification, refer to [§5.4.1.9 Sinsemilla Hash Function](https://zips.z.cash/protocol/protocol.pdf#concretesinsemillahash) in the protocol spec. The incomplete point addition operator, ⸭, that we use below is also defined there.

Let $\mathbb{G}$ be a cryptographic group of prime order $q$. We write $\mathbb{G}$ additively, with identity $\mathcal{O}$, and using $[m] P$ for scalar multiplication of $P$ by $m$.

Let $k \geq 1$ be an integer chosen based on efficiency considerations (the table size will be $2^k$). Let $n$ be an integer, fixed for each instantiation, such that messages are $kn$ bits, where $2^n \leq \frac{q-1}{2}$. We use zero-padding to the next multiple of $k$ bits if necessary.

$\textsf{Setup}$: Choose $Q$ and $P[0..2^k - 1]$ as $2^k + 1$ independent, verifiably random generators of $\mathbb{G}$, using a suitable hash into $\mathbb{G}$, such that none of $Q$ or $P[0..2^k - 1]$ are $\mathcal{O}$.

> In Orchard, we define $Q$ to be dependent on a domain separator $D$. The protocol specification uses $\mathcal{Q}(D)$ in place of $Q$ and $\mathcal{S}(m)$ in place of $P[m]$.

$\textsf{Hash}(M)$:
- Split $M$ into $n$ groups of $k$ bits. Interpret each group as a $k$-bit little-endian integer $m_i$.
- let $\mathsf{Acc}_0 := Q$
- for $i$ from $0$ up to $n-1$:
  - let $\mathsf{Acc}_{i+1} := (\mathsf{Acc}_i \;⸭\; P[m_{i+1}]) \;⸭\; \mathsf{Acc}_i$
- return $\mathsf{Acc}_n$

Let $\textsf{ShortHash}(M)$ be the $x$-coordinate of $\textsf{Hash}(M)$. (This assumes that $\mathbb{G}$ is a prime-order elliptic curve in short Weierstrass form, as is the case for Pallas and Vesta.)

> It is slightly more efficient to express a double-and-add $[2] A + R$ as $(A + R) + A$. We also use incomplete additions: it is shown in the [Sinsemilla security argument](https://zips.z.cash/protocol/protocol.pdf#sinsemillasecurity) that in the case where $\mathbb{G}$ is a prime-order short Weierstrass elliptic curve, an exceptional case for addition would lead to finding a discrete logarithm, which can be assumed to occur with negligible probability even for adversarial input.

### Use as a commitment scheme
Choose another generator $H$ independently of $Q$ and $P[0..2^k - 1]$.

The randomness $r$ for a commitment is chosen uniformly on $[0, q)$.

Let $\textsf{Commit}_r(M) = \textsf{Hash}(M) \;⸭\; [r] H$.

Let $\textsf{ShortCommit}_r(M)$ be the $x\text{-coordinate}$ of $\textsf{Commit}_r(M)$. (This again assumes that $\mathbb{G}$ is a prime-order elliptic curve in short Weierstrass form.)

Note that unlike a simple Pedersen commitment, this commitment scheme ($\textsf{Commit}$ or $\textsf{ShortCommit}$) is not additively homomorphic.

## Efficient implementation
The aim of the design is to optimize the number of bits that can be processed for each step of the algorithm (which requires a doubling and addition in $\mathbb{G}$) for a given table size. Using a single table of size $2^k$ group elements, we can process $k$ bits at a time.

### Incomplete addition

In each step of Sinsemilla we want to compute $A_{i+1} := (A_i \;⸭\; P_i) \;⸭\; A_i$. Let
$R_i := A_i \;⸭\; P_i$ be the intermediate result such that $A_{i+1} := A_i \;⸭\; R_i$.
Recalling the [incomplete addition formulae](ecc/addition.md#incomplete-addition):

$$
\begin{aligned}
x_3 &= \left(\frac{y_1 - y_2}{x_1 - x_2}\right)^2 - x_1 - x_2 \\
y_3 &= \frac{y_1 - y_2}{x_1 - x_2} \cdot (x_1 - x_3) - y_1 \\
\end{aligned}
$$

Let $\lambda = \frac{y_1 - y_2}{x_1 - x_2}$. Substituting the coordinates for each of the
incomplete additions in turn, and rearranging, we get

$$
\begin{aligned}
\lambda_{1,i} &= \frac{y_{A,i} - y_{P,i}}{x_{A,i} - x_{P,i}} \\
&\implies y_{A,i} - y_{P,i} = \lambda_{1,i} \cdot (x_{A,i} - x_{P,i}) \\
&\implies y_{P,i} = y_{A,i} - \lambda_{1,i} \cdot (x_{A,i} - x_{P,i}) \\
x_{R,i} &= \lambda_{1,i}^2 - x_{A,i} - x_{P,i} \\
y_{R,i} &= \lambda_{1,i} \cdot (x_{A,i} - x_{R,i}) - y_{A,i} \\
\end{aligned}
$$
and
$$
\begin{aligned}
\lambda_{2,i} &= \frac{y_{A,i} - y_{R,i}}{x_{A,i} - x_{R,i}} \\
&\implies y_{A,i} - y_{R,i} = \lambda_{2,i} \cdot (x_{A,i} - x_{R,i}) \\
&\implies y_{A,i} - \left( \lambda_{1,i} \cdot (x_{A,i} - x_{R,i}) - y_{A,i} \right) = \lambda_{2,i} \cdot (x_{A,i} - x_{R,i}) \\
&\implies 2 \cdot y_{A,i} = (\lambda_{1,i} + \lambda_{2,i}) \cdot (x_{A,i} - x_{R,i}) \\
x_{A,i+1} &= \lambda_{2,i}^2 - x_{A,i} - x_{R,i} \\
y_{A,i+1} &= \lambda_{2,i} \cdot (x_{A,i} - x_{A,i+1}) - y_{A,i}. \\
\end{aligned}
$$

### Constraint program
Let $\mathcal{P} = \left\{(j,\, x_{P[j]},\, y_{P[j]}) \text{ for } j \in \{0..2^k - 1\}\right\}$.

Input: $m_{1..=n}$. (The message words are 1-indexed here, as in the [protocol spec](https://zips.z.cash/protocol/nu5.pdf#concretesinsemillahash), but we start the loop from $i = 0$ so that $(x_{A,i}, y_{A,i})$ corresponds to $\mathsf{Acc}_i$ in the protocol spec.)

Output: $(x_{A,n},\, y_{A,n})$.

- $(x_{A,0},\, y_{A,0}) = Q$
- for $i$ from $0$ up to $n-1$:
  - $y_{P,i} = y_{A,i} - \lambda_{1,i} \cdot (x_{A,i} - x_{P,i})$
  - $x_{R,i} = \lambda_{1,i}^2 - x_{A,i} - x_{P,i}$
  - $2 \cdot y_{A,i} = (\lambda_{1,i} + \lambda_{2,i}) \cdot (x_{A,i} - x_{R,i})$
  - $(m_{i+1},\, x_{P,i},\, y_{P,i}) \in \mathcal{P}$
  - $\lambda_{2,i}^2 = x_{A,i+1} + x_{R,i} + x_{A,i}$
  - $\lambda_{2,i} \cdot (x_{A,i} - x_{A,i+1}) = y_{A,i} + y_{A,i+1}$


## PLONK / Halo 2 constraints

### Message decomposition
We have an $n$-bit message $m = m_1 + 2^k m_2 + ... + 2^{k\cdot (n-1)} m_n$. (Note that the message words are 1-indexed as in the [protocol spec](https://zips.z.cash/protocol/nu5.pdf#concretesinsemillahash).)

Initialise the running sum $z_0 = \alpha$ and define $z_{i + 1} := \frac{z_{i} - m_{i+1}}{2^K}$. We will end up with $z_n = 0.$

Rearranging gives us an expression for each word of the original message
$m_{i+1} = z_{i} - 2^k \cdot z_{i + 1}$, which we can look up in the table. We position
$z_{i}$ and $z_{i + 1}$ in adjacent rows of the same column, so we can sequentially apply
the constraint across the entire message.

In other words, $z_{n-i} = \sum\limits_{h=0}^{i-1} 2^{kh} \cdot m_{h+1}$.

> For a little-endian decomposition as used here, the running sum is initialized to the scalar and ends at 0. For a big-endian decomposition as used in [variable-base scalar multiplication](https://hackmd.io/o9EzZBwxSWSi08kQ_fMIOw), the running sum would start at 0 and end with recovering the original scalar.

### Efficient packing

The running sum only applies to message words within a single field element. That means if
$n \geq \mathtt{PrimeField::NUM\_BITS}$ then we will need several disjoint running sums. A
longer message can be constructed by splitting the message words across several field
elements, and then running several instances of the constraints below.

The expression for $m_{i+1}$ above requires $n + 1$ rows in the $z_{i}$ column, leaving a
one-row gap in adjacent columns and making $\mathsf{Acc}_i$ tricker to accumulate. In
order to support chaining multiple field elements without a gap, we use a slightly more
complicated expression for $m_{i+1}$ that includes a selector:

$$m_{i+1} = z_{i} - 2^k \cdot q_{run,i} \cdot z_{i+1}$$

This effectively forces $\mathbf{z}_n$ to zero for the last step of each element, which
allows the cell that would have been $\mathbf{z}_n$ to be used to reinitialize the running
sum for the next element.

With this sorted out, the incomplete addition accumulator can eliminate $y_{A,i}$ almost
entirely, by substituting for $x$ and $\lambda$ values in the current and next rows. The
two exceptions are at the start of Sinsemilla (where we need to constrain the accumulator
to have initial value $Q$), and the end (where we need to witness $y_{A,n}$ for use
outside of Sinsemilla).

### Selectors

We need a total of four logical selectors to:

- Control the Sinsemilla gate and lookup.
- Distinguish between the last message word in a running sum and its earlier words.
- Mark the start of Sinsemilla.
- Mark the end of Sinsemilla.

We use regular selector columns for the Sinsemilla gate selector $q_{S1}$ and Sinsemilla
start selector $q_{S4}.$ The other two selectors are synthesized from a single fixed
column $q_{S2}$ as follows:

$$
\begin{aligned}
q_{S3}  &= q_{S2} \cdot (q_{S2} - 1) \\
q_{run} &= q_{S2} - q_{S3} \\
\end{aligned}
$$

$$
\begin{array}{|c|c|c|}
\hline
q_{S2} & q_{S3} & q_{run} \\\hline
   0   &   0    &    0    \\\hline
   1   &   0    &    1    \\\hline
   2   &   2    &    0    \\\hline
\end{array}
$$

We set $q_{S2}$ to $1$ on most Sinsemilla rows, and $0$ for the last step of each element,
except for the last element where it is set to $2$. We can then use $q_{S3}$ to toggle
between constraining the substituted $y_{A,i+1}$ on adjacent rows, and the witnessed
$y_{A,n}$ at the end of Sinsemilla:

$$
\lambda_{2,i} \cdot (x_{A,i} - x_{A,i+1}) = y_{A,i} + \frac{2 - q_{S3}}{2} \cdot y_{A,i+1} + \frac{q_{S3}}{2} \cdot y_{A,n}
$$

### Generator lookup table
The Sinsemilla circuit makes use of $2^{10}$ pre-computed random generators. These are loaded into a lookup table:
$$
\begin{array}{|c|c|c|}
\hline
 table_{idx} & table_x         & table_y         \\\hline
 0           & x_{P[0]}        & y_{P[0]}        \\\hline
 1           & x_{P[1]}        & y_{P[1]}        \\\hline
 2           & x_{P[2]}        & y_{P[2]}        \\\hline
 \vdots      & \vdots          & \vdots          \\\hline
 2^{10} - 1  & x_{P[2^{10}-1]} & y_{P[2^{10}-1]} \\\hline
\end{array}
$$

### Layout
$$
\begin{array}{|c|c|c|c|c|c|c|c|c|c|}
\hline
\text{Step} &    x_A     &    x_P      &   bits   &    \lambda_1     &   \lambda_2      & q_{S1} & q_{S2} &    q_{S4}  & \textsf{fixed\_y\_Q}\\\hline
    0       & x_Q        & x_{P[m_1]}  & z_0      & \lambda_{1,0}    & \lambda_{2,0}    & 1      & 1      &     1      &    y_Q              \\\hline
    1       & x_{A,1}    & x_{P[m_2]}  & z_1      & \lambda_{1,1}    & \lambda_{2,1}    & 1      & 1      &     0      &     0               \\\hline
    2       & x_{A,2}    & x_{P[m_3]}  & z_2      & \lambda_{1,2}    & \lambda_{2,2}    & 1      & 1      &     0      &     0               \\\hline
  \vdots    & \vdots     & \vdots      & \vdots   & \vdots           & \vdots           & 1      & 1      &     0      &     0               \\\hline
   n-1      & x_{A,n-1}  & x_{P[m_n]}  & z_{n-1}  & \lambda_{1,n-1}  & \lambda_{2,n-1}  & 1      & 0      &     0      &     0               \\\hline
    0'      & x'_{A,0}   & x_{P[m'_1]} & z'_0     & \lambda'_{1,0}   & \lambda'_{2,0}   & 1      & 1      &     0      &     0               \\\hline
    1'      & x'_{A,1}   & x_{P[m'_2]} & z'_1     & \lambda'_{1,1}   & \lambda'_{2,1}   & 1      & 1      &     0      &     0               \\\hline
    2'      & x'_{A,2}   & x_{P[m'_3]} & z'_2     & \lambda'_{1,2}   & \lambda'_{2,2}   & 1      & 1      &     0      &     0               \\\hline
  \vdots    & \vdots     & \vdots      & \vdots   & \vdots           & \vdots           & 1      & 1      &     0      &     0               \\\hline
   n-1'     & x'_{A,n-1} & x_{P[m'_n]} & z'_{n-1} & \lambda'_{1,n-1} & \lambda'_{2,n-1} & 1      & 2      &     0      &     0               \\\hline
    n'      &  x'_{A,n}  &             &          &       y_{A,n}    &                  & 0      & 0      &     0      &     0               \\\hline
\end{array}
$$

$x_Q$, $z_0$, $z'_0$, etc. are copied in using equality constraints.

### Optimized Sinsemilla gate
$$
\begin{array}{lrcl}
\text{For } i \in [0, n), \text{ let} &x_{R,i} &=& \lambda_{1,i}^2 - x_{A,i} - x_{P,i} \\
                                      &Y_{A,i} &=& (\lambda_{1,i} + \lambda_{2,i}) \cdot (x_{A,i} - x_{R,i}) \\
                                      &y_{P,i} &=& Y_{A,i}/2 - \lambda_{1,i} \cdot (x_{A,i} - x_{P,i}) \\
                                      &m_{i+1} &=& z_{i} - q_{run,i} \cdot z_{i+1} \cdot 2^k \\
                                      &q_{run} &=& q_{S2} - q_{S3} \\
                                      &q_{S3}  &=& q_{S2} \cdot (q_{S2} - 1)
\end{array}
$$

The Halo 2 circuit API can automatically substitute $y_{P,i}$, $x_{R,i}$, $Y_{A,i}$, and
$Y_{A,i+1}$, so we don't need to do that manually.

- $x_{A,0} = x_Q$
- $2 \cdot y_Q = Y_{A,0}$
- for $i$ from $0$ up to $n-1$:
  - $(m_{i+1},\, x_{P,i},\, y_{P,i}) \in \mathcal{P}$
  - $\lambda_{2,i}^2 = x_{A,i+1} + x_{R,i} + x_{A,i}$
  - $4 \cdot \lambda_{2,i} \cdot (x_{A,i} - x_{A,i+1}) = 2 \cdot Y_{A,i} + (2 - q_{S3}) \cdot Y_{A,i+1} + 2 q_{S3} \cdot y_{A,n}$

Note that each term of the last constraint is multiplied by $4$ relative to the constraint program given earlier. This is a small optimization that avoids divisions by $2$.

By gating the lookup expression on $q_{S1}$, we avoid the need to fill in unused cells with dummy values to pass the lookup argument. The optimized lookup value (using a default index of $0$) is:

$$
\begin{array}{ll}
(&q_{S1} \cdot m_{i+1}, \\
 &q_{S1} \cdot x_{P,i} + (1 - q_{S1}) \cdot x_{P,0}, \\
 &q_{S1} \cdot y_{P,i} + (1 - q_{S1}) \cdot y_{P,0} \;\;\;)
\end{array}
$$

This increases the degree of the lookup argument to $6$.

$$
\begin{array}{|c|l|}
\hline
\text{Degree} & \text{Constraint} \\\hline
4   & q_{S4} \cdot (2 \cdot y_Q - Y_{A,0}) = 0 \\\hline
6   & q_{S1,i} \Rightarrow (m_{i+1},\, x_{P,i},\, y_{P,i}) \in \mathcal{P} \\\hline
3   & q_{S1,i} \cdot \big(\lambda_{2,i}^2 - (x_{A,i+1} + x_{R,i} + x_{A,i})\big) \\\hline
5   & q_{S1,i} \cdot \left(4 \cdot \lambda_{2,i} \cdot (x_{A,i} - x_{A,i+1}) - (2 \cdot Y_{A,i} + (2 - q_{S3,i}) \cdot Y_{A,i+1} + 2 \cdot q_{S3,i} \cdot y_{A,n})\right) = 0 \\\hline
\end{array}
$$


# Sinsemilla

## 概述
Sinsemilla 是一种抗碰撞的哈希函数和承诺方案，旨在在支持[查找表](https://zcash.github.io/halo2/design/proving-system/lookup.html)的代数电路模型（如 PLONK 或 Halo 2）中高效运行。

Sinsemilla 的安全性与 Pedersen 哈希相似；它**不**设计用于需要随机预言机、伪随机函数（PRF）或抗原像哈希的场景。​**哈希函数唯一声称的安全属性是固定长度输入的抗碰撞性。​**

Sinsemilla 在电路内的效率比 Rescue 和 Poseidon 等代数哈希低约 4 倍，但在电路外的效率比 Rescue 高约 19 倍。与这些哈希不同，Sinsemilla 的抗碰撞性可以基于至少 20 年来广泛研究的密码学假设进行证明。Sinsemilla 还可以用作计算绑定且完全隐藏的承诺方案。

其基本思路是将消息分割为 $k$ 位的片段，然后为每个片段从密码学群的 $2^k$ 个基表中选择一个基。我们使用双倍加算法（double-and-add）来组合这些基。这种方法最终被证明与向量 Pedersen 哈希一样安全，并且充分利用了 Halo 2 支持的查找表功能。

## 描述
本节概述了 Sinsemilla 的工作原理：具体规范请参考协议规范中的[§5.4.1.9 Sinsemilla 哈希函数](https://zips.z.cash/protocol/protocol.pdf#concretesinsemillahash)。我们在下面使用的不完全点加法运算符 ⸭ 也在那里定义。

设 $\mathbb{G}$ 为一个素数阶 $q$ 的密码学群。我们以加法形式表示 $\mathbb{G}$，其单位元为 $\mathcal{O}$，并使用 $[m] P$ 表示 $P$ 乘以 $m$ 的标量乘法。

设 $k \geq 1$ 为基于效率考虑选择的整数（表的大小为 $2^k$）。设 $n$ 为每个实例化中固定的整数，使得消息为 $kn$ 位，其中 $2^n \leq \frac{q-1}{2}$。如有必要，我们使用零填充到 $k$ 位的下一个倍数。

$\textsf{Setup}$：选择 $Q$ 和 $P[0..2^k - 1]$ 作为 $\mathbb{G}$ 的 $2^k + 1$ 个独立的、可验证的随机生成器，使用适当的哈希函数映射到 $\mathbb{G}$，确保 $Q$ 和 $P[0..2^k - 1]$ 都不为 $\mathcal{O}$。

> 在 Orchard 中，我们定义 $Q$ 依赖于域分隔符 $D$。协议规范使用 $\mathcal{Q}(D)$ 代替 $Q$，使用 $\mathcal{S}(m)$ 代替 $P[m]$。

$\textsf{Hash}(M)$：
- 将 $M$ 分割为 $n$ 组，每组 $k$ 位。将每组解释为 $k$ 位小端整数 $m_i$。
- 令 $\mathsf{Acc}_0 := Q$
- 对于 $i$ 从 $0$ 到 $n-1$：
  - 令 $\mathsf{Acc}_{i+1} := (\mathsf{Acc}_i \;⸭\; P[m_{i+1}]) \;⸭\; \mathsf{Acc}_i$
- 返回 $\mathsf{Acc}_n$

设 $\textsf{ShortHash}(M)$ 为 $\textsf{Hash}(M)$ 的 $x$ 坐标。（假设 $\mathbb{G}$ 是一个素数阶的短 Weierstrass 椭圆曲线，如 Pallas 和 Vesta。）

> 将双倍加 $[2] A + R$ 表示为 $(A + R) + A$ 会更高效。我们还使用不完全加法：在 [Sinsemilla 安全论证](https://zips.z.cash/protocol/protocol.pdf#sinsemillasecurity) 中显示，当 $\mathbb{G}$ 是素数阶短 Weierstrass 椭圆曲线时，加法的异常情况将导致发现离散对数，可以假设在对抗性输入下发生的概率可忽略不计。

### 作为承诺方案使用
选择另一个生成器 $H$，独立于 $Q$ 和 $P[0..2^k - 1]$。

承诺的随机性 $r$ 在 $[0, q)$ 上均匀选择。

设 $\textsf{Commit}_r(M) = \textsf{Hash}(M) \;⸭\; [r] H$。

设 $\textsf{ShortCommit}_r(M)$ 为 $\textsf{Commit}_r(M)$ 的 $x$ 坐标。（再次假设 $\mathbb{G}$ 是素数阶短 Weierstrass 椭圆曲线。）

注意，与简单的 Pedersen 承诺不同，此承诺方案（$\textsf{Commit}$ 或 $\textsf{ShortCommit}$）不是加法同态的。

## 高效实现
设计的目标是优化算法每一步可以处理的比特数（需要在 $\mathbb{G}$ 中进行一次双倍和加法），以应对给定的表大小。使用大小为 $2^k$ 的单个群元素表，我们可以一次处理 $k$ 位。

### 不完全加法

在 Sinsemilla 的每一步中，我们希望计算 $A_{i+1} := (A_i \;⸭\; P_i) \;⸭\; A_i$。设
$R_i := A_i \;⸭\; P_i$ 为中间结果，使得 $A_{i+1} := A_i \;⸭\; R_i$。
回顾[不完全加法公式](ecc/addition.md#incomplete-addition)：

$$
\begin{aligned}
x_3 &= \left(\frac{y_1 - y_2}{x_1 - x_2}\right)^2 - x_1 - x_2 \\
y_3 &= \frac{y_1 - y_2}{x_1 - x_2} \cdot (x_1 - x_3) - y_1 \\
\end{aligned}
$$

设 $\lambda = \frac{y_1 - y_2}{x_1 - x_2}$。依次替换每个不完全加法的坐标并重新排列，我们得到

$$
\begin{aligned}
\lambda_{1,i} &= \frac{y_{A,i} - y_{P,i}}{x_{A,i} - x_{P,i}} \\
&\implies y_{A,i} - y_{P,i} = \lambda_{1,i} \cdot (x_{A,i} - x_{P,i}) \\
&\implies y_{P,i} = y_{A,i} - \lambda_{1,i} \cdot (x_{A,i} - x_{P,i}) \\
x_{R,i} &= \lambda_{1,i}^2 - x_{A,i} - x_{P,i} \\
y_{R,i} &= \lambda_{1,i} \cdot (x_{A,i} - x_{R,i}) - y_{A,i} \\
\end{aligned}
$$
和
$$
\begin{aligned}
\lambda_{2,i} &= \frac{y_{A,i} - y_{R,i}}{x_{A,i} - x_{R,i}} \\
&\implies y_{A,i} - y_{R,i} = \lambda_{2,i} \cdot (x_{A,i} - x_{R,i}) \\
&\implies y_{A,i} - \left( \lambda_{1,i} \cdot (x_{A,i} - x_{R,i}) - y_{A,i} \right) = \lambda_{2,i} \cdot (x_{A,i} - x_{R,i}) \\
&\implies 2 \cdot y_{A,i} = (\lambda_{1,i} + \lambda_{2,i}) \cdot (x_{A,i} - x_{R,i}) \\
x_{A,i+1} &= \lambda_{2,i}^2 - x_{A,i} - x_{R,i} \\
y_{A,i+1} &= \lambda_{2,i} \cdot (x_{A,i} - x_{A,i+1}) - y_{A,i}. \\
\end{aligned}
$$

### 约束程序
设 $\mathcal{P} = \left\{(j,\, x_{P[j]},\, y_{P[j]}) \text{ 对于 } j \in \{0..2^k - 1\}\right\}$。

输入：$m_{1..=n}$。（消息字在这里是 1 索引的，如[协议规范](https://zips.z.cash/protocol/nu5.pdf#concretesinsemillahash)中所述，但我们将循环从 $i = 0$ 开始，以便 $(x_{A,i}, y_{A,i})$ 对应于协议规范中的 $\mathsf{Acc}_i$。）

输出：$(x_{A,n},\, y_{A,n})$。

- $(x_{A,0},\, y_{A,0}) = Q$
- 对于 $i$ 从 $0$ 到 $n-1$：
  - $y_{P,i} = y_{A,i} - \lambda_{1,i} \cdot (x_{A,i} - x_{P,i})$
  - $x_{R,i} = \lambda_{1,i}^2 - x_{A,i} - x_{P,i}$
  - $2 \cdot y_{A,i} = (\lambda_{1,i} + \lambda_{2,i}) \cdot (x_{A,i} - x_{R,i})$
  - $(m_{i+1},\, x_{P,i},\, y_{P,i}) \in \mathcal{P}$
  - $\lambda_{2,i}^2 = x_{A,i+1} + x_{R,i} + x_{A,i}$
  - $\lambda_{2,i} \cdot (x_{A,i} - x_{A,i+1}) = y_{A,i} + y_{A,i+1}$

## PLONK / Halo 2 约束

### 消息分解
我们有一个 $n$ 位的消息 $m = m_1 + 2^k m_2 + ... + 2^{k\cdot (n-1)} m_n$。（注意，消息字是 1 索引的，如[协议规范](https://zips.z.cash/protocol/nu5.pdf#concretesinsemillahash)中所述。）

初始化运行和 $z_0 = \alpha$ 并定义 $z_{i + 1} := \frac{z_{i} - m_{i+1}}{2^K}$。我们最终得到 $z_n = 0.$

重新排列得到原始消息的每个字的表达式
$m_{i+1} = z_{i} - 2^k \cdot z_{i + 1}$，我们可以在表中查找。我们将
$z_{i}$ 和 $z_{i + 1}$ 放置在相邻行的同一列中，以便我们可以顺序地将约束应用于整个消息。

换句话说，$z_{n-i} = \sum\limits_{h=0}^{i-1} 2^{kh} \cdot m_{h+1}$。

> 对于此处使用的小端分解，运行和初始化为标量并最终为 0。对于[变基标量乘法](https://hackmd.io/o9EzZBwxSWSi08kQ_fMIOw)中使用的大端分解，运行和将从 0 开始并最终恢复原始标量。

### 高效打包

运行和仅适用于单个域元素内的消息字。这意味着如果
$n \geq \mathtt{PrimeField::NUM\_BITS}$，那么我们将需要多个不相交的运行和。可以通过将消息字分割到多个域元素中来构造更长的消息，然后运行多个约束实例。

上述 $m_{i+1}$ 的表达式需要在 $z_{i}$ 列中占用 $n + 1$ 行，导致相邻列中出现一行间隙，并使 $\mathsf{Acc}_i$ 更难累积。为了支持在没有间隙的情况下链接多个域元素，我们使用了一个稍微复杂的 $m_{i+1}$ 表达式，其中包括一个选择器：

$$m_{i+1} = z_{i} - 2^k \cdot q_{run,i} \cdot z_{i+1}$$

这有效地强制 $\mathbf{z}_n$ 在每个元素的最后一步为零，从而允许将本应为 $\mathbf{z}_n$ 的单元格用于重新初始化下一个元素的运行和。

通过解决这个问题，不完全加法累加器可以通过在当前和下一行中替换 $x$ 和 $\lambda$ 值来几乎完全消除 $y_{A,i}$。唯一的两个例外是在 Sinsemilla 的开始（我们需要约束累加器的初始值为 $Q$）和结束（我们需要见证 $y_{A,n}$ 以便在 Sinsemilla 之外使用）。

### 选择器

我们总共需要四个逻辑选择器来：

- 控制 Sinsemilla 门和查找。
- 区分运行和中的最后一个消息字和其前面的字。
- 标记 Sinsemilla 的开始。
- 标记 Sinsemilla 的结束。

我们使用常规选择器列来表示 Sinsemilla 门选择器 $q_{S1}$ 和 Sinsemilla 开始选择器 $q_{S4}.$ 其他两个选择器从单个固定列 $q_{S2}$ 合成如下：

$$
\begin{aligned}
q_{S3}  &= q_{S2} \cdot (q_{S2} - 1) \\
q_{run} &= q_{S2} - q_{S3} \\
\end{aligned}
$$

$$
\begin{array}{|c|c|c|}
\hline
q_{S2} & q_{S3} & q_{run} \\\hline
   0   &   0    &    0    \\\hline
   1   &   0    &    1    \\\hline
   2   &   2    &    0    \\\hline
\end{array}
$$

我们将 $q_{S2}$ 设置为 $1$ 在大多数 Sinsemilla 行上，并在每个元素的最后一步设置为 $0$，除了最后一个元素设置为 $2$。然后我们可以使用 $q_{S3}$ 在相邻行上的替换 $y_{A,i+1}$ 和 Sinsemilla 结束时的见证 $y_{A,n}$ 之间切换：

$$
\lambda_{2,i} \cdot (x_{A,i} - x_{A,i+1}) = y_{A,i} + \frac{2 - q_{S3}}{2} \cdot y_{A,i+1} + \frac{q_{S3}}{2} \cdot y_{A,n}
$$

### 生成器查找表
Sinsemilla 电路使用了 $2^{10}$ 个预计算的随机生成器。这些生成器被加载到查找表中：
$$
\begin{array}{|c|c|c|}
\hline
 table_{idx} & table_x         & table_y         \\\hline
 0           & x_{P[0]}        & y_{P[0]}        \\\hline
 1           & x_{P[1]}        & y_{P[1]}        \\\hline
 2           & x_{P[2]}        & y_{P[2]}        \\\hline
 \vdots      & \vdots          & \vdots          \\\hline
 2^{10} - 1  & x_{P[2^{10}-1]} & y_{P[2^{10}-1]} \\\hline
\end{array}
$$

### 布局
$$
\begin{array}{|c|c|c|c|c|c|c|c|c|c|}
\hline
\text{步骤} &    x_A     &    x_P      &   位    &    \lambda_1     &   \lambda_2      & q_{S1} & q_{S2} &    q_{S4}  & \textsf{fixed\_y\_Q}\\\hline
    0       & x_Q        & x_{P[m_1]}  & z_0      & \lambda_{1,0}    & \lambda_{2,0}    & 1      & 1      &     1      &    y_Q              \\\hline
    1       & x_{A,1}    & x_{P[m_2]}  & z_1      & \lambda_{1,1}    & \lambda_{2,1}    & 1      & 1      &     0      &     0               \\\hline
    2       & x_{A,2}    & x_{P[m_3]}  & z_2      & \lambda_{1,2}    & \lambda_{2,2}    & 1      & 1      &     0      &     0               \\\hline
  \vdots    & \vdots     & \vdots      & \vdots   & \vdots           & \vdots           & 1      & 1      &     0      &     0               \\\hline
   n-1      & x_{A,n-1}  & x_{P[m_n]}  & z_{n-1}  & \lambda_{1,n-1}  & \lambda_{2,n-1}  & 1      & 0      &     0      &     0               \\\hline
    0'      & x'_{A,0}   & x_{P[m'_1]} & z'_0     & \lambda'_{1,0}   & \lambda'_{2,0}   & 1      & 1      &     0      &     0               \\\hline
    1'      & x'_{A,1}   & x_{P[m'_2]} & z'_1     & \lambda'_{1,1}   & \lambda'_{2,1}   & 1      & 1      &     0      &     0               \\\hline
    2'      & x'_{A,2}   & x_{P[m'_3]} & z'_2     & \lambda'_{1,2}   & \lambda'_{2,2}   & 1      & 1      &     0      &     0               \\\hline
  \vdots    & \vdots     & \vdots      & \vdots   & \vdots           & \vdots           & 1      & 1      &     0      &     0               \\\hline
   n-1'     & x'_{A,n-1} & x_{P[m'_n]} & z'_{n-1} & \lambda'_{1,n-1} & \lambda'_{2,n-1} & 1      & 2      &     0      &     0               \\\hline
    n'      &  x'_{A,n}  &             &          &       y_{A,n}    &                  & 0      & 0      &     0      &     0               \\\hline
\end{array}
$$

$x_Q$, $z_0$, $z'_0$ 等通过等式约束进行复制。

### 优化的 Sinsemilla 门
$$
\begin{array}{lrcl}
\text{对于 } i \in [0, n), \text{ 令} &x_{R,i} &=& \lambda_{1,i}^2 - x_{A,i} - x_{P,i} \\
                                      &Y_{A,i} &=& (\lambda_{1,i} + \lambda_{2,i}) \cdot (x_{A,i} - x_{R,i}) \\
                                      &y_{P,i} &=& Y_{A,i}/2 - \lambda_{1,i} \cdot (x_{A,i} - x_{P,i}) \\
                                      &m_{i+1} &=& z_{i} - q_{run,i} \cdot z_{i+1} \cdot 2^k \\
                                      &q_{run} &=& q_{S2} - q_{S3} \\
                                      &q_{S3}  &=& q_{S2} \cdot (q_{S2} - 1)
\end{array}
$$

Halo 2 电路 API 可以自动替换 $y_{P,i}$、$x_{R,i}$、$Y_{A,i}$ 和 $Y_{A,i+1}$，因此我们无需手动进行替换。

- $x_{A,0} = x_Q$
- $2 \cdot y_Q = Y_{A,0}$
- 对于 $i$ 从 $0$ 到 $n-1$：
  - $(m_{i+1},\, x_{P,i},\, y_{P,i}) \in \mathcal{P}$
  - $\lambda_{2,i}^2 = x_{A,i+1} + x_{R,i} + x_{A,i}$
  - $4 \cdot \lambda_{2,i} \cdot (x_{A,i} - x_{A,i+1}) = 2 \cdot Y_{A,i} + (2 - q_{S3,i}) \cdot Y_{A,i+1} + 2 q_{S3,i} \cdot y_{A,n}$

注意，最后一个约束的每一项都相对于之前给出的约束程序乘以了 $4$。这是一个小的优化，避免了除以 $2$ 的操作。

通过在查找表达式中使用 $q_{S1}$ 进行门控，我们避免了需要填充未使用的单元格以通过查找参数的情况。优化后的查找值（使用默认索引 $0$）为：

$$
\begin{array}{ll}
(&q_{S1} \cdot m_{i+1}, \\
 &q_{S1} \cdot x_{P,i} + (1 - q_{S1}) \cdot x_{P,0}, \\
 &q_{S1} \cdot y_{P,i} + (1 - q_{S1}) \cdot y_{P,0} \;\;\;)
\end{array}
$$

这将查找参数的度数提高到 $6$。

$$
\begin{array}{|c|l|}
\hline
\text{度数} & \text{约束} \\\hline
4   & q_{S4} \cdot (2 \cdot y_Q - Y_{A,0}) = 0 \\\hline
6   & q_{S1,i} \Rightarrow (m_{i+1},\, x_{P,i},\, y_{P,i}) \in \mathcal{P} \\\hline
3   & q_{S1,i} \cdot \big(\lambda_{2,i}^2 - (x_{A,i+1} + x_{R,i} + x_{A,i})\big) \\\hline
5   & q_{S1,i} \cdot \left(4 \cdot \lambda_{2,i} \cdot (x_{A,i} - x_{A,i+1}) - (2 \cdot Y_{A,i} + (2 - q_{S3,i}) \cdot Y_{A,i+1} + 2 \cdot q_{S3,i} \cdot y_{A,n})\right) = 0 \\\hline
\end{array}
$$