# MerkleCRH

## Message decomposition
$\mathsf{SinsemillaHash}$ is used in the [$\mathsf{MerkleCRH^{Orchard}}$ hash function](https://zips.z.cash/protocol/protocol.pdf#orchardmerklecrh). The input to $\mathsf{SinsemillaHash}$ is:

$${l\star} \,||\, {\textsf{left}\star} \,||\, {\textsf{right}\star},$$

where:
- ${l\star} = \textsf{I2LEBSP}_{10}(l) = \textsf{I2LEBSP}_{10}(\textsf{MerkleDepth}^\textsf{Orchard} - 1 - \textsf{layer})$,
- ${\textsf{left}\star} = \textsf{I2LEBSP}_{\ell_{\textsf{Merkle}}^{\textsf{Orchard}}}(\textsf{left})$,
- ${\textsf{right}\star} = \textsf{I2LEBSP}_{\ell_{\textsf{Merkle}}^{\textsf{Orchard}}}(\textsf{right})$,

with $\ell_{\textsf{Merkle}}^{\textsf{Orchard}} = 255.$ $\textsf{left}\star$ and
$\textsf{right}\star$ are allowed to be non-canonical $255$-bit encodings of
$\textsf{left}$ and $\textsf{right}$.

Sinsemilla operates on multiples of 10 bits, so we start by decomposing the message into
chunks:

$$
\begin{aligned}
l\star              &= a_0 \\
\textsf{left}\star  &= a_1 \bconcat b_0 \bconcat b_1 \\
  &= (\text{bits } 0..=239 \text{ of } \textsf{ left }) \bconcat
     (\text{bits } 240..=249 \text{ of } \textsf{left}) \bconcat
     (\text{bits } 250..=254 \text{ of } \textsf{left}) \\
\textsf{right}\star &= b_2 \bconcat c \\
  &= (\text{bits } 0..=4 \text{ of } \textsf{right}) \bconcat
     (\text{bits } 5..=254 \text{ of } \textsf{right})
\end{aligned}
$$

Then we recompose the chunks into `MessagePiece`s:

$$
\begin{array}{|c|l|}
\hline
\text{Length (bits)} & \text{Piece} \\\hline
250 & a = a_0 \bconcat a_1 \\
20  & b = b_0 \bconcat b_1 \bconcat b_2 \\
250 & c \\\hline
\end{array}
$$

Each message piece is constrained by $\SinsemillaHash$ to its stated length. Additionally,
$\textsf{left}$ and $\textsf{right}$ are witnessed as field elements, so we know that they
are canonical. However, we need additional constraints to enforce that the chunks are the
correct bit lengths (or else they could overlap in the decompositions and allow the prover
to witness an arbitrary $\SinsemillaHash$ message).

Some of these constraints can be implemented with reusable circuit gadgets. We define a
custom gate controlled by the selector $q_\mathsf{decompose}$ to hold the remaining
constraints.

## Bit length constraints

Chunk $c$ is directly constrained by Sinsemilla. We constrain the remaining chunks with
the following constraints:

### $a_0, a_1$

$z_{1,a}$, the index-1 running sum output of $\textsf{SinsemillaHash}(a)$, is copied into
the gate. $z_{1,a}$ has been constrained by $\textsf{SinsemillaHash}$ to be $240$ bits,
and is precisely $a_1$. We recover chunk $a_0$ using $a, z_{1,a}:$
$$
\begin{aligned}
z_{1,a} &= \frac{a - a_0}{2^{10}}\\
        &= a_1 \\
        \implies a_0 &= a - z_{1,a} \cdot 2^{10}.
\end{aligned}
$$

### $b_0, b_1, b_2$

$z_{1,b}$, the index-1 running sum output of $\textsf{SinsemillaHash}(b)$, is copied into
the gate. $z_{1,b}$ has been constrained by $\textsf{SinsemillaHash}$ to be $10$ bits. We
witness the subpieces $b_1, b_2$ outside this gate, and constrain them each to be $5$
bits. Inside the gate, we check that $$b_1 + 2^5 \cdot b_2 = z_{1,b}.$$
We also recover the subpiece $b_0$ using $(b, z_{1,b})$:
$$
\begin{aligned}
z_{1,b} &= \frac{b - b_{0..=10}}{2^{10}}\\
        \implies b_0 &= b - (z_{1,b} \cdot 2^{10}).
\end{aligned}
$$

### Constraints

$$
\begin{array}{|c|l|}
\hline
\text{Degree} & \text{Constraint} \\\hline
  & \ShortLookupRangeCheck{b_1, 5} \\\hline
  & \ShortLookupRangeCheck{b_2, 5} \\\hline
2 & q_\mathsf{decompose} \cdot (z_{1,b} - (b_1 + b_2 \cdot 2^5)) = 0 \\\hline
\end{array}
$$

where $\ShortLookupRangeCheck{}$ is a
[short lookup range check](../decomposition.md#short-range-check).

## Decomposition constraints

We have now derived or witnessed every subpiece, and range-constrained every subpiece:
- $a_0$ ($10$ bits), derived as $a_0 = a - 2^{10} \cdot z_{1,a}$;
- $a_1$ ($240$ bits), equal to $z_{1,a}$;
- $b_0$ ($10$ bits), derived as $b_0 = b - 2^{10} \cdot z_{1,b}$;
- $b_1$ ($5$ bits) is witnessed and constrained outside the gate;
- $b_2$ ($5$ bits) is witnessed and constrained outside the gate;
- $c$ ($250$ bits) is witnessed and constrained outside the gate.
- $b_1 + 2^5 \cdot b_2$ is constrained to equal $z_{1, b}$.

We can now use them to reconstruct the original field element inputs:

$$
\begin{align}
l &= a_0 \\
\mathsf{left} &= a_1 + 2^{240} \cdot b_0 + 2^{250} \cdot b_1 \\
\mathsf{right} &= b_2 + 2^5 \cdot c
\end{align}
$$

$$
\begin{array}{|c|l|}
\hline
\text{Degree} & \text{Constraint} \\\hline
2 & q_\mathsf{decompose} \cdot (a_0 - l) = 0 \\\hline
2 & q_\mathsf{decompose} \cdot (a_1 + (b_0 + b_1 \cdot 2^{10}) \cdot 2^{240} - \mathsf{left}) = 0 \\\hline
2 & q_\mathsf{decompose} \cdot (b_2 + c \cdot 2^5 - \mathsf{right}) = 0 \\\hline
\end{array}
$$

## Region layout

$$
\begin{array}{|c|c|c|c|c|c}
        &        &     &               &                & q_\mathsf{decompose} \\\hline
   a    &   b    &  c  & \mathsf{left} & \mathsf{right} & 1 \\\hline
z_{1,a} &z_{1,b} & b_1 &      b_2      &       l        & 0 \\\hline
\end{array}
$$

## Circuit components
The Orchard circuit spans $10$ advice columns while the $\textsf{Sinsemilla}$ chip only uses $5$ advice columns. We distribute the path hashing evenly across two $\textsf{Sinsemilla}$ chips to make better use of the available circuit area. Since the output from the previous layer hash is copied into the next layer hash, we maintain continuity even when moving from one chip to the other.


# MerkleCRH

## 消息分解

$\mathsf{SinsemillaHash}$ 用于 [$\mathsf{MerkleCRH^{Orchard}}$ 哈希函数](https://zips.z.cash/protocol/protocol.pdf#orchardmerklecrh)。$\mathsf{SinsemillaHash}$ 的输入为：

$${l\star} \,||\, {\textsf{left}\star} \,||\, {\textsf{right}\star},$$

其中：
- ${l\star} = \textsf{I2LEBSP}_{10}(l) = \textsf{I2LEBSP}_{10}(\textsf{MerkleDepth}^\textsf{Orchard} - 1 - \textsf{layer})$,
- ${\textsf{left}\star} = \textsf{I2LEBSP}_{\ell_{\textsf{Merkle}}^{\textsf{Orchard}}}(\textsf{left})$,
- ${\textsf{right}\star} = \textsf{I2LEBSP}_{\ell_{\textsf{Merkle}}^{\textsf{Orchard}}}(\textsf{right})$,

其中 $\ell_{\textsf{Merkle}}^{\textsf{Orchard}} = 255$。$\textsf{left}\star$ 和
$\textsf{right}\star$ 允许是 $\textsf{left}$ 和 $\textsf{right}$ 的非规范 $255$ 位编码。

Sinsemilla 以 10 位的倍数运行，因此我们首先将消息分解为块：

$$
\begin{aligned}
l\star              &= a_0 \\
\textsf{left}\star  &= a_1 \bconcat b_0 \bconcat b_1 \\
  &= (\text{位 } 0..=239 \text{ 的 } \textsf{left}) \bconcat
     (\text{位 } 240..=249 \text{ 的 } \textsf{left}) \bconcat
     (\text{位 } 250..=254 \text{ 的 } \textsf{left}) \\
\textsf{right}\star &= b_2 \bconcat c \\
  &= (\text{位 } 0..=4 \text{ 的 } \textsf{right}) \bconcat
     (\text{位 } 5..=254 \text{ 的 } \textsf{right})
\end{aligned}
$$

然后我们将这些块重新组合为 `MessagePiece`：

$$
\begin{array}{|c|l|}
\hline
\text{长度（位）} & \text{块} \\\hline
250 & a = a_0 \bconcat a_1 \\
20  & b = b_0 \bconcat b_1 \bconcat b_2 \\
250 & c \\\hline
\end{array}
$$

每个消息块由 $\SinsemillaHash$ 约束为指定的长度。此外，$\textsf{left}$ 和 $\textsf{right}$ 作为域元素被见证，因此我们知道它们是规范的。然而，我们需要额外的约束来确保这些块是正确的位长度（否则它们可能在分解中重叠，并允许证明者见证任意的 $\SinsemillaHash$ 消息）。

其中一些约束可以通过可重用的电路 gadget 实现。我们定义了一个由选择器 $q_\mathsf{decompose}$ 控制的自定义门来保存剩余的约束。

## 位长度约束

块 $c$ 直接由 Sinsemilla 约束。我们通过以下约束来约束剩余的块：

### $a_0, a_1$

$z_{1,a}$，即 $\textsf{SinsemillaHash}(a)$ 的索引 1 运行和输出，被复制到门中。$z_{1,a}$ 已被 $\textsf{SinsemillaHash}$ 约束为 $240$ 位，并且恰好是 $a_1$。我们使用 $a, z_{1,a}$ 恢复块 $a_0$：
$$
\begin{aligned}
z_{1,a} &= \frac{a - a_0}{2^{10}}\\
        &= a_1 \\
        \implies a_0 &= a - z_{1,a} \cdot 2^{10}.
\end{aligned}
$$

### $b_0, b_1, b_2$

$z_{1,b}$，即 $\textsf{SinsemillaHash}(b)$ 的索引 1 运行和输出，被复制到门中。$z_{1,b}$ 已被 $\textsf{SinsemillaHash}$ 约束为 $10$ 位。我们在门外部见证子块 $b_1, b_2$，并将它们分别约束为 $5$ 位。在门内部，我们检查 $$b_1 + 2^5 \cdot b_2 = z_{1,b}.$$
我们还使用 $(b, z_{1,b})$ 恢复子块 $b_0$：
$$
\begin{aligned}
z_{1,b} &= \frac{b - b_{0..=10}}{2^{10}}\\
        \implies b_0 &= b - (z_{1,b} \cdot 2^{10}).
\end{aligned}
$$

### 约束

$$
\begin{array}{|c|l|}
\hline
\text{度数} & \text{约束} \\\hline
  & \ShortLookupRangeCheck{b_1, 5} \\\hline
  & \ShortLookupRangeCheck{b_2, 5} \\\hline
2 & q_\mathsf{decompose} \cdot (z_{1,b} - (b_1 + b_2 \cdot 2^5)) = 0 \\\hline
\end{array}
$$

其中 $\ShortLookupRangeCheck{}$ 是一个
[短查找范围检查](../decomposition.md#short-range-check)。

## 分解约束

我们现在已经推导或见证了所有子块，并对每个子块进行了范围约束：
- $a_0$（$10$ 位），推导为 $a_0 = a - 2^{10} \cdot z_{1,a}$；
- $a_1$（$240$ 位），等于 $z_{1,a}$；
- $b_0$（$10$ 位），推导为 $b_0 = b - 2^{10} \cdot z_{1,b}$；
- $b_1$（$5$ 位）在门外部见证并约束；
- $b_2$（$5$ 位）在门外部见证并约束；
- $c$（$250$ 位）在门外部见证并约束。
- $b_1 + 2^5 \cdot b_2$ 被约束为等于 $z_{1, b}$。

我们现在可以使用它们来重建原始的域元素输入：

$$
\begin{align}
l &= a_0 \\
\mathsf{left} &= a_1 + 2^{240} \cdot b_0 + 2^{250} \cdot b_1 \\
\mathsf{right} &= b_2 + 2^5 \cdot c
\end{align}
$$

$$
\begin{array}{|c|l|}
\hline
\text{度数} & \text{约束} \\\hline
2 & q_\mathsf{decompose} \cdot (a_0 - l) = 0 \\\hline
2 & q_\mathsf{decompose} \cdot (a_1 + (b_0 + b_1 \cdot 2^{10}) \cdot 2^{240} - \mathsf{left}) = 0 \\\hline
2 & q_\mathsf{decompose} \cdot (b_2 + c \cdot 2^5 - \mathsf{right}) = 0 \\\hline
\end{array}
$$

## 区域布局

$$
\begin{array}{|c|c|c|c|c|c}
        &        &     &               &                & q_\mathsf{decompose} \\\hline
   a    &   b    &  c  & \mathsf{left} & \mathsf{right} & 1 \\\hline
z_{1,a} &z_{1,b} & b_1 &      b_2      &       l        & 0 \\\hline
\end{array}
$$

## 电路组件
Orchard 电路跨越 $10$ 个建议列，而 $\textsf{Sinsemilla}$ 芯片仅使用 $5$ 个建议列。我们将路径哈希均匀分布在两个 $\textsf{Sinsemilla}$ 芯片上，以更好地利用可用的电路区域。由于前一层的哈希输出被复制到下一层的哈希中，即使在从一个芯片移动到另一个芯片时，我们也保持了连续性。