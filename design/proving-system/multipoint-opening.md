# Multipoint opening argument

Consider the commitments $A, B, C, D$ to polynomials $a(X), b(X), c(X), d(X)$.
Let's say that $a$ and $b$ were queried at the point $x$, while $c$ and $d$
were queried at both points $x$ and $\omega x$. (Here, $\omega$ is the primitive
root of unity in the multiplicative subgroup over which we constructed the
polynomials).

To open these commitments, we could create a polynomial $Q$ for each point that we queried
at (corresponding to each relative rotation used in the circuit). But this would not be
efficient in the circuit; for example, $c(X)$ would appear in multiple polynomials.

Instead, we can group the commitments by the sets of points at which they were queried:
$$
\begin{array}{cccc}
&\{x\}&     &\{x, \omega x\}& \\
 &A&            &C& \\
 &B&            &D&
\end{array}
$$

For each of these groups, we combine them into a polynomial set, and create a single $Q$
for that set, which we open at each rotation.

## Optimization steps

The multipoint opening optimization takes as input:

- A random $x$ sampled by the verifier, at which we evaluate $a(X), b(X), c(X), d(X)$.
- Evaluations of each polynomial at each point of interest, provided by the prover:
  $a(x), b(x), c(x), d(x), c(\omega x), d(\omega x)$

