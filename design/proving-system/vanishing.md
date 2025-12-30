# Vanishing argument

Having committed to the circuit assignments, the prover now needs to demonstrate that the
various circuit relations are satisfied:

- The custom gates, represented by polynomials $\text{gate}_i(X)$.
- The rules of the lookup arguments.
- The rules of the equality constraint permutations.

Each of these relations is represented as a polynomial of degree $d$ (the maximum degree
of any of the relations) with respect to the circuit columns. Given that the degree of the
assignment polynomials for each column is $n - 1$, the relation polynomials have degree
$d(n - 1)$ with respect to $X$.

> In our [example](../proving-system.md#example), these would be the gate polynomials, of
> degree $3n - 3$:
>
> - $\text{gate}_0(X) = a_0(X) \cdot a_1(X) \cdot a_2(X \omega^{-1}) - a_3(X)$
> - $\text{gate}_1(X) = f_0(X \omega^{-1}) \cdot a_2(X)$
> - $\text{gate}_2(X) = f_0(X) \cdot a_3(X) \cdot a_0(X)$

A relation is satisfied if its polynomial is equal to zero. One way to demonstrate this is
to divide each polynomial relation by the vanishing polynomial $t(X) = (X^n - 1)$, which
is the lowest-degree polynomial that has roots at every $\omega^i$. If relation's polynomial
is perfectly divisible by $t(X)$, it is equal to zero over the domain (as desired).

This simple construction would require a polynomial commitment per relation. Instead, we
commit to all of the circuit relations simultaneously: the verifier samples $y$, and then
the prover constructs the quotient polynomial

$$h(X) = \frac{\text{gate}_0(X) + y \cdot \text{gate}_1(X) + \dots + y^i \cdot \text{gate}_i(X) + \dots}{t(X)},$$

where the numerator is a random (the prover commits to the cell assignments before the
verifier samples $y$) linear combination of the circuit relations.

- If the numerator polynomial (in formal indeterminate $X$) is perfectly divisible by
  $t(X)$, then with high probability all relations are satisfied.
- Conversely, if at least one relation is not satisfied, then with high probability
  $h(x) \cdot t(x)$ will not equal the evaluation of the numerator at $x$. In this case,
  the numerator polynomial would not be perfectly divisible by $t(X)$.

## Committing to $h(X)$

$h(X)$ has degree $d(n - 1) - n$ (because the divisor $t(X)$ has degree $n$). However, the
polynomial commitment scheme we use for Halo 2 only supports committing to polynomials of
degree $n - 1$ (which is the maximum degree that the rest of the protocol needs to commit
to). Instead of increasing the cost of the polynomial commitment scheme, the prover split
$h(X)$ into pieces of degree $n - 1$

$$h_0(X) + X^n h_1(X) + \dots + X^{n(d-1)} h_{d-1}(X),$$

and produces blinding commitments to each piece

$$\mathbf{H} = [\text{Commit}(h_0(X)), \text{Commit}(h_1(X)), \dots, \text{Commit}(h_{d-1}(X))].$$

## Evaluating the polynomials

At this point, we have committed to all properties of the circuit. The verifier now
wants to see if the prover committed to the correct $h(X)$ polynomial. The verifier
samples $x$, and the prover produces the purported evaluations of the various polynomials
at $x$, for all the relative offsets used in the circuit, as well as $h(X)$.

> In our [example](../proving-system.md#example), this would be:
>
> - $a_0(x)$
> - $a_1(x)$
> - $a_2(x)$, $a_2(x \omega^{-1})$
> - $a_3(x)$
> - $f_0(x)$, $f_0(x \omega^{-1})$
> - $h_0(x)$, ..., $h_{d-1}(x)$

The verifier checks that these evaluations satisfy the form of $h(X)$:

$$\frac{\text{gate}_0(x) + \dots + y^i \cdot \text{gate}_i(x) + \dots}{t(x)} = h_0(x) + \dots + x^{n(d-1)} h_{d-1}(x)$$

Now content that the evaluations collectively satisfy the gate constraints, the verifier
needs to check that the evaluations themselves are consistent with the original
[circuit commitments](circuit-commitments.md), as well as $\mathbf{H}$. To implement this
efficiently, we use a [multipoint opening argument](multipoint-opening.md).





# 消失论证

在完成电路赋值承诺后，证明者现在需要证明各种电路关系得到满足：

- 自定义门，由多项式 $\text{gate}_i(X)$ 表示。
- 查找论证的规则。
- 等式约束置换的规则。

这些关系中的每一个都表示为关于电路列的多项式，其最高次数为 $d$（所有关系中的最大次数）。由于每列的赋值多项式的次数为 $n - 1$，因此关系多项式关于 $X$ 的次数为 $d(n - 1)$。

> 在我们的[示例](../proving-system.md#example)中，这些将是门多项式，次数为 $3n - 3$：
>
> - $\text{gate}_0(X) = a_0(X) \cdot a_1(X) \cdot a_2(X \omega^{-1}) - a_3(X)$
> - $\text{gate}_1(X) = f_0(X \omega^{-1}) \cdot a_2(X)$
> - $\text{gate}_2(X) = f_0(X) \cdot a_3(X) \cdot a_0(X)$

如果关系多项式等于零，则该关系得到满足。一种证明方法是让每个关系多项式除以消失多项式 $t(X) = (X^n - 1)$，这是在每个 $\omega^i$ 处有根的最低次多项式。如果关系多项式能被 $t(X)$ 完全整除，则它在定义域上等于零（符合预期）。

这种简单构造需要每个关系都有一个多项式承诺。相反，我们同时承诺所有电路关系：验证者采样 $y$，然后证明者构造商多项式

$$h(X) = \frac{\text{gate}_0(X) + y \cdot \text{gate}_1(X) + \dots + y^i \cdot \text{gate}_i(X) + \dots}{t(X)},$$

其中分子是电路关系的随机（证明者在验证者采样 $y$ 之前承诺单元格赋值）线性组合。

- 如果分子多项式（关于形式未定元 $X$）能被 $t(X)$ 完全整除，则所有关系大概率得到满足。
- 相反，如果至少有一个关系未满足，则 $h(x) \cdot t(x)$ 大概率不等于分子在 $x$ 处的求值。此时，分子多项式将不能被 $t(X)$ 完全整除。

## 对 $h(X)$ 的承诺

$h(X)$ 的次数为 $d(n - 1) - n$（因为除数 $t(X)$ 的次数为 $n$）。然而，Halo 2 中使用的多项式承诺方案仅支持对次数为 $n - 1$ 的多项式进行承诺（这是协议其余部分需要承诺的最大次数）。为了避免增加多项式承诺方案的成本，证明者将 $h(X)$ 拆分为次数为 $n - 1$ 的片段

$$h_0(X) + X^n h_1(X) + \dots + X^{n(d-1)} h_{d-1}(X),$$

并对每个片段生成盲化承诺

$$\mathbf{H} = [\text{Commit}(h_0(X)), \text{Commit}(h_1(X)), \dots, \text{Commit}(h_{d-1}(X))].$$

## 多项式求值

此时，我们已经承诺了电路的所有属性。验证者现在希望确认证明者承诺的 $h(X)$ 多项式是否正确。验证者采样 $x$，证明者生成各种多项式在 $x$ 处的声称求值，包括电路中使用的所有相对偏移量以及 $h(X)$。

> 在我们的[示例](../proving-system.md#example)中，这将是：
>
> - $a_0(x)$
> - $a_1(x)$
> - $a_2(x)$, $a_2(x \omega^{-1})$
> - $a_3(x)$
> - $f_0(x)$, $f_0(x \omega^{-1})$
> - $h_0(x)$, ..., $h_{d-1}(x)$

验证者检查这些求值是否满足 $h(X)$ 的形式：

$$\frac{\text{gate}_0(x) + \dots + y^i \cdot \text{gate}_i(x) + \dots}{t(x)} = h_0(x) + \dots + x^{n(d-1)} h_{d-1}(x)$$

在确认求值共同满足门约束后，验证者需要检查这些求值本身是否与原始的[电路承诺](circuit-commitments.md)以及 $\mathbf{H}$ 一致。为了高效实现这一点，我们使用[多点打开论证](multipoint-opening.md)。
