# Tips and tricks

This section contains various ideas and snippets that you might find useful while writing
halo2 circuits.

## Small range constraints

A common constraint used in R1CS circuits is the boolean constraint: $b * (1 - b) = 0$.
This constraint can only be satisfied by $b = 0$ or $b = 1$.

In halo2 circuits, you can similarly constrain a cell to have one of a small set of
values. For example, to constrain $a$ to the range $[0..5]$, you would create a gate of
the form:

$$a \cdot (1 - a) \cdot (2 - a) \cdot (3 - a) \cdot (4 - a) = 0$$

while to constrain $c$ to be either 7 or 13, you would use:

$$(7 - c) \cdot (13 - c) = 0$$

> The underlying principle here is that we create a polynomial constraint with roots at
> each value in the set of possible values we want to allow. In R1CS circuits, the maximum
> supported polynomial degree is 2 (due to all constraints being of the form $a * b = c$).
> In halo2 circuits, you can use arbitrary-degree polynomials - with the proviso that
> higher-degree constraints are more expensive to use.

Note that the roots don't have to be constants; for example $(a - x) \cdot (a - y) \cdot (a - z) = 0$ will constrain $a$ to be equal to one of $\{ x, y, z \}$ where the latter can be arbitrary polynomials, as long as the whole expression stays within the maximum degree bound.

## Small set interpolation
We can use Lagrange interpolation to create a polynomial constraint that maps
$f(X) = Y$ for small sets of $X \in \{x_i\}, Y \in \{y_i\}$. 

For instance, say we want to map a 2-bit value to a "spread" version interleaved
with zeros. We first precompute the evaluations at each point:

$$
\begin{array}{rcl}
00 \rightarrow 0000 &\implies& 0 \rightarrow 0 \\
01 \rightarrow 0001 &\implies& 1 \rightarrow 1 \\
10 \rightarrow 0100 &\implies& 2 \rightarrow 4 \\
11 \rightarrow 0101 &\implies& 3 \rightarrow 5
\end{array}
$$

Then, we construct the Lagrange basis polynomial for each point using the
identity:
$$\mathcal{l}_j(X) = \prod_{0 \leq m < k,\; m \neq j} \frac{x - x_m}{x_j - x_m},$$
where $k$ is the number of data points. ($k = 4$ in our example above.)

Recall that the Lagrange basis polynomial $\mathcal{l}_j(X)$ evaluates to $1$ at
$X = x_j$ and $0$ at all other $x_i, j \neq i.$

Continuing our example, we get four Lagrange basis polynomials:

$$
\begin{array}{ccc}
l_0(X) &=& \frac{(X - 3)(X - 2)(X - 1)}{(-3)(-2)(-1)} \\[1ex]
l_1(X) &=& \frac{(X - 3)(X - 2)(X)}{(-2)(-1)(1)} \\[1ex]
l_2(X) &=& \frac{(X - 3)(X - 1)(X)}{(-1)(1)(2)} \\[1ex]
l_3(X) &=& \frac{(X - 2)(X - 1)(X)}{(1)(2)(3)}
\end{array}
$$

Our polynomial constraint is then

$$
\begin{array}{cccccccccccl}
&f(0) \cdot l_0(X) &+& f(1) \cdot l_1(X) &+& f(2) \cdot l_2(X) &+& f(3) \cdot l_3(X) &-& f(X) &=& 0 \\
\implies& 0 \cdot l_0(X) &+& 1 \cdot l_1(X) &+& 4 \cdot l_2(X) &+& 5 \cdot l_3(X) &-& f(X) &=& 0. \\
\end{array}
$$


# 技巧与小贴士

本节包含各种在编写 halo2 电路时可能有用的想法和代码片段。

## 小范围约束

R1CS 电路中常用的一个约束是布尔约束：$b * (1 - b) = 0$。这个约束只能通过 $b = 0$ 或 $b = 1$ 来满足。

在 halo2 电路中，您可以类似地约束一个单元格具有一小部分值。例如，要将 $a$ 约束在范围 $[0..5]$ 内，您可以创建一个如下形式的门：

$$a \cdot (1 - a) \cdot (2 - a) \cdot (3 - a) \cdot (4 - a) = 0$$

而要将 $c$ 约束为 7 或 13，您可以使用：

$$(7 - c) \cdot (13 - c) = 0$$

> 这里的基本原理是，我们创建一个多项式约束，其根位于我们希望允许的可能值集合中的每个值处。在 R1CS 电路中，支持的最大多项式次数为 2（因为所有约束都是 $a * b = c$ 的形式）。在 halo2 电路中，您可以使用任意次数的多项式——但要注意，更高次数的约束使用起来更昂贵。

注意，根不必是常数；例如，$(a - x) \cdot (a - y) \cdot (a - z) = 0$ 将约束 $a$ 等于 $\{ x, y, z \}$ 中的一个，其中后者可以是任意多项式，只要整个表达式保持在最大次数限制内。

## 小集合插值

我们可以使用拉格朗日插值来创建一个多项式约束，将 $f(X) = Y$ 映射到小的集合 $X \in \{x_i\}, Y \in \{y_i\}$。

例如，假设我们想将一个 2 位值映射到一个与零交错的“扩展”版本。我们首先在每个点处预先计算评估值：

$$
\begin{array}{rcl}
00 \rightarrow 0000 &\implies& 0 \rightarrow 0 \\
01 \rightarrow 0001 &\implies& 1 \rightarrow 1 \\
10 \rightarrow 0100 &\implies& 2 \rightarrow 4 \\
11 \rightarrow 0101 &\implies& 3 \rightarrow 5
\end{array}
$$

然后，我们使用以下恒等式为每个点构造拉格朗日基多项式：
$$\mathcal{l}_j(X) = \prod_{0 \leq m < k,\; m \neq j} \frac{x - x_m}{x_j - x_m},$$
其中 $k$ 是数据点的数量。（在上面的示例中，$k = 4$。）

回想一下，拉格朗日基多项式 $\mathcal{l}_j(X)$ 在 $X = x_j$ 处评估为 $1$，在所有其他 $x_i, j \neq i$ 处评估为 $0$。

继续我们的示例，我们得到四个拉格朗日基多项式：

$$
\begin{array}{ccc}
l_0(X) &=& \frac{(X - 3)(X - 2)(X - 1)}{(-3)(-2)(-1)} \\[1ex]
l_1(X) &=& \frac{(X - 3)(X - 2)(X)}{(-2)(-1)(1)} \\[1ex]
l_2(X) &=& \frac{(X - 3)(X - 1)(X)}{(-1)(1)(2)} \\[1ex]
l_3(X) &=& \frac{(X - 2)(X - 1)(X)}{(1)(2)(3)}
\end{array}
$$

我们的多项式约束是

$$
\begin{array}{cccccccccccl}
&f(0) \cdot l_0(X) &+& f(1) \cdot l_1(X) &+& f(2) \cdot l_2(X) &+& f(3) \cdot l_3(X) &-& f(X) &=& 0 \\
\implies& 0 \cdot l_0(X) &+& 1 \cdot l_1(X) &+& 4 \cdot l_2(X) &+& 5 \cdot l_3(X) &-& f(X) &=& 0. \\
\end{array}
$$
