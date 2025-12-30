# Elliptic curves

Elliptic curves constructed over finite fields are another important cryptographic tool.

We use elliptic curves because they provide a cryptographic [group](fields.md#Groups),
i.e. a group in which the discrete logarithm problem (discussed below) is hard.

There are several ways to define the curve equation, but for our purposes, let
$\mathbb{F}_p$ be a large (255-bit) field, and then let the set of solutions $(x, y)$ to
$y^2 = x^3 + b$ for some constant $b$ define the $\mathbb{F}_p$-rational points on an
elliptic curve $E(\mathbb{F}_p)$. These $(x, y)$ coordinates are called "affine
coordinates". Each of the $\mathbb{F}_p$-rational points, together with a "point at
infinity" $\mathcal{O}$ that serves as the group identity, can be interpreted as an
element of a group. By convention, elliptic curve groups are written additively.

![](https://i.imgur.com/JvLS6yE.png)
*"Three points on a line sum to zero, which is the point at infinity."*

The group addition law is simple: to add two points together, find the line that
intersects both points and obtain the third point, and then negate its $y$-coordinate. The
case that a point is being added to itself, called point doubling, requires special
handling: we find the line tangent to the point, and then find the single other point that
intersects this line and then negate. Otherwise, in the event that a point is being
"added" to its negation, the result is the point at infinity.

The ability to add and double points naturally gives us a way to scale them by integers,
called _scalars_. The number of points on the curve is the group order. If this number
is a prime $q$, then the scalars can be considered as elements of a _scalar field_,
$\mathbb{F}_q$.

Elliptic curves, when properly designed, have an important security property. Given two
random elements $G, H \in E(\mathbb{F}_p)$ finding $a$ such that $[a] G = H$, otherwise
known as the discrete log of $H$ with respect to $G$, is considered computationally
infeasible with classical computers. This is called the elliptic curve discrete log
assumption.

If an elliptic curve group $\mathbb{G}$ has prime order $q$ (like the ones used in Halo 2),
then it is a finite cyclic group. Recall from the section on [groups](fields.md#Groups)
that this implies it is isomorphic to $\mathbb{Z}/q\mathbb{Z}$, or equivalently, to the
scalar field $\mathbb{F}_q$. Each possible generator $G$ fixes the isomorphism; then
an element on the scalar side is precisely the discrete log of the corresponding group
element with respect to $G$. In the case of a cryptographically secure elliptic curve,
the isomorphism is hard to compute in the $\mathbb{G} \rightarrow \mathbb{F}_q$ direction
because the elliptic curve discrete log problem is hard.

> It is sometimes helpful to make use of this isomorphism by thinking of group-based
> cryptographic protocols and algorithms in terms of the scalars instead of in terms of
> the group elements. This can make proofs and notation simpler.
>
> For instance, it has become common in papers on proof systems to use the notation $[x]$
> to denote a group element with discrete log $x$, where the generator is implicit.
>
> We also used this idea in the
> "[distinct-x theorem](https://zips.z.cash/protocol/protocol.pdf#thmdistinctx)",
> in order to prove correctness of optimizations
> [for elliptic curve scalar multiplication](https://github.com/zcash/zcash/issues/3924)
> in Sapling, and an endomorphism-based optimization in Appendix C of the original
> [Halo paper](https://eprint.iacr.org/2019/1021.pdf).

## Curve arithmetic

### Point doubling

The simplest situation is doubling a point $(x_0, y_0)$. Continuing with our example
$y^2 = x^3 + b$, this is done first by computing the derivative
$$
\lambda = \frac{\mathrm{d}y}{\mathrm{d}x} = \frac{3x^2}{2y}.
$$

To obtain expressions for $(x_1, y_1) = (x_0, y_0) + (x_0, y_0),$ we consider 

$$
\begin{aligned}
\frac{-y_1 - y_0}{x_1 - x_0} = \lambda &\implies -y_1 = \lambda(x_1 - x_0) + y_0 \\
&\implies \boxed{y_1 = \lambda(x_0 - x_1) - y_0}.
\end{aligned}
$$

To get the expression for $x_1,$ we substitute $y = \lambda(x_0 - x) - y_0$ into the
elliptic curve equation:

$$
\begin{aligned}
y^2 = x^3 + b &\implies (\lambda(x_0 - x) - y_0)^2 = x^3 + b \\
&\implies x^3 - \lambda^2 x^2 + \cdots = 0 \leftarrow\text{(rearranging terms)} \\
&= (x - x_0)(x - x_0)(x - x_1) \leftarrow\text{(known roots $x_0, x_0, x_1$)} \\
&= x^3 - (x_0 + x_0 + x_1)x^2 + \cdots.
\end{aligned}
$$

Comparing coefficients for the $x^2$ term gives us
$\lambda^2 = x_0 + x_0 + x_1 \implies \boxed{x_1 = \lambda^2 - 2x_0}.$


### Projective coordinates
This unfortunately requires an expensive inversion of $2y$. We can avoid this by arranging
our equations to "defer" the computation of the inverse, since we often do not need the
actual affine $(x', y')$ coordinate of the resulting point immediately after an individual
curve operation. Let's introduce a third coordinate $Z$ and scale our curve equation by
$Z^3$ like so:

$$
Z^3 y^2 = Z^3 x^3 + Z^3 b
$$

Our original curve is just this curve at the restriction $Z = 1$. If we allow the affine
point $(x, y)$ to be represented by $X = xZ$, $Y = yZ$ and $Z \neq 0$ then we have the
[homogenous projective curve](https://en.wikipedia.org/wiki/Homogeneous_coordinates)

$$
Y^2 Z = X^3 + Z^3 b.
$$

Obtaining $(x, y)$ from $(X, Y, Z)$ is as simple as computing $(X/Z, Y/Z)$ when
$Z \neq 0$. (When $Z = 0,$ we are dealing with the point at infinity $O := (0:1:0)$.) In
this form, we now have a convenient way to defer the inversion required by doubling a
point. The general strategy is to express $x', y'$ as rational functions using $x = X/Z$
and $y = Y/Z$, rearrange to make their denominators the same, and then take the resulting
point $(X, Y, Z)$ to have $Z$ be the shared denominator and $X = x'Z, Y = y'Z$.

> Projective coordinates are often, but not always, more efficient than affine
> coordinates. There may be exceptions to this when either we have a different way to
> apply Montgomery's trick, or when we're in the circuit setting where multiplications and
> inversions are about equally as expensive (at least in terms of circuit size).

The following shows an example of doubling a point $(X, Y, Z) = (xZ, yZ, Z)$ without an
inversion. Substituting with $X, Y, Z$ gives us
$$
\lambda = \frac{3x^2}{2y} = \frac{3(X/Z)^2}{2(Y/Z)} = \frac{3 X^2}{2YZ}
$$

and gives us
$$
\begin{aligned}
x' &= \lambda^2 - 2x \\
&= \lambda^2 - \frac{2X}{Z} \\
&= \frac{9 X^4}{4Y^2Z^2} - \frac{2X}{Z} \\
&= \frac{9 X^4 - 8XY^2Z}{4Y^2Z^2} \\
&= \frac{18 X^4 Y Z - 16XY^3Z^2}{8Y^3Z^3} \\
\\
y' &= \lambda (x - x') - y \\
&= \lambda (\frac{X}{Z} - \frac{9 X^4 - 8XY^2Z}{4Y^2Z^2}) - \frac{Y}{Z} \\
&= \frac{3 X^2}{2YZ} (\frac{X}{Z} - \frac{9 X^4 - 8XY^2Z}{4Y^2Z^2}) - \frac{Y}{Z} \\
&= \frac{3 X^3}{2YZ^2} - \frac{27 X^6 - 24X^3Y^2Z}{8Y^3Z^3} - \frac{Y}{Z} \\
&= \frac{12 X^3Y^2Z - 8Y^4Z^2 - 27 X^6 + 24X^3Y^2Z}{8Y^3Z^3}
\end{aligned}
$$

Notice how the denominators of $x'$ and $y'$ are the same. Thus, instead of computing
$(x', y')$ we can compute $(X, Y, Z)$ with $Z = 8Y^3Z^3$ and $X, Y$ set to the
corresponding numerators such that $X/Z = x'$ and $Y/Z = y'$. This completely avoids the
need to perform an inversion when doubling, and something analogous to this can be done
when adding two distinct points.

### Point addition
We now add two points with distinct $x$-coordinates, $P = (x_0, y_0)$ and $Q = (x_1, y_1),$
where $x_0 \neq x_1,$ to obtain $R = P + Q = (x_2, y_2).$ The line $\overline{PQ}$ has slope
$$\lambda = \frac{y_1 - y_0}{x_1 - x_0} \implies y - y_0 = \lambda \cdot (x - x_0).$$

Using the expression for $\overline{PQ}$, we compute $y$-coordinate $-y_2$ of $-R$ as:
$$-y_2 - y_0 = \lambda \cdot (x_2 - x_0) \implies \boxed{y_2 =\lambda (x_0 - x_2) - y_0}.$$

Plugging the expression for $\overline{PQ}$ into the curve equation $y^2 = x^3 + b$ yields
$$
\begin{aligned}
y^2 = x^3 + b &\implies (\lambda \cdot (x - x_0) + y_0)^2 = x^3 + b \\
&\implies x^3 - \lambda^2 x^2 + \cdots = 0 \leftarrow\text{(rearranging terms)} \\
&= (x - x_0)(x - x_1)(x - x_2) \leftarrow\text{(known roots $x_0, x_1, x_2$)} \\
&= x^3 - (x_0 + x_1 + x_2)x^2 + \cdots.
\end{aligned}
$$

Comparing coefficients for the $x^2$ term gives us
$\lambda^2 = x_0 + x_1 + x_2 \implies \boxed{x_2 = \lambda^2 - x_0 - x_1}$.

----------

Important notes:

* There exist efficient formulae[^complete-formulae] for point addition that do not have
  edge cases (so-called "complete" formulae) and that unify the addition and doubling
  cases together. The result of adding a point to its negation using those formulae
  produces $Z = 0$, which represents the point at infinity.
* In addition, there are other models like the Jacobian representation where
  $(x, y) = (xZ^2, yZ^3, Z)$ where the curve is rescaled by $Z^6$ instead of $Z^3$, and
  this representation has even more efficient arithmetic but no unified/complete formulae.
* We can easily compare two curve points $(X_1, Y_1, Z_1)$ and $(X_2, Y_2, Z_2)$ for
  equality in the homogenous projective coordinate space by "homogenizing" their
  Z-coordinates; the checks become $X_1 Z_2 = X_2 Z_1$ and $Y_1 Z_2 = Y_2 Z_1$.

## Curve endomorphisms

Imagine that $\mathbb{F}_p$ has a primitive cube root of unity, or in other words that
$3 | p - 1$ and so an element $\zeta_p$ generates a $3$-order multiplicative subgroup.
Notice that a point $(x, y)$ on our example elliptic curve $y^2 = x^3 + b$ has two cousin
points: $(\zeta_p x,y), (\zeta_p^2 x,y)$, because the computation $x^3$ effectively kills the
$\zeta$ component of the $x$-coordinate. Applying the map $(x, y) \mapsto (\zeta_p x, y)$
is an application of an endomorphism over the curve. The exact mechanics involved are
complicated, but when the curve has a prime $q$ number of points (and thus a prime
"order") the effect of the endomorphism is to multiply the point by a scalar in
$\mathbb{F}_q$ which is also a primitive cube root $\zeta_q$ in the scalar field.

## Curve point compression
Given a point on the curve $P = (x,y)$, we know that its negation $-P = (x, -y)$ is also
on the curve. To uniquely specify a point, we need only encode its $x$-coordinate along
with the sign of its $y$-coordinate.

### Serialization
As mentioned in the [Fields](./fields.md) section, we can interpret the least significant
bit of a field element as its "sign", since its additive inverse will always have the
opposite LSB. So we record the LSB of the $y$-coordinate as `sign`.

Pallas and Vesta are defined over the $\mathbb{F}_p$ and $\mathbb{F}_q$ fields, which
elements can be expressed in $255$ bits. This conveniently leaves one unused bit in a
32-byte representation. We pack the $y$-coordinate `sign` bit into the highest bit in
the representation of the $x$-coordinate:

```text
         <----------------------------------- x --------------------------------->
Enc(P) = [_ _ _ _ _ _ _ _] [_ _ _ _ _ _ _ _] ... [_ _ _ _ _ _ _ _] [_ _ _ _ _ _ _ sign]
          ^                <------------------------------------->                 ^
         LSB                              30 bytes                                MSB
```

The "point at infinity" $\mathcal{O}$ that serves as the group identity, does not have an
affine $(x, y)$ representation. However, it turns out that there are no points on either
the Pallas or Vesta curve with $x = 0$ or $y = 0$. We therefore use the "fake" affine
coordinates $(0, 0)$ to encode $\mathcal{O}$, which results in the all-zeroes 32-byte
array.

### Deserialization
When deserializing a compressed curve point, we first read the most significant bit as
`ysign`, the sign of the $y$-coordinate. Then, we set this bit to zero to recover the
original $x$-coordinate.

If $x = 0, y = 0,$ we return the "point at infinity" $\mathcal{O}$. Otherwise, we proceed
to compute $y = \sqrt{x^3 + b}.$ Here, we read the least significant bit of $y$ as `sign`.
If `sign == ysign`, we already have the correct sign and simply return the curve point
$(x, y)$. Otherwise, we negate $y$ and return $(x, -y)$.

## Cycles of curves
Let $E_p$ be an elliptic curve over a finite field $\mathbb{F}_p,$ where $p$ is a prime.
We denote this by $E_p/\mathbb{F}_p.$ and we denote the group of points of $E_p$ over
$\mathbb{F}_p,$ with order $q = \#E(\mathbb{F}_p).$ For this curve, we call $\mathbb{F}_p$
the "base field" and  $\mathbb{F}_q$ the "scalar field".

We instantiate our proof system over the elliptic curve $E_p/\mathbb{F}_p$. This allows us
to prove statements about $\mathbb{F}_q$-arithmetic circuit satisfiability.

> **(aside) If our curve $E_p$ is over $\mathbb{F}_p,$ why is the arithmetic circuit instead in $\mathbb{F}_q$?**
> The proof system is basically working on encodings of the scalars in the circuit (or
> more precisely, commitments to polynomials whose coefficients are scalars). The scalars
> are in $\mathbb{F}_q$ when their encodings/commitments are elliptic curve points in
> $E_p/\mathbb{F}_p$.

However, most of the verifier's arithmetic computations are over the base field
$\mathbb{F}_p,$ and are thus efficiently expressed as an $\mathbb{F}_p$-arithmetic
circuit.

> **(aside) Why are the verifier's computations (mainly) over $\mathbb{F}_p$?**
> The Halo 2 verifier actually has to perform group operations using information output by
> the circuit. Group operations like point doubling and addition use arithmetic in
> $\mathbb{F}_p$, because the coordinates of points are in $\mathbb{F}_p.$ 

This motivates us to construct another curve with scalar field $\mathbb{F}_p$, which has
an $\mathbb{F}_p$-arithmetic circuit that can efficiently verify proofs from the first
curve. As a bonus, if this second curve had base field $E_q/\mathbb{F}_q,$ it would
generate proofs that could be efficiently verified in the first curve's
$\mathbb{F}_q$-arithmetic circuit. In other words, we instantiate a second proof system
over $E_q/\mathbb{F}_q,$ forming a 2-cycle with the first:

![](https://i.imgur.com/bNMyMRu.png)

### TODO: Pallas-Vesta curves
Reference: https://github.com/zcash/pasta

## Hashing to curves

Sometimes it is useful to be able to produce a random point on an elliptic curve
$E_p/\mathbb{F}_p$ corresponding to some input, in such a way that no-one will know its
discrete logarithm (to any other base).

This is described in detail in the [Internet draft on Hashing to Elliptic Curves][cfrg-hash-to-curve].
Several algorithms can be used depending on efficiency and security requirements. The
framework used in the Internet Draft makes use of several functions:

* ``hash_to_field``: takes a byte sequence input and maps it to a element in the base
  field $\mathbb{F}_p$
* ``map_to_curve``: takes an $\mathbb{F}_p$ element and maps it to $E_p$.

[cfrg-hash-to-curve]: https://datatracker.ietf.org/doc/draft-irtf-cfrg-hash-to-curve/?include_text=1

### TODO: Simplified SWU
Reference: https://eprint.iacr.org/2019/403.pdf

## References
[^complete-formulae]: [Renes, J., Costello, C., & Batina, L. (2016, May). "Complete addition formulas for prime order elliptic curves." In Annual International Conference on the Theory and Applications of Cryptographic Techniques (pp. 403-428). Springer, Berlin, Heidelberg.](https://eprint.iacr.org/2015/1060)


# 椭圆曲线

在有限域上构造的椭圆曲线是另一种重要的密码学工具。

我们使用椭圆曲线是因为它们提供了一个密码学意义上的[群](fields.md#Groups)，即离散对数问题（下文讨论）难以解决的群。

有几种定义椭圆曲线方程的方式，但就我们的目的而言，令 $\mathbb{F}_p$ 为一个大的（255位）有限域，满足方程 $y^2 = x^3 + b$（其中 $b$ 为常数）的解集 $(x, y)$ 定义了椭圆曲线 $E(\mathbb{F}_p)$ 上的 $\mathbb{F}_p$-有理点。这些 $(x, y)$ 坐标被称为"仿射坐标"。每个 $\mathbb{F}_p$-有理点与作为群单位元的"无穷远点" $\mathcal{O}$ 共同构成群的元素。按照惯例，椭圆曲线群采用加法表示。

![](https://i.imgur.com/JvLS6yE.png)
*"同一直线上的三个点之和为零，即无穷远点。"*

群加法法则简单直观：要将两个点相加，找到同时经过这两个点的直线并求得第三个交点，然后取其 $y$ 坐标的相反数。当点与自身相加时（称为**点加倍**），需要特殊处理：找到该点处的切线，再求切线的另一交点并取反。若要将点与其逆元相加，则结果是无穷远点。

通过点的加法与加倍运算，我们自然可以将其与整数（称为**标量**）相乘。曲线上的点数量即为群的阶。如果这个数是质数 $q$，那么标量可以视为**标量域** $\mathbb{F}_q$ 的元素。

当椭圆曲线被正确设计时，具有重要的安全特性：给定两个随机元素 $G, H \in E(\mathbb{F}_p)$，找到满足 $[a]G = H$ 的 $a$（即 $H$ 关于 $G$ 的离散对数）在经典计算机上被认为是计算不可行的。这称为**椭圆曲线离散对数假设**。

如果椭圆曲线群 $\mathbb{G}$ 具有质数阶 $q$（如 Halo 2 中使用的曲线），那么它是有限循环群。根据[群论章节](fields.md#Groups)可知，这意味着它与 $\mathbb{Z}/q\mathbb{Z}$ 同构，或者说与标量域 $\mathbb{F}_q$ 同构。每个可能的生成元 $G$ 固定了这个同构关系，此时标量侧的元素正是对应群元素关于 $G$ 的离散对数。对于密码学安全的椭圆曲线，$\mathbb{G} \rightarrow \mathbb{F}_q$ 方向的同构计算是困难的，因为椭圆曲线离散对数问题是难解的。

> 有时通过标量而非群元素来思考基于群的密码协议和算法是有帮助的，这可以简化证明和符号表示。
> 
> 例如，在证明系统论文中使用符号 $[x]$ 表示具有离散对数 $x$ 的群元素（生成元隐式指定）已成为惯例。
> 
> 我们在"[distinct-x 定理](https://zips.z.cash/protocol/protocol.pdf#thmdistinctx)"中也应用了这个思想，以证明 Sapling 中[椭圆曲线标量乘法优化](https://github.com/zcash/zcash/issues/3924)的正确性，以及原始 [Halo 论文](https://eprint.iacr.org/2019/1021.pdf)附录 C 中基于自同态的优化。

## 曲线算术

### 点加倍

最简单的情况是加倍点 $(x_0, y_0)$。以我们的示例曲线 $y^2 = x^3 + b$ 为例，首先计算导数：
$$
\lambda = \frac{\mathrm{d}y}{\mathrm{d}x} = \frac{3x^2}{2y}
$$

要得到 $(x_1, y_1) = (x_0, y_0) + (x_0, y_0)$ 的表达式，考虑：

$$
\begin{aligned}
\frac{-y_1 - y_0}{x_1 - x_0} = \lambda &\implies -y_1 = \lambda(x_1 - x_0) + y_0 \\
&\implies \boxed{y_1 = \lambda(x_0 - x_1) - y_0}
\end{aligned}
$$

将 $y = \lambda(x_0 - x) - y_0$ 代入椭圆曲线方程求 $x_1$：

$$
\begin{aligned}
y^2 = x^3 + b &\implies (\lambda(x_0 - x) - y_0)^2 = x^3 + b \\
&\implies x^3 - \lambda^2 x^2 + \cdots = 0 \leftarrow\text{（整理项）} \\
&= (x - x_0)(x - x_0)(x - x_1) \leftarrow\text{（已知根 $x_0, x_0, x_1$）} \\
&= x^3 - (x_0 + x_0 + x_1)x^2 + \cdots
\end{aligned}
$$

比较 $x^2$ 项的系数得：
$\lambda^2 = x_0 + x_0 + x_1 \implies \boxed{x_1 = \lambda^2 - 2x_0}$

### 投影坐标

上述方法需要计算代价高昂的 $2y$ 的逆。我们可以通过重新排列方程来"延迟"逆计算，因为在单个曲线操作后通常不需要立即得到结果点的实际仿射坐标 $(x', y')$。引入第三个坐标 $Z$，将曲线方程缩放为：

$$
Z^3 y^2 = Z^3 x^3 + Z^3 b
$$

当 $Z = 1$ 时即为原曲线。若允许仿射点 $(x, y)$ 表示为 $X = xZ$, $Y = yZ$（$Z \neq 0$），则得到[齐次投影曲线](https://en.wikipedia.org/wiki/Homogeneous_coordinates)：

$$
Y^2 Z = X^3 + Z^3 b
$$

当 $Z \neq 0$ 时，从 $(X, Y, Z)$ 获取 $(x, y)$ 只需计算 $(X/Z, Y/Z)$。（当 $Z = 0$ 时对应无穷远点 $O := (0:1:0)$）。这种方法可以延迟点加倍所需的逆运算，策略是将 $x', y'$ 表示为有理函数，重组分母后取 $(X, Y, Z)$ 使得 $Z$ 为公共分母，$X = x'Z$, $Y = y'Z$。

> 投影坐标通常（但非总是）比仿射坐标更高效。例外情况包括：使用不同的蒙哥马利技巧，或在乘法与逆运算代价相当的电路环境中。

以下展示无逆运算的点加倍 $(X, Y, Z) = (xZ, yZ, Z)$ 的示例：

$$
\lambda = \frac{3x^2}{2y} = \frac{3(X/Z)^2}{2(Y/Z)} = \frac{3 X^2}{2YZ}
$$

得到：

$$
\begin{aligned}
x' &= \lambda^2 - 2x \\
&= \lambda^2 - \frac{2X}{Z} \\
&= \frac{9 X^4}{4Y^2Z^2} - \frac{2X}{Z} \\
&= \frac{9 X^4 - 8XY^2Z}{4Y^2Z^2} \\
&= \frac{18 X^4 Y Z - 16XY^3Z^2}{8Y^3Z^3} \\
\\
y' &= \lambda (x - x') - y \\
&= \lambda (\frac{X}{Z} - \frac{9 X^4 - 8XY^2Z}{4Y^2Z^2}) - \frac{Y}{Z} \\
&= \frac{3 X^2}{2YZ} (\frac{X}{Z} - \frac{9 X^4 - 8XY^2Z}{4Y^2Z^2}) - \frac{Y}{Z} \\
&= \frac{3 X^3}{2YZ^2} - \frac{27 X^6 - 24X^3Y^2Z}{8Y^3Z^3} - \frac{Y}{Z} \\
&= \frac{12 X^3Y^2Z - 8Y^4Z^2 - 27 X^6 + 24X^3Y^2Z}{8Y^3Z^3}
\end{aligned}
$$

注意到 $x'$ 和 $y'$ 的分母相同。因此，我们可以计算 $(X, Y, Z)$（其中 $Z = 8Y^3Z^3$，$X$ 和 $Y$ 设为对应分子）来代替 $(x', y')$，从而完全避免逆运算。类似方法可用于两个不同点的加法。

### 点加法

现考虑不同横坐标点的加法：$P = (x_0, y_0)$ 和 $Q = (x_1, y_1)$（$x_0 \neq x_1$），求 $R = P + Q = (x_2, y_2)$。直线 $\overline{PQ}$ 的斜率为：

$$\lambda = \frac{y_1 - y_0}{x_1 - x_0} \implies y - y_0 = \lambda \cdot (x - x_0)$$

计算 $-R$ 的 $y$-坐标 $-y_2$：

$$-y_2 - y_0 = \lambda \cdot (x_2 - x_0) \implies \boxed{y_2 = \lambda (x_0 - x_2) - y_0}$$

将 $\overline{PQ}$ 的表达式代入曲线方程：

$$
\begin{aligned}
y^2 = x^3 + b &\implies (\lambda(x - x_0) + y_0)^2 = x^3 + b \\
&\implies x^3 - \lambda^2 x^2 + \cdots = 0 \leftarrow\text{（整理项）} \\
&= (x - x_0)(x - x_1)(x - x_2) \leftarrow\text{（已知根 $x_0, x_1, x_2$）} \\
&= x^3 - (x_0 + x_1 + x_2)x^2 + \cdots
\end{aligned}
$$

比较 $x^2$ 项的系数得：
$\lambda^2 = x_0 + x_1 + x_2 \implies \boxed{x_2 = \lambda^2 - x_0 - x_1}$

## 重要说明

* 存在无边界情况（称为"完整公式"）的点加法高效公式[^complete-formulae]，这类公式统一了加法和加倍操作。使用这些公式将点与其逆元相加会产生 $Z = 0$，表示无穷远点
* 另有雅可比表示法 $(x, y) = (xZ^2, yZ^3, Z)$，其中曲线按 $Z^6$ 缩放。这种表示法运算效率更高，但缺乏统一公式
* 在齐次投影坐标系中，通过"齐次化"Z坐标可轻松比较曲线点：$X_1 Z_2 = X_2 Z_1$ 且 $Y_1 Z_2 = Y_2 Z_1$

## 曲线自同态

假设 $\mathbb{F}_p$ 包含本原单位立方根，即 $3 | p - 1$，存在元素 $\zeta_p$ 生成3阶乘法子群。对于示例曲线 $y^2 = x^3 + b$ 上的点 $(x, y)$，其关联点为 $(\zeta_p x,y)$ 和 $(\zeta_p^2 x,y)$，因为 $x^3$ 运算会消除 $\zeta$ 分量。映射 $(x, y) \mapsto (\zeta_p x, y)$ 是曲线的自同态应用。当曲线具有素数阶 $q$ 时，该自同态效果相当于用标量域中的本原立方根 $\zeta_q$ 乘以标量。

## 曲线点压缩
对于曲线点 $P = (x,y)$，其逆元 $-P = (x, -y)$ 也在曲线上。只需编码x坐标和y坐标符号即可唯一标识点。

### 序列化
如[字段](./fields.md)章节所述，用字段元素最低有效位（LSB）表示"符号"。将y坐标的LSB作为`sign`位打包到x坐标的最高位：

```text
         <----------------------------------- x --------------------------------->
Enc(P) = [_ _ _ _ _ _ _ _] [_ _ _ _ _ _ _ _] ... [_ _ _ _ _ _ _ _] [_ _ _ _ _ _ _ sign]
          ^                <------------------------------------->                 ^
         LSB                              30 bytes                                MSB
```
作为群单位元的"无穷远点" $\mathcal{O}$ 没有仿射坐标 $(x, y)$ 表示。然而，在 Pallas 和 Vesta 曲线上都不存在 $x = 0$ 或 $y = 0$ 的点。因此，我们使用"伪仿射坐标" $(0, 0)$ 来编码 $\mathcal{O}$，这会生成全零的 32 字节数组。

### 反序列化

当反序列化压缩的曲线点时，我们首先将最高有效位读取为 $y$ 坐标的符号位 `ysign`，然后将该位清零以恢复原始 $x$ 坐标。

如果 $x = 0, y = 0$，我们返回无穷远点 $\mathcal{O}$。否则，我们计算 $y = \sqrt{x^3 + b}$。此时，我们读取 $y$ 的最低有效位作为 `sign`。如果 `sign == ysign`，则直接返回曲线点 $(x, y)$；否则，取反 $y$ 并返回 $(x, -y)$。

## 曲线循环

设 $E_p$ 为定义在有限域 $\mathbb{F}_p$ 上的椭圆曲线（其中 $p$ 为素数），记作 $E_p/\mathbb{F}_p$。我们用 $q = \#E(\mathbb{F}_p)$ 表示该曲线的点群阶，其中：
- $\mathbb{F}_p$ 称为**基域**
- $\mathbb{F}_q$ 称为**标量域**

我们在椭圆曲线 $E_p/\mathbb{F}_p$ 上实例化证明系统，这使得我们可以证明 $\mathbb{F}_q$-算术电路的可满足性。

> **​（注）为什么基域为 $\mathbb{F}_p$ 的曲线对应 $\mathbb{F}_q$ 的算术电路？**  
> 证明系统本质处理的是电路中标量的编码（更准确地说，是对多项式系数的承诺）。当编码/承诺为 $E_p/\mathbb{F}_p$ 的椭圆曲线点时，标量自然属于 $\mathbb{F}_q$。

然而，验证者的大部分算术计算仍在基域 $\mathbb{F}_p$ 上进行，因此可高效表示为 $\mathbb{F}_p$-算术电路。

> **​（注）为什么验证者的计算主要在 $\mathbb{F}_p$ 上？**  
> Halo 2 验证器需要执行基于电路输出信息的群操作。点加倍和加法等群操作使用 $\mathbb{F}_p$ 的算术，因为点的坐标属于 $\mathbb{F}_p$。

这促使我们构建另一条标量域为 $\mathbb{F}_p$ 的曲线，其 $\mathbb{F}_p$-算术电路可高效验证第一条曲线的证明。若第二条曲线的基域为 $E_q/\mathbb{F}_q$，其生成的证明也可被第一条曲线的 $\mathbb{F}_q$-算术电路高效验证。即我们在 $E_q/\mathbb{F}_q$ 上实例化第二个证明系统，形成双向循环：

![](https://i.imgur.com/bNMyMRu.png)

### TODO: Pallas-Vesta 曲线
参考实现：https://github.com/zcash/pasta

## 哈希到曲线

有时需要将输入确定性地映射到椭圆曲线 $E_p/\mathbb{F}_p$ 上的随机点，且无人知晓其离散对数（相对于其他基点）。

[哈希到椭圆曲线的互联网草案][cfrg-hash-to-curve] 详细描述了此过程，根据效率和安全需求可选择不同算法。草案框架使用以下函数：
- `hash_to_field`：将字节序列映射到基域 $\mathbb{F}_p$ 元素
- `map_to_curve`：将 $\mathbb{F}_p$ 元素映射到曲线 $E_p$

[cfrg-hash-to-curve]: https://datatracker.ietf.org/doc/draft-irtf-cfrg-hash-to-curve/?include_text=1

### TODO: 简化 SWU
参考论文：https://eprint.iacr.org/2019/403.pdf

## 参考文献
[^complete-formulae]: [Renes, J., Costello, C., & Batina, L. (2016).《素数阶椭圆曲线的完整加法公式》，欧密会论文集，Springer, pp. 403-428](https://eprint.iacr.org/2015/1060)