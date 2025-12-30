# Fixed-base scalar multiplication

There are $6$ fixed bases in the Orchard protocol:
- $\mathcal{K}^{\mathsf{Orchard}}$, used in deriving the nullifier;
- $\mathcal{G}^{\mathsf{Orchard}}$, used in spend authorization;
- $\mathcal{R}$ base for $\mathsf{NoteCommit}^{\mathsf{Orchard}}$;
- $\mathcal{V}$ and $\mathcal{R}$ bases for $\mathsf{ValueCommit}^{\mathsf{Orchard}}$; and
- $\mathcal{R}$ base for $\mathsf{Commit}^{\mathsf{ivk}}$.
## Decompose scalar
We support fixed-base scalar multiplication with three types of scalars:
### Full-width scalar
A $255$-bit scalar from $\mathbb{F}_q$. We decompose a full-width scalar $\alpha$ into $85$ $3$-bit windows:

$$\alpha = k_0 + k_1 \cdot (2^3)^1 + \cdots + k_{84} \cdot (2^3)^{84}, k_i \in [0..2^3).$$

The scalar multiplication will be computed correctly for $k_{0..84}$ representing any
integer in the range $[0, 2^{255})$ - that is, the scalar is allowed to be non-canonical.

We range-constrain each $3$-bit word of the scalar decomposition using a polynomial range-check constraint:
$$
\begin{array}{|c|l|}
\hline
\text{Degree} & \text{Constraint} \\\hline
9 & q_\texttt{mul\_fixed\_full} \cdot \RangeCheck{\text{word}}{2^3} = 0 \\\hline
\end{array}
$$
where $\RangeCheck{\text{word}}{\texttt{range}} = \text{word} \cdot (1 - \text{word}) \cdots (\texttt{range} - 1 - \text{word}).$

### Base field element
We support using a base field element as the scalar in fixed-base multiplication. This occurs, for example, in the scalar multiplication for the nullifier computation of the Action circuit $\mathsf{DeriveNullifier_{nk}} = \mathsf{Extract}_\mathbb{P}\left(\left[(\mathsf{PRF_{nk}^{nfOrchard}}(\rho) + \psi) \bmod{q_\mathbb{P}}\right]\mathcal{K}^\mathsf{Orchard} + \mathsf{cm}\right)$: here, the scalar $$\left[(\mathsf{PRF_{nk}^{nfOrchard}}(\rho) + \psi) \bmod{q_\mathbb{P}}\right]$$ is the result of a base field addition.

