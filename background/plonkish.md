# [WIP] PLONKish arithmetization

We call the field over which the circuit is defined $\mathbb{F} = \mathbb{F}_p$.

Let $n = 2^k$, and assume that $\omega$ is a primitive root of unity of order $n$ in
$\mathbb{F}^\times$, so that $\mathbb{F}^\times$ has a multiplicative subgroup
$\mathcal{H} = \{1, \omega, \omega^2, \cdots, \omega^{n-1}\}$. This forms a Lagrange
basis corresponding to the elements in the subgroup.

## Polynomial rules
A polynomial rule defines a constraint that must hold between its specified columns at
every row (i.e. at every element in the multiplicative subgroup).

e.g.

```text
a * sa + b * sb + a * b * sm + c * sc + PI = 0
```

## Columns
- **fixed columns**: fixed for all instances of a particular circuit. These include
  selector columns, which toggle parts of a polynomial rule "on" or "off" to form a
  "custom gate". They can also include any other fixed data.
- **advice columns**: variable values assigned in each instance of the circuit.
  Corresponds to the prover's secret witness.
- **public input**: like advice columns, but publicly known values.

Each column is a vector of $n$ values, e.g. $\mathbf{a} = [a_0, a_1, \cdots, a_{n-1}]$. We
can think of the vector as the evaluation form of the column polynomial
$a(X), X \in \mathcal{H}.$ To recover the coefficient form, we can use
[Lagrange interpolation](polynomials.md#lagrange-interpolation), such that
$a(\omega^i) = a_i.$

## Equality constraints
- Define permutation between a set of columns, e.g. $\sigma(a, b, c)$
- Assert equalities between specific cells in these columns, e.g. $b_1 = c_0$
- Construct permuted columns which should evaluate to same value as original columns

## Permutation grand product
$$Z(\omega^i) := \prod_{0 \leq j \leq i} \frac{C_k(\omega^j) + \beta\delta^k \omega^j + \gamma}{C_k(\omega^j) + \beta S_k(\omega^j) + \gamma},$$
where $i = 0, \cdots, n-1$ indexes over the size of the multiplicative subgroup, and
$k = 0, \cdots, m-1$ indexes over the advice columns involved in the permutation. This is
a running product, where each term includes the cumulative product of the terms before it.

> TODO: what is $\delta$? keep columns linearly independent

Check the constraints:

1. First term is equal to one
   $$\mathcal{L}_0(X) \cdot (1 - Z(X)) = 0$$

2. Running product is well-constructed. For each row, we check that this holds:
   $$Z(\omega^i) \cdot{(C(\omega^i) + \beta S_k(\omega^i) + \gamma)} - Z(\omega^{i-1}) \cdot{(C(\omega^i) + \delta^k \beta \omega^i + \gamma)} = 0$$
   Rearranging gives
   $$Z(\omega^i) = Z(\omega^{i-1}) \frac{C(\omega^i) + \beta\delta^k \omega^i + \gamma}{C(\omega^i) + \beta S_k(\omega^i) + \gamma},$$
   which is how we defined the grand product polynomial in the first place.

### Lookup
Reference: [Generic Lookups with PLONK (DRAFT)](/LTPc5f-3S0qNF6MtwD-Tdg?view)

### Vanishing argument
We want to check that the expressions defined by the gate constraints, permutation
constraints and lookup constraints evaluate to zero at all elements in the multiplicative
subgroup. To do this, the prover collapses all the expressions into one polynomial
$$H(X) = \sum_{i=0}^e y^i E_i(X),$$
where $e$ is the number of expressions and $y$ is a random challenge used to keep the
constraints linearly independent. The prover then divides this by the vanishing polynomial
(see section: [Vanishing polynomial](polynomials.md#vanishing-polynomial)) and commits to
the resulting quotient

$$\text{Commit}(Q(X)), \text{where } Q(X) = \frac{H(X)}{Z_H(X)}.$$

The verifier responds with a random evaluation point $x,$ to which the prover replies with
the claimed evaluations $q = Q(x), \{e_i\}_{i=0}^e = \{E_i(x)\}_{i=0}^e.$ Now, all that
remains for the verifier to check is that the evaluations satisfy

$$q \stackrel{?}{=} \frac{\sum_{i=0}^e y^i e_i}{Z_H(x)}.$$

Notice that we have yet to check that the committed polynomials indeed evaluate to the
claimed values at
$x, q \stackrel{?}{=} Q(x), \{e_i\}_{i=0}^e \stackrel{?}{=} \{E_i(x)\}_{i=0}^e.$
This check is handled by the polynomial commitment scheme (described in the next section).


# [WIP] PLONKish 算术化

我们定义电路所在的域为 $\mathbb{F} = \mathbb{F}_p$。

设 $n = 2^k$，并假设 $\omega$ 是 $\mathbb{F}^\times$ 中的一个 $n$ 阶本原单位根，因此 $\mathbb{F}^\times$ 有一个乘法子群 $\mathcal{H} = \{1, \omega, \omega^2, \cdots, \omega^{n-1}\}$。这形成了一个对应于子群元素的拉格朗日基。

## 多项式规则
多项式规则定义了在其指定的列之间必须满足的约束，该约束在每一行（即在乘法子群中的每个元素）都成立。

例如：

```text
a * sa + b * sb + a * b * sm + c * sc + PI = 0
```

## 列
- **固定列**：对于特定电路的所有实例都是固定的。这些包括选择器列，它们将多项式规则的部分“打开”或“关闭”以形成“自定义门”。它们还可以包括任何其他固定数据。
- **建议列**：在电路的每个实例中分配的变量值。对应于证明者的秘密见证。
- **公共输入**：类似于建议列，但值是公开已知的。

每列是一个包含 $n$ 个值的向量，例如 $\mathbf{a} = [a_0, a_1, \cdots, a_{n-1}]$。我们可以将向量视为列多项式 $a(X), X \in \mathcal{H}$ 的评估形式。为了恢复系数形式，我们可以使用[拉格朗日插值](polynomials.md#lagrange-interpolation)，使得 $a(\omega^i) = a_i$。

## 等式约束
- 定义列之间的置换，例如 $\sigma(a, b, c)$
- 断言这些列中特定单元格之间的等式，例如 $b_1 = c_0$
- 构造置换列，这些列应评估为与原始列相同的值

## 置换大积
$$Z(\omega^i) := \prod_{0 \leq j \leq i} \frac{C_k(\omega^j) + \beta\delta^k \omega^j + \gamma}{C_k(\omega^j) + \beta S_k(\omega^j) + \gamma},$$
其中 $i = 0, \cdots, n-1$ 索引乘法子群的大小，$k = 0, \cdots, m-1$ 索引置换中涉及的建议列。这是一个累积积，其中每一项包括之前所有项的累积积。

> TODO: $\delta$ 是什么？保持列线性独立

检查约束：

1. 第一项等于一
   $$\mathcal{L}_0(X) \cdot (1 - Z(X)) = 0$$

2. 累积积构造正确。对于每一行，我们检查以下是否成立：
   $$Z(\omega^i) \cdot{(C(\omega^i) + \beta S_k(\omega^i) + \gamma)} - Z(\omega^{i-1}) \cdot{(C(\omega^i) + \delta^k \beta \omega^i + \gamma)} = 0$$
   重新排列得到
   $$Z(\omega^i) = Z(\omega^{i-1}) \frac{C(\omega^i) + \beta\delta^k \omega^i + \gamma}{C(\omega^i) + \beta S_k(\omega^i) + \gamma},$$
   这正是我们最初定义大积多项式的方式。

### 查找
参考：[Generic Lookups with PLONK (DRAFT)](/LTPc5f-3S0qNF6MtwD-Tdg?view)

### 消失论证
我们希望检查由门约束、置换约束和查找约束定义的表达式在乘法子群中的所有元素处是否评估为零。为此，证明者将所有表达式折叠成一个多项式
$$H(X) = \sum_{i=0}^e y^i E_i(X),$$
其中 $e$ 是表达式的数量，$y$ 是一个随机挑战，用于保持约束的线性独立性。证明者然后将该多项式除以消失多项式（参见章节：[消失多项式](polynomials.md#vanishing-polynomial)）并承诺所得商

$$\text{Commit}(Q(X)), \text{其中 } Q(X) = \frac{H(X)}{Z_H(X)}.$$

验证者响应一个随机评估点 $x$，证明者回复声称的评估值 $q = Q(x), \{e_i\}_{i=0}^e = \{E_i(x)\}_{i=0}^e$。现在，验证者只需检查评估是否满足

$$q \stackrel{?}{=} \frac{\sum_{i=0}^e y^i e_i}{Z_H(x)}.$$

注意，我们尚未检查承诺的多项式是否确实在 $x$ 处评估为声称的值，$q \stackrel{?}{=} Q(x), \{e_i\}_{i=0}^e \stackrel{?}{=} \{E_i(x)\}_{i=0}^e$。此检查由多项式承诺方案处理（在下一节中描述）。