These are the outputs of the [vanishing argument](vanishing.md#evaluating-the-polynomials).

The multipoint opening optimization proceeds as such:

1. Sample random $x_1$, to keep $a, b, c, d$ linearly independent.
2. Accumulate polynomials and their corresponding evaluations according
   to the point set at which they were queried:
    `q_polys`:
    $$
    \begin{array}{rccl}
    q_1(X) &=& a(X) &+& x_1 b(X) \\
    q_2(X) &=& c(X) &+& x_1 d(X)
    \end{array}
    $$
    `q_eval_sets`:
    ```math
            [
                [a(x) + x_1 b(x)],
                [
                    c(x) + x_1 d(x),
                    c(\omega x) + x_1 d(\omega x)
                ]
            ]
    ```
    NB: `q_eval_sets` is a vector of sets of evaluations, where the outer vector
    corresponds to the point sets, which in this example are $\{x\}$ and $\{x, \omega x\}$,
    and the inner vector corresponds to the points in each set.
3. Interpolate each set of values in `q_eval_sets`:
    `r_polys`:
    $$
    \begin{array}{cccc}
    r_1(X) s.t.&&& \\
        &r_1(x) &=& a(x) + x_1 b(x) \\
    r_2(X) s.t.&&& \\
        &r_2(x) &=& c(x) + x_1 d(x) \\
        &r_2(\omega x) &=& c(\omega x) + x_1 d(\omega x) \\
    \end{array}
    $$
4. Construct `f_polys` which check the correctness of `q_polys`:
    `f_polys`
    $$
    \begin{array}{rcl}
    f_1(X) &=& \frac{ q_1(X) - r_1(X)}{X - x} \\
    f_2(X) &=& \frac{ q_2(X) - r_2(X)}{(X - x)(X - \omega x)} \\
    \end{array}
    $$

    If $q_1(x) = r_1(x)$, then $f_1(X)$ should be a polynomial.
    If $q_2(x) = r_2(x)$ and $q_2(\omega x) = r_2(\omega x)$
    then $f_2(X)$ should be a polynomial.
5. Sample random $x_2$ to keep the `f_polys` linearly independent.
6. Construct $f(X) = f_1(X) + x_2 f_2(X)$.
7.  Sample random $x_3$, at which we evaluate $f(X)$:
    $$
    \begin{array}{rcccl}
    f(x_3) &=& f_1(x_3) &+& x_2 f_2(x_3) \\
           &=& \frac{q_1(x_3) - r_1(x_3)}{x_3 - x} &+& x_2\frac{q_2(x_3) - r_2(x_3)}{(x_3 - x)(x_3 - \omega x)}
    \end{array}
    $$
8.  Sample random $x_4$ to keep $f(X)$ and `q_polys` linearly independent.
9.  Construct `final_poly`, $$final\_poly(X) = f(X) + x_4 q_1(X) + x_4^2 q_2(X),$$
    which is the polynomial we commit to in the inner product argument.


# 多点打开论证

考虑对多项式 $a(X), b(X), c(X), d(X)$ 的承诺 $A, B, C, D$。假设 $a$ 和 $b$ 在点 $x$ 处被查询，而 $c$ 和 $d$ 在点 $x$ 和 $\omega x$ 处被查询。（这里，$\omega$ 是在构造多项式的乘法子群中的单位原根）。

为了打开这些承诺，我们可以为每个查询的点创建一个多项式 $Q$（对应于电路中使用的每个相对旋转）。但这在电路中并不高效；例如，$c(X)$ 会出现在多个多项式中。

相反，我们可以根据它们被查询的点集对承诺进行分组：
$$
\begin{array}{cccc}
&\{x\}&     &\{x, \omega x\}& \\
 &A&            &C& \\
 &B&            &D&
\end{array}
$$

对于每个组，我们将它们组合成一个多项式集，并为该集创建一个单一的 $Q$，在每个旋转处打开它。

## 优化步骤

多点打开优化接受以下输入：

- 由验证者采样的随机点 $x$，在此处我们评估 $a(X), b(X), c(X), d(X)$。
- 由证明者提供的每个多项式在每个感兴趣点的评估值：
  $a(x), b(x), c(x), d(x), c(\omega x), d(\omega x)$

这些是[消失论证](vanishing.md#evaluating-the-polynomials)的输出。

多点打开优化按以下步骤进行：

1. 采样随机 $x_1$，以保持 $a, b, c, d$ 线性独立。
2. 根据它们被查询的点集累积多项式及其对应的评估值：
    `q_polys`:
    $$
    \begin{array}{rccl}
    q_1(X) &=& a(X) &+& x_1 b(X) \\
    q_2(X) &=& c(X) &+& x_1 d(X)
    \end{array}
    $$
    `q_eval_sets`:
    ```math
            [
                [a(x) + x_1 b(x)],
                [
                    c(x) + x_1 d(x),
                    c(\omega x) + x_1 d(\omega x)
                ]
            ]
    ```
    注意：`q_eval_sets` 是一个评估集的向量，其中外部向量对应于点集，在此示例中为 $\{x\}$ 和 $\{x, \omega x\}$，内部向量对应于每个集中的点。
3. 对 `q_eval_sets` 中的每组值进行插值：
    `r_polys`:
    $$
    \begin{array}{cccc}
    r_1(X) s.t.&&& \\
        &r_1(x) &=& a(x) + x_1 b(x) \\
    r_2(X) s.t.&&& \\
        &r_2(x) &=& c(x) + x_1 d(x) \\
        &r_2(\omega x) &=& c(\omega x) + x_1 d(\omega x) \\
    \end{array}
    $$
4. 构造 `f_polys` 以检查 `q_polys` 的正确性：
    `f_polys`
    $$
    \begin{array}{rcl}
    f_1(X) &=& \frac{ q_1(X) - r_1(X)}{X - x} \\
    f_2(X) &=& \frac{ q_2(X) - r_2(X)}{(X - x)(X - \omega x)} \\
    \end{array}
    $$

    如果 $q_1(x) = r_1(x)$，则 $f_1(X)$ 应该是一个多项式。
    如果 $q_2(x) = r_2(x)$ 且 $q_2(\omega x) = r_2(\omega x)$，
    则 $f_2(X)$ 应该是一个多项式。
5. 采样随机 $x_2$ 以保持 `f_polys` 线性独立。
6. 构造 $f(X) = f_1(X) + x_2 f_2(X)$。
7. 采样随机 $x_3$，在此处我们评估 $f(X)$：
    $$
    \begin{array}{rcccl}
    f(x_3) &=& f_1(x_3) &+& x_2 f_2(x_3) \\
           &=& \frac{q_1(x_3) - r_1(x_3)}{x_3 - x} &+& x_2\frac{q_2(x_3) - r_2(x_3)}{(x_3 - x)(x_3 - \omega x)}
    \end{array}
    $$
8. 采样随机 $x_4$ 以保持 $f(X)$ 和 `q_polys` 线性独立。
9. 构造 `final_poly`，$$final\_poly(X) = f(X) + x_4 q_1(X) + x_4^2 q_2(X),$$
    这是我们在内积论证中承诺的多项式。