Decompose the base field element $\alpha$ into three-bit windows, and range-constrain each window, using the [short range decomposition](../decomposition.md#short-range-decomposition) gadget in strict mode, with $W = 85, K = 3.$

If $k_{0..84}$ is witnessed directly then no issue of canonicity arises. However, because the scalar is given as a base field element here, care must be taken to ensure a canonical representation, since $2^{255} > p$. That is, we must check that $0 \leq \alpha < p,$ where $p$ the is Pallas base field modulus $$p = 2^{254} + t_p = 2^{254} + 45560315531419706090280762371685220353.$$ Note that $t_p < 2^{130}.$

To do this, we decompose $\alpha$ into three pieces: $$\alpha = \alpha_0 \text{ (252 bits) } \,||\, \alpha_1 \text{ (2 bits) } \,||\, \alpha_2 \text{ (1 bit) }.$$

We check the correctness of this decomposition by:
$$
\begin{array}{|c|l|}
\hline
\text{Degree} & \text{Constraint} \\\hline
5 & q_\text{canon-base-field} \cdot \RangeCheck{\alpha_1}{2^2} = 0 \\\hline
3 & q_\text{canon-base-field} \cdot \BoolCheck{\alpha_2} = 0 \\\hline
2 & q_\text{canon-base-field} \cdot \left(z_{84} - (\alpha_1 + \alpha_2 \cdot 2^2)\right) = 0 \\\hline
\end{array}
$$
If the MSB $\alpha_2 = 0$ is not set, then $\alpha < 2^{254} < p.$ However, in the case where $\alpha_2 = 1$, we must check:
- $\alpha_2 = 1 \implies \alpha_1 = 0;$
- $\alpha_2 = 1 \implies \alpha_0 < t_p$:
  - $\alpha_2 = 1 \implies 0 \leq \alpha_0 < 2^{130}$,
  - $\alpha_2 = 1 \implies 0 \leq \alpha_0 + 2^{130} - t_p < 2^{130}$

To check that $0 \leq \alpha_0 < 2^{130},$ we make use of the three-bit running sum decomposition:
- Firstly, we constrain $\alpha_0$ to be a $132$-bit value by enforcing its high $120$ bits to be all-zero. We can get $\textsf{alpha\_0\_hi\_120}$ from the decomposition:
$$
\begin{aligned}
z_{44} &= k_{44} + 2^3 k_{45} + \cdots + 2^{3 \cdot (84 - 44)} k_{84}\\
\implies \textsf{alpha\_0\_hi\_120} &= z_{44} - 2^{3 \cdot (84 - 44)} k_{84}\\
&= z_{44} - 2^{3 \cdot (40)} z_{84}.
\end{aligned}
$$
- Then, we constrain bits $130..\!\!=\!\!131$ of $\alpha_0$ to be zeroes; in other words, we constrain the three-bit word $k_{43} = \alpha[129..\!\!=\!\!131] = \alpha_0[129..\!\!=\!\!131] \in \{0, 1\}.$ We make use of the running sum decomposition to obtain $k_{43} = z_{43} - z_{44} \cdot 2^3.$

Define $\alpha'_0 = \alpha_0 + 2^{130} - t_p$. To check that $0 \leq \alpha'_0 < 2^{130},$ we use 13 ten-bit [lookups](../decomposition.md#lookup-decomposition), where we constrain the $z_{13}$ running sum output of the lookup to be $0$ if $\alpha_2 = 1.$
$$
\begin{array}{|c|l|l|}
\hline
\text{Degree} & \text{Constraint} & \text{Comment} \\\hline
2 & q_\text{canon-base-field} \cdot (\alpha_0' - (\alpha_0 + 2^{130} - t_\mathbb{P})) = 0 \\\hline
3 & q_\text{canon-base-field} \cdot \alpha_2 \cdot \alpha_1 = 0 & \alpha_2 = 1 \implies \alpha_1 = 0 \\\hline
3 & q_\text{canon-base-field} \cdot \alpha_2 \cdot \textsf{alpha\_0\_hi\_120} = 0 & \text{Constrain $\alpha_0$ to be a $132$-bit value} \\\hline
4 & q_\text{canon-base-field} \cdot \alpha_2 \cdot \BoolCheck{k_{43}} = 0 & \text{Constrain $\alpha_0[130..\!\!=\!\!131]$ to $0$}  \\\hline
3 & q_\text{canon-base-field} \cdot \alpha_2 \cdot z_{13}(\texttt{lookup}(\alpha_0', 13)) = 0 & \alpha_2 = 1 \implies 0 \leq \alpha'_0 < 2^{130}\\\hline
\end{array}
$$
### Short signed scalar
A short signed scalar is witnessed as a magnitude $m$ and sign $s$ such that
$$
s \in \{-1, 1\} \\
m \in [0, 2^{64}) \\
\mathsf{v^{old}} - \mathsf{v^{new}} = s \cdot m.
$$

This is used for $\mathsf{ValueCommit^{Orchard}}$. We want to compute $\mathsf{ValueCommit^{Orchard}_{rcv}}(\mathsf{v^{old}} - \mathsf{v^{new}}) = [\mathsf{v^{old}} - \mathsf{v^{new}}] \mathcal{V} + [\mathsf{rcv}] \mathcal{R}$, where
$$
-(2^{64}-1) \leq \mathsf{v^{old}} - \mathsf{v^{new}} \leq 2^{64}-1
$$

$\mathsf{v^{old}}$ and $\mathsf{v^{new}}$ are each already constrained to $64$ bits (by their use as inputs to $\mathsf{NoteCommit^{Orchard}}$).

Decompose the magnitude $m$ into three-bit windows, and range-constrain each window, using the [short range decomposition](../decomposition.md#short-range-decomposition) gadget in strict mode, with $W = 22, K = 3.$

<a name="constrain-short-signed-msb"></a> We have two additional constraints:
$$
\begin{array}{|c|l|l|}
\hline
\text{Degree} & \text{Constraint} & \text{Comment} \\\hline
3 & q_\texttt{mul\_fixed\_short} \cdot \BoolCheck{k_{21}} = 0 & \text{The last window must be a single bit.}\\\hline
3 & q_\texttt{mul\_fixed\_short} \cdot \left(s^2 - 1\right) = 0  &\text{The sign must be $1$ or $-1$.}\\\hline
\end{array}
$$
where $\BoolCheck{x} = x \cdot (1 - x)$.

## Load fixed base
Then, we precompute multiples of the fixed base $B$ for each window. This takes the form of a window table: $M[0..W)[0..8)$ such that:

- for the first (W-1) rows $M[0..(W-1))[0..8)$: $$M[w][k] = [(k+2) \cdot (2^3)^w]B$$
- in the last row $M[W-1][0..8)$: $$M[w][k] = [k \cdot (2^3)^w - \sum\limits_{j=0}^{83} 2^{3j+1}]B$$

The additional $(k + 2)$ term lets us avoid adding the point at infinity in the case $k = 0$. We offset these accumulated terms by subtracting them in the final window, i.e. we subtract $\sum\limits_{j=0}^{W-2} 2^{3j+1}$.

> Note: Although an offset of $(k + 1)$ would naively suffice, it introduces an edge case when $k_0 = 7, k_1= 0$.
> In this case, the window table entries evaluate to the same point:
> * $M[0][k_0] = [(7+1)*(2^3)^0]B = [8]B,$
> * $M[1][k_1] = [(0+1)*(2^3)^1]B = [8]B.$
>
> In fixed-base scalar multiplication, we sum the multiples of $B$ at each window (except the last) using incomplete addition.
> Since the point doubling case is not handled by incomplete addition, we avoid it by using an offset of $(k+2).$

For each window of fixed-base multiples $M[w] = (M[w][0], \cdots, M[w][7]), w \in [0..(W-1))$:
- Define a Lagrange interpolation polynomial $\mathcal{L}_x(k)$ that maps $k \in [0..8)$ to the $x$-coordinate of the multiple $M[w][k]$, i.e.
  $$
  \mathcal{L}_x(k) = \begin{cases}
    ([(k + 2) \cdot (2^3)^w] B)_x &\text{for } w \in [0..(W-1)); \\
    ([k \cdot (2^3)^w - \sum\limits_{j=0}^{83} 2^{3j+1}] B)_x &\text{for } w = 84; \text{ and}
  \end{cases}
  $$
- Find a value $z_w$ such that $z_w + (M[w][k])_y$ is a square $u^2$ in the field, but the wrong-sign $y$-coordinate $z_w - (M[w][k])_y$ does not produce a square.

Repeating this for all $W$ windows, we end up with:
- an $W \times 8$ table $\mathcal{L}_x$ storing $8$ coefficients interpolating the $x-$coordinate for each window. Each $x$-coordinate interpolation polynomial will be of the form
$$\mathcal{L}_x[w](k) = c_0 + c_1 \cdot k + c_2 \cdot k^2 + \cdots + c_7 \cdot k^7,$$
where $k \in [0..8), w \in [0..85)$ and $c_k$'s are the coefficients for each power of $k$; and
- a length-$W$ array $Z$ of $z_w$'s.

We load these precomputed values into fixed columns whenever we do fixed-base scalar multiplication in the circuit.

## Fixed-base scalar multiplication
Given a decomposed scalar $\alpha$ and a fixed base $B$, we compute $[\alpha]B$ as follows:

1. For each $k_w, w \in [0..85), k_w \in [0..8)$ in the scalar decomposition, witness the $x$- and $y$-coordinates $(x_w,y_w) = M[w][k_w].$
2. Check that $(x_w, y_w)$ is on the curve: $y_w^2 = x_w^3 + b$.
3. Witness $u_w$ such that $y_w + z_w = u_w^2$.
4. For all windows but the last, use [incomplete addition](./incomplete-add.md) to sum the $M[w][k_w]$'s, resulting in $[\alpha - k_{84} \cdot (2^3)^{84} + \sum\limits_{j=0}^{83} 2^{3j+1}]B$.
5. For the last window, use complete addition $M[83][k_{83}] + M[84][k_{84}]$ and return the final result.

> Note: complete addition is required in the final step to correctly map $[0]B$ to a representation of the point at infinity, $(0,0)$; and also to handle a corner case for which the last step is a doubling.

<a name="constrain-coordinates"></a> Constraints:
$$
\begin{array}{|c|l|}
\hline
\text{Degree} & \text{Constraint} \\\hline
8 & q_\text{mul-fixed} \cdot \left( \mathcal{L}_x[w](k_w) - x_w \right) = 0 \\\hline
4 & q_\text{mul-fixed} \cdot \left( y_w^2 - x_w^3 - b \right) = 0 \\\hline
3 & q_\text{mul-fixed} \cdot \left( u_w^2 - y_w - Z[w] \right) = 0 \\\hline
\end{array}
$$

where $b = 5$ (from the Pallas curve equation).

### Signed short exponent
Recall that the signed short exponent is witnessed as a $64-$bit magnitude $m$, and a sign $s \in {1, -1}.$ Using the above algorithm, we compute $P = [m] \mathcal{B}$. Then, to get the final result $P',$ we conditionally negate $P$ using $(x, y) \mapsto (x, s \cdot y)$.

<a name="constrain-short-signed-conditional-neg"></a> Constraints:
$$
\begin{array}{|c|l|}
\hline
\text{Degree} & \text{Constraint} \\\hline
3 & q_\texttt{mul\_fixed\_short} \cdot \left(P'_y - P_y\right) \cdot \left(P'_y + P_y\right) = 0 \\\hline
3 & q_\texttt{mul\_fixed\_short} \cdot \left(s \cdot P'_y - P_y\right) = 0 \\\hline
\end{array}
$$

## Layout

$$
\begin{array}{|c|c|c|c|c|c|c|c|}
\hline
  x_P   &   y_P   &      x_{QR}       &        y_{QR}      &    u   & \text{window}   & L_{0..=7}   & \textsf{fixed\_z}   \\\hline
x_{P,0} & y_{P,0} &                   &                    &   u_0  & \text{window}_0 & L_{0..=7,0} & \textsf{fixed\_z}_0 \\\hline
x_{P,1} & y_{P,1} & x_{Q,1} = x_{P,0} & y_{Q,1} = y_{P,0}  &   u_1  & \text{window}_1 & L_{0..=7,1} & \textsf{fixed\_z}_1 \\\hline
x_{P,2} & y_{P,2} & x_{Q,2} = x_{R,1} & y_{Q,2} = y_{R,1}  &   u_2  & \text{window}_2 & L_{0..=7,1} & \textsf{fixed\_z}_2 \\\hline
\vdots  & \vdots  &      \vdots       &       \vdots       & \vdots &     \vdots      &    \vdots   &        \vdots       \\\hline
\end{array}
$$

Note: this doesn't include the last row that uses [complete addition](./addition.md#Complete-addition). In the implementation this is allocated in a different region.

# 固定基标量乘法

Orchard 协议中有 $6$ 个固定基：
- $\mathcal{K}^{\mathsf{Orchard}}$，用于推导无效符；
- $\mathcal{G}^{\mathsf{Orchard}}$，用于支出授权；
- $\mathsf{NoteCommit}^{\mathsf{Orchard}}$ 的 $\mathcal{R}$ 基；
- $\mathsf{ValueCommit}^{\mathsf{Orchard}}$ 的 $\mathcal{V}$ 和 $\mathcal{R}$ 基；以及
- $\mathsf{Commit}^{\mathsf{ivk}}$ 的 $\mathcal{R}$ 基。

## 标量分解

我们支持三种类型的标量进行固定基标量乘法：

### 全宽度标量

一个来自 $\mathbb{F}_q$ 的 $255$ 位标量。我们将全宽度标量 $\alpha$ 分解为 $85$ 个 $3$ 位窗口：

$$\alpha = k_0 + k_1 \cdot (2^3)^1 + \cdots + k_{84} \cdot (2^3)^{84}, k_i \in [0..2^3).$$

标量乘法将正确计算 $k_{0..84}$ 表示的任何在范围 $[0, 2^{255})$ 内的整数——即标量允许是非规范的。

我们使用多项式范围检查约束对每个 $3$ 位字进行范围约束：
$$
\begin{array}{|c|l|}
\hline
\text{度数} & \text{约束} \\\hline
9 & q_\texttt{mul\_fixed\_full} \cdot \RangeCheck{\text{字}}{2^3} = 0 \\\hline
\end{array}
$$
其中 $\RangeCheck{\text{字}}{\texttt{范围}} = \text{字} \cdot (1 - \text{字}) \cdots (\texttt{范围} - 1 - \text{字}).$

### 基域元素

我们支持使用基域元素作为固定基乘法中的标量。例如，在 Action 电路的无效符计算 $\mathsf{DeriveNullifier_{nk}} = \mathsf{Extract}_\mathbb{P}\left(\left[(\mathsf{PRF_{nk}^{nfOrchard}}(\rho) + \psi) \bmod{q_\mathbb{P}}\right]\mathcal{K}^\mathsf{Orchard} + \mathsf{cm}\right)$ 中，标量 $$\left[(\mathsf{PRF_{nk}^{nfOrchard}}(\rho) + \psi) \bmod{q_\mathbb{P}}\right]$$ 是基域加法的结果。

将基域元素 $\alpha$ 分解为 $3$ 位窗口，并对每个窗口进行范围约束，使用 [短范围分解](../decomposition.md#short-range-decomposition) gadget 的严格模式，其中 $W = 85, K = 3.$

如果 $k_{0..84}$ 直接见证，则不会出现规范性问题。然而，由于标量在此作为基域元素给出，必须注意确保规范表示，因为 $2^{255} > p$。也就是说，我们必须检查 $0 \leq \alpha < p,$ 其中 $p$ 是 Pallas 基域模数 $$p = 2^{254} + t_p = 2^{254} + 45560315531419706090280762371685220353.$$ 注意 $t_p < 2^{130}.$

为此，我们将 $\alpha$ 分解为三部分： $$\alpha = \alpha_0 \text{ (252 位) } \,||\, \alpha_1 \text{ (2 位) } \,||\, \alpha_2 \text{ (1 位) }.$$

我们通过以下方式检查此分解的正确性：
$$
\begin{array}{|c|l|}
\hline
\text{度数} & \text{约束} \\\hline
5 & q_\text{canon-base-field} \cdot \RangeCheck{\alpha_1}{2^2} = 0 \\\hline
3 & q_\text{canon-base-field} \cdot \BoolCheck{\alpha_2} = 0 \\\hline
2 & q_\text{canon-base-field} \cdot \left(z_{84} - (\alpha_1 + \alpha_2 \cdot 2^2)\right) = 0 \\\hline
\end{array}
$$
如果最高位 $\alpha_2 = 0$ 未设置，则 $\alpha < 2^{254} < p.$ 然而，在 $\alpha_2 = 1$ 的情况下，我们必须检查：
- $\alpha_2 = 1 \implies \alpha_1 = 0;$
- $\alpha_2 = 1 \implies \alpha_0 < t_p$：
  - $\alpha_2 = 1 \implies 0 \leq \alpha_0 < 2^{130}$,
  - $\alpha_2 = 1 \implies 0 \leq \alpha_0 + 2^{130} - t_p < 2^{130}$

为了检查 $0 \leq \alpha_0 < 2^{130},$ 我们使用三位的运行和分解：
- 首先，我们通过强制其高 $120$ 位为零来约束 $\alpha_0$ 为 $132$ 位值。我们可以从分解中获取 $\textsf{alpha\_0\_hi\_120}$：
$$
\begin{aligned}
z_{44} &= k_{44} + 2^3 k_{45} + \cdots + 2^{3 \cdot (84 - 44)} k_{84}\\
\implies \textsf{alpha\_0\_hi\_120} &= z_{44} - 2^{3 \cdot (84 - 44)} k_{84}\\
&= z_{44} - 2^{3 \cdot (40)} z_{84}.
\end{aligned}
$$
- 然后，我们约束 $\alpha_0$ 的第 $130..\!\!=\!\!131$ 位为零；换句话说，我们约束三位字 $k_{43} = \alpha[129..\!\!=\!\!131] = \alpha_0[129..\!\!=\!\!131] \in \{0, 1\}.$ 我们使用运行和分解来获取 $k_{43} = z_{43} - z_{44} \cdot 2^3.$

定义 $\alpha'_0 = \alpha_0 + 2^{130} - t_p$。为了检查 $0 \leq \alpha'_0 < 2^{130},$ 我们使用 $13$ 个十位 [查找](../decomposition.md#lookup-decomposition)，其中如果 $\alpha_2 = 1$，我们约束查找的 $z_{13}$ 运行和输出为 $0$。
$$
\begin{array}{|c|l|l|}
\hline
\text{度数} & \text{约束} & \text{注释} \\\hline
2 & q_\text{canon-base-field} \cdot (\alpha_0' - (\alpha_0 + 2^{130} - t_\mathbb{P})) = 0 \\\hline
3 & q_\text{canon-base-field} \cdot \alpha_2 \cdot \alpha_1 = 0 & \alpha_2 = 1 \implies \alpha_1 = 0 \\\hline
3 & q_\text{canon-base-field} \cdot \alpha_2 \cdot \textsf{alpha\_0\_hi\_120} = 0 & \text{约束 $\alpha_0$ 为 $132$ 位值} \\\hline
4 & q_\text{canon-base-field} \cdot \alpha_2 \cdot \BoolCheck{k_{43}} = 0 & \text{约束 $\alpha_0[130..\!\!=\!\!131]$ 为 $0$}  \\\hline
3 & q_\text{canon-base-field} \cdot \alpha_2 \cdot z_{13}(\texttt{lookup}(\alpha_0', 13)) = 0 & \alpha_2 = 1 \implies 0 \leq \alpha'_0 < 2^{130}\\\hline
\end{array}
$$

### 短有符号标量

短有符号标量见证为一个幅度 $m$ 和符号 $s$，使得
$$
s \in \{-1, 1\} \\
m \in [0, 2^{64}) \\
\mathsf{v^{old}} - \mathsf{v^{new}} = s \cdot m.
$$

这用于 $\mathsf{ValueCommit^{Orchard}}$。我们希望计算 $\mathsf{ValueCommit^{Orchard}_{rcv}}(\mathsf{v^{old}} - \mathsf{v^{new}}) = [\mathsf{v^{old}} - \mathsf{v^{new}}] \mathcal{V} + [\mathsf{rcv}] \mathcal{R}$，其中
$$
-(2^{64}-1) \leq \mathsf{v^{old}} - \mathsf{v^{new}} \leq 2^{64}-1
$$

$\mathsf{v^{old}}$ 和 $\mathsf{v^{new}}$ 各自已经被约束为 $64$ 位（通过它们作为 $\mathsf{NoteCommit^{Orchard}}$ 的输入）。

将幅度 $m$ 分解为三位窗口，并对每个窗口进行范围约束，使用 [短范围分解](../decomposition.md#short-range-decomposition) gadget 的严格模式，其中 $W = 22, K = 3.$

<a name="constrain-short-signed-msb"></a> 我们有两个额外的约束：
$$
\begin{array}{|c|l|l|}
\hline
\text{度数} & \text{约束} & \text{注释} \\\hline
3 & q_\texttt{mul\_fixed\_short} \cdot \BoolCheck{k_{21}} = 0 & \text{最后一个窗口必须是一个位。}\\\hline
3 & q_\texttt{mul\_fixed\_short} \cdot \left(s^2 - 1\right) = 0  &\text{符号必须是 $1$ 或 $-1$。}\\\hline
\end{array}
$$
其中 $\BoolCheck{x} = x \cdot (1 - x)$。

## 加载固定基

然后，我们为每个窗口预计算固定基 $B$ 的倍数。这采用窗口表的形式：$M[0..W)[0..8)$，使得：

- 对于前 $W-1$ 行 $M[0..(W-1))[0..8)$：$$M[w][k] = [(k+2) \cdot (2^3)^w]B$$
- 在最后一行 $M[W-1][0..8)$：$$M[w][k] = [k \cdot (2^3)^w - \sum\limits_{j=0}^{83} 2^{3j+1}]B$$

额外的 $(k + 2)$ 项让我们避免在 $k = 0$ 的情况下添加无穷远点。我们通过在最后一个窗口中减去这些累积项来抵消这些项，即我们减去 $\sum\limits_{j=0}^{W-2} 2^{3j+1}$。

> 注意：虽然 $(k + 1)$ 的偏移量在理论上足够，但它引入了当 $k_0 = 7, k_1= 0$ 时的边缘情况。
> 在这种情况下，窗口表条目评估为相同的点：
> * $M[0][k_0] = [(7+1)*(2^3)^0]B = [8]B,$
> * $M[1][k_1] = [(0+1)*(2^3)^1]B = [8]B.$
>
> 在固定基标量乘法中，我们使用不完全加法对每个窗口的 $B$ 的倍数求和（除了最后一个）。
> 由于不完全加法不处理点加倍的情况，我们通过使用 $(k+2)$ 的偏移量来避免它。

对于每个固定基倍数的窗口 $M[w] = (M[w][0], \cdots, M[w][7]), w \in [0..(W-1))$：
- 定义一个拉格朗日插值多项式 $\mathcal{L}_x(k)$，将 $k \in [0..8)$ 映射到倍数 $M[w][k]$ 的 $x$-坐标，即
  $$
  \mathcal{L}_x(k) = \begin{cases}
    ([(k + 2) \cdot (2^3)^w] B)_x &\text{对于 } w \in [0..(W-1)); \\
    ([k \cdot (2^3)^w - \sum\limits_{j=0}^{83} 2^{3j+1}] B)_x &\text{对于 } w = 84; \text{ 以及}
  \end{cases}
  $$
- 找到一个值 $z_w$，使得 $z_w + (M[w][k])_y$ 是域中的平方 $u^2$，但错误的符号 $y$-坐标 $z_w - (M[w][k])_y$ 不会产生平方。

对所有 $W$ 个窗口重复此操作，我们最终得到：
- 一个 $W \times 8$ 的表 $\mathcal{L}_x$，存储每个窗口的 $8$ 个插值 $x$-坐标的系数。每个 $x$-坐标插值多项式将具有以下形式
$$\mathcal{L}_x[w](k) = c_0 + c_1 \cdot k + c_2 \cdot k^2 + \cdots + c_7 \cdot k^7,$$
其中 $k \in [0..8), w \in [0..85)$ 且 $c_k$ 是每个 $k$ 幂的系数；以及
- 一个长度为 $W$ 的 $z_w$ 数组 $Z$。

我们在电路中执行固定基标量乘法时，将这些预计算值加载到固定列中。

## 固定基标量乘法

给定分解的标量 $\alpha$ 和固定基 $B$，我们按如下方式计算 $[\alpha]B$：

1. 对于标量分解中的每个 $k_w, w \in [0..85), k_w \in [0..8)$，见证 $x$-和 $y$-坐标 $(x_w,y_w) = M[w][k_w].$
2. 检查 $(x_w, y_w)$ 是否在曲线上：$y_w^2 = x_w^3 + b$。
3. 见证 $u_w$ 使得 $y_w + z_w = u_w^2$。
4. 对于所有窗口（除了最后一个），使用 [不完全加法](./incomplete-add.md) 对 $M[w][k_w]$ 求和，结果为 $[\alpha - k_{84} \cdot (2^3)^{84} + \sum\limits_{j=0}^{83} 2^{3j+1}]B$。
5. 对于最后一个窗口，使用完全加法 $M[83][k_{83}] + M[84][k_{84}]$ 并返回最终结果。

> 注意：最后一步需要完全加法，以正确地将 $[0]B$ 映射到无穷远点的表示 $(0,0)$；并且还要处理最后一步是加倍的角落情况。

<a name="constrain-coordinates"></a> 约束：
$$
\begin{array}{|c|l|}
\hline
\text{度数} & \text{约束} \\\hline
8 & q_\text{mul-fixed} \cdot \left( \mathcal{L}_x[w](k_w) - x_w \right) = 0 \\\hline
4 & q_\text{mul-fixed} \cdot \left( y_w^2 - x_w^3 - b \right) = 0 \\\hline
3 & q_\text{mul-fixed} \cdot \left( u_w^2 - y_w - Z[w] \right) = 0 \\\hline
\end{array}
$$
其中 $b = 5$（来自 Pallas 曲线方程）。

### 有符号短指数

回想一下，有符号短指数见证为一个 $64$ 位幅度 $m$ 和一个符号 $s \in {1, -1}.$ 使用上述算法，我们计算 $P = [m] \mathcal{B}$。然后，为了获得最终结果 $P'$，我们使用 $(x, y) \mapsto (x, s \cdot y)$ 有条件地取反 $P$。

<a name="constrain-short-signed-conditional-neg"></a> 约束：
$$
\begin{array}{|c|l|}
\hline
\text{度数} & \text{约束} \\\hline
3 & q_\texttt{mul\_fixed\_short} \cdot \left(P'_y - P_y\right) \cdot \left(P'_y + P_y\right) = 0 \\\hline
3 & q_\texttt{mul\_fixed\_short} \cdot \left(s \cdot P'_y - P_y\right) = 0 \\\hline
\end{array}
$$

## 布局

$$
\begin{array}{|c|c|c|c|c|c|c|c|}
\hline
  x_P   &   y_P   &      x_{QR}       &        y_{QR}      &    u   & \text{窗口}   & L_{0..=7}   & \textsf{fixed\_z}   \\\hline
x_{P,0} & y_{P,0} &                   &                    &   u_0  & \text{窗口}_0 & L_{0..=7,0} & \textsf{fixed\_z}_0 \\\hline
x_{P,1} & y_{P,1} & x_{Q,1} = x_{P,0} & y_{Q,1} = y_{P,0}  &   u_1  & \text{窗口}_1 & L_{0..=7,1} & \textsf{fixed\_z}_1 \\\hline
x_{P,2} & y_{P,2} & x_{Q,2} = x_{R,1} & y_{Q,2} = y_{R,1}  &   u_2  & \text{窗口}_2 & L_{0..=7,1} & \textsf{fixed\_z}_2 \\\hline
\vdots  & \vdots  &      \vdots       &       \vdots       & \vdots &     \vdots      &    \vdots   &        \vdots       \\\hline
\end{array}
$$

注意：这不包括使用 [完全加法](./addition.md#Complete-addition) 的最后一行。在实现中，这分配在不同的区域。