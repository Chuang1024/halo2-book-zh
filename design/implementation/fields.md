# Fields

The [Pasta curves](https://electriccoin.co/blog/the-pasta-curves-for-halo-2-and-beyond/)
that we use in `halo2` are designed to be highly 2-adic, meaning that a large $2^S$
[multiplicative subgroup](../../background/fields.md#multiplicative-subgroups) exists in
each field. That is, we can write $p - 1 \equiv 2^S \cdot T$ with $T$ odd. For both Pallas
and Vesta, $S = 32$; this helps to simplify the field implementations.

## Sarkar square-root algorithm (table-based variant)

We use a technique from [Sarkar2020](https://eprint.iacr.org/2020/1407.pdf) to compute
[square roots](../../background/fields.md#square-roots) in `halo2`. The intuition behind
the algorithm is that we can split the task into computing square roots in each
multiplicative subgroup.

Suppose we want to find the square root of $u$ modulo one of the Pasta primes $p$, where
$u$ is a non-zero square in $\mathbb{Z}_p^\times$. We define a $2^S$
[root of unity](../../background/fields.md#roots-of-unity) $g = z^T$ where $z$ is a
non-square in $\mathbb{Z}_p^\times$, and precompute the following tables:

$$
gtab = \begin{bmatrix}
g^0 & g^1 & ... & g^{2^8 - 1} \\
(g^{2^8})^0 & (g^{2^8})^1 & ... & (g^{2^8})^{2^8 - 1} \\
(g^{2^{16}})^0 & (g^{2^{16}})^1 & ... & (g^{2^{16}})^{2^8 - 1} \\
(g^{2^{24}})^0 & (g^{2^{24}})^1 & ... & (g^{2^{24}})^{2^8 - 1}
\end{bmatrix}
$$

$$
invtab = \begin{bmatrix}
(g^{-2^{24}})^0 & (g^{-2^{24}})^1 & ... & (g^{-2^{24}})^{2^8 - 1}
\end{bmatrix}
$$

Let $v = u^{(T-1)/2}$. We can then define $x = uv \cdot v = u^T$ as an element of the
$2^S$ multiplicative subgroup.

Let $x_3 = x, x_2 = x_3^{2^8}, x_1 = x_2^{2^8}, x_0 = x_1^{2^8}.$

### i = 0, 1
Using $invtab$, we lookup $t_0$ such that
$$
x_0 = (g^{-2^{24}})^{t_0} \implies x_0 \cdot g^{t_0 \cdot 2^{24}} = 1.
$$

Define $\alpha_1 = x_1 \cdot (g^{2^{16}})^{t_0}.$

### i = 2
Lookup $t_1$ s.t. 
$$
\begin{array}{ll}
\alpha_1 = (g^{-2^{24}})^{t_1} &\implies x_1 \cdot (g^{2^{16}})^{t_0} = (g^{-2^{24}})^{t_1} \\
&\implies
x_1 \cdot g^{(t_0 + 2^8 \cdot t_1) \cdot 2^{16}} = 1.
\end{array}
$$

Define $\alpha_2 = x_2 \cdot (g^{2^8})^{t_0 + 2^8 \cdot t_1}.$
         
### i = 3
Lookup $t_2$ s.t. 

$$
\begin{array}{ll}
\alpha_2 = (g^{-2^{24}})^{t_2} &\implies x_2 \cdot (g^{2^8})^{t_0 + 2^8\cdot {t_1}} = (g^{-2^{24}})^{t_2} \\
&\implies x_2 \cdot g^{(t_0 + 2^8 \cdot t_1 + 2^{16} \cdot t_2) \cdot 2^8} = 1.
\end{array}
$$

Define $\alpha_3 = x_3 \cdot g^{t_0 + 2^8 \cdot t_1 + 2^{16} \cdot t_2}.$

### Final result
Lookup $t_3$ such that

$$
\begin{array}{ll}
\alpha_3 = (g^{-2^{24}})^{t_3} &\implies x_3 \cdot g^{t_0 + 2^8\cdot {t_1} + 2^{16} \cdot t_2} = (g^{-2^{24}})^{t_3} \\
&\implies x_3 \cdot g^{t_0 + 2^8 \cdot t_1 + 2^{16} \cdot t_2 + 2^{24} \cdot t_3} = 1.
\end{array}
$$

Let $t = t_0 + 2^8 \cdot t_1 + 2^{16} \cdot t_2 + 2^{24} \cdot t_3$.

We can now write
$$
\begin{array}{lclcl}
x_3 \cdot g^{t} = 1 &\implies& x_3 &=& g^{-t} \\
&\implies& uv^2 &=& g^{-t} \\
&\implies& uv &=& v^{-1} \cdot g^{-t} \\
&\implies& uv \cdot g^{t / 2} &=& v^{-1} \cdot g^{-t / 2}.
\end{array}
$$

Squaring the RHS, we observe that $(v^{-1} g^{-t / 2})^2 = v^{-2}g^{-t} = u.$ Therefore,
the square root of $u$ is $uv \cdot g^{t / 2}$; the first part we computed earlier, and
the second part can be computed with three multiplications using lookups in $gtab$.


# 域

我们在 `halo2` 中使用的 [Pasta 曲线](https://electriccoin.co/blog/the-pasta-curves-for-halo-2-and-beyond/) 被设计为高度 2-adic 的，这意味着在每个域中存在一个大的 $2^S$ [乘法子群](../../background/fields.md#multiplicative-subgroups)。也就是说，我们可以将 $p - 1$ 写成 $p - 1 \equiv 2^S \cdot T$，其中 $T$ 为奇数。对于 Pallas 和 Vesta，$S = 32$；这有助于简化域的实现。

## Sarkar 平方根算法（基于表的变体）

我们使用 [Sarkar2020](https://eprint.iacr.org/2020/1407.pdf) 中的技术在 `halo2` 中计算[平方根](../../background/fields.md#square-roots)。该算法的直觉是，我们可以将任务分解为在每个乘法子群中计算平方根。

假设我们想找到 $u$ 在 Pasta 素数 $p$ 下的平方根，其中 $u$ 是 $\mathbb{Z}_p^\times$ 中的一个非零平方。我们定义一个 $2^S$ [单位根](../../background/fields.md#roots-of-unity) $g = z^T$，其中 $z$ 是 $\mathbb{Z}_p^\times$ 中的一个非平方，并预计算以下表：

$$
gtab = \begin{bmatrix}
g^0 & g^1 & ... & g^{2^8 - 1} \\
(g^{2^8})^0 & (g^{2^8})^1 & ... & (g^{2^8})^{2^8 - 1} \\
(g^{2^{16}})^0 & (g^{2^{16}})^1 & ... & (g^{2^{16}})^{2^8 - 1} \\
(g^{2^{24}})^0 & (g^{2^{24}})^1 & ... & (g^{2^{24}})^{2^8 - 1}
\end{bmatrix}
$$

$$
invtab = \begin{bmatrix}
(g^{-2^{24}})^0 & (g^{-2^{24}})^1 & ... & (g^{-2^{24}})^{2^8 - 1}
\end{bmatrix}
$$

令 $v = u^{(T-1)/2}$。然后我们可以将 $x = uv \cdot v = u^T$ 定义为 $2^S$ 乘法子群中的一个元素。

令 $x_3 = x, x_2 = x_3^{2^8}, x_1 = x_2^{2^8}, x_0 = x_1^{2^8}.$

### i = 0, 1
使用 $invtab$，我们查找 $t_0$ 使得
$$
x_0 = (g^{-2^{24}})^{t_0} \implies x_0 \cdot g^{t_0 \cdot 2^{24}} = 1.
$$

定义 $\alpha_1 = x_1 \cdot (g^{2^{16}})^{t_0}.$

### i = 2
查找 $t_1$ 使得
$$
\begin{array}{ll}
\alpha_1 = (g^{-2^{24}})^{t_1} &\implies x_1 \cdot (g^{2^{16}})^{t_0} = (g^{-2^{24}})^{t_1} \\
&\implies
x_1 \cdot g^{(t_0 + 2^8 \cdot t_1) \cdot 2^{16}} = 1.
\end{array}
$$

定义 $\alpha_2 = x_2 \cdot (g^{2^8})^{t_0 + 2^8 \cdot t_1}.$
         
### i = 3
查找 $t_2$ 使得

$$
\begin{array}{ll}
\alpha_2 = (g^{-2^{24}})^{t_2} &\implies x_2 \cdot (g^{2^8})^{t_0 + 2^8\cdot {t_1}} = (g^{-2^{24}})^{t_2} \\
&\implies x_2 \cdot g^{(t_0 + 2^8 \cdot t_1 + 2^{16} \cdot t_2) \cdot 2^8} = 1.
\end{array}
$$

定义 $\alpha_3 = x_3 \cdot g^{t_0 + 2^8 \cdot t_1 + 2^{16} \cdot t_2}.$

### 最终结果
查找 $t_3$ 使得

$$
\begin{array}{ll}
\alpha_3 = (g^{-2^{24}})^{t_3} &\implies x_3 \cdot g^{t_0 + 2^8\cdot {t_1} + 2^{16} \cdot t_2} = (g^{-2^{24}})^{t_3} \\
&\implies x_3 \cdot g^{t_0 + 2^8 \cdot t_1 + 2^{16} \cdot t_2 + 2^{24} \cdot t_3} = 1.
\end{array}
$$

令 $t = t_0 + 2^8 \cdot t_1 + 2^{16} \cdot t_2 + 2^{24} \cdot t_3$。

我们现在可以写出
$$
\begin{array}{lclcl}
x_3 \cdot g^{t} = 1 &\implies& x_3 &=& g^{-t} \\
&\implies& uv^2 &=& g^{-t} \\
&\implies& uv &=& v^{-1} \cdot g^{-t} \\
&\implies& uv \cdot g^{t / 2} &=& v^{-1} \cdot g^{-t / 2}.
\end{array}
$$

对 RHS 进行平方，我们观察到 $(v^{-1} g^{-t / 2})^2 = v^{-2}g^{-t} = u.$ 因此，$u$ 的平方根是 $uv \cdot g^{t / 2}$；第一部分我们之前已经计算过，第二部分可以通过在 $gtab$ 中查找并使用三次乘法来计算。