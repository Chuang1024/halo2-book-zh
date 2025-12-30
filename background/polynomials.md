# Polynomials

Let $A(X)$ be a polynomial over $\mathbb{F}_p$ with formal indeterminate $X$. As an example,

$$
A(X) = a_0 + a_1 X + a_2 X^2 + a_3 X^3
$$

defines a degree-$3$ polynomial. $a_0$ is referred to as the constant term. Polynomials of
degree $n-1$ have $n$ coefficients. We will often want to compute the result of replacing
the formal indeterminate $X$ with some concrete value $x$, which we denote by $A(x)$.

> In mathematics this is commonly referred to as "evaluating $A(X)$ at a point $x$".
> The word "point" here stems from the geometrical usage of polynomials in the form
> $y = A(x)$, where $(x, y)$ is the coordinate of a point in two-dimensional space.
> However, the polynomials we deal with are almost always constrained to equal zero, and
> $x$ will be an [element of some field](fields.md). This should not be confused
> with points on an [elliptic curve](curves.md), which we also make use of, but never in
> the context of polynomial evaluation.

Important notes:

* Multiplication of polynomials produces a product polynomial that is the sum of the
  degrees of its factors. Polynomial division subtracts from the degree.
  $$\deg(A(X)B(X)) = \deg(A(X)) + \deg(B(X)),$$
  $$\deg(A(X)/B(X)) = \deg(A(X)) -\deg(B(X)).$$
* Given a polynomial $A(X)$ of degree $n-1$, if we obtain $n$ evaluations of the
  polynomial at distinct points then these evaluations perfectly define the polynomial. In
  other words, given these evaluations we can obtain a unique polynomial $A(X)$ of degree
  $n-1$ via polynomial interpolation.
* $[a_0, a_1, \cdots, a_{n-1}]$ is the **coefficient representation** of the polynomial
  $A(X)$. Equivalently, we could use its **evaluation representation**
  $$[(x_0, A(x_0)), (x_1, A(x_1)), \cdots, (x_{n-1}, A(x_{n-1}))]$$
  at $n$ distinct points. Either representation uniquely specifies the same polynomial.

> #### (aside) Horner's rule
> Horner's rule allows for efficient evaluation of a polynomial of degree $n-1$, using
> only $n-1$ multiplications and $n-1$ additions. It is the following identity:
> $$\begin{aligned}a_0 &+ a_1X + a_2X^2 + \cdots + a_{n-1}X^{n-1} \\ &= a_0 + X\bigg( a_1 + X \Big( a_2 + \cdots + X(a_{n-2} + X a_{n-1}) \Big)\!\bigg),\end{aligned}$$

## Fast Fourier Transform (FFT)
The FFT is an efficient way of converting between the coefficient and evaluation
representations of a polynomial. It evaluates the polynomial at the $n$th roots of unity
$\{\omega^0, \omega^1, \cdots, \omega^{n-1}\},$ where $\omega$ is a primitive $n$th root
of unity. By exploiting symmetries in the roots of unity, each round of the FFT reduces
the evaluation into a problem only half the size. Most commonly we use polynomials of
length some power of two, $n = 2^k$, and apply the halving reduction recursively.

### Motivation: Fast polynomial multiplication
In the coefficient representation, it takes $O(n^2)$ operations to multiply two
polynomials $A(X)\cdot B(X) = C(X)$:

$$
\begin{aligned}
A(X) &= a_0 + a_1X + a_2X^2 + \cdots + a_{n-1}X^{n-1}, \\
B(X) &= b_0 + b_1X + b_2X^2 + \cdots + b_{n-1}X^{n-1}, \\
C(X) &= a_0\cdot (b_0 + b_1X + b_2X^2 + \cdots + b_{n-1}X^{n-1}) \\
&+ a_1X\cdot (b_0 + b_1X + b_2X^2 + \cdots + b_{n-1}X^{n-1})\\
&+ \cdots \\
&+ a_{n-1}X^{n-1} \cdot (b_0 + b_1X + b_2X^2 + \cdots + b_{n-1}X^{n-1}),
\end{aligned}
$$

where each of the $n$ terms in the first polynomial has to be multiplied by the $n$ terms
of the second polynomial.

In the evaluation representation, however, polynomial multiplication only requires $O(n)$
operations:

$$
\begin{aligned}
A&: \{(x_0, A(x_0)), (x_1, A(x_1)), \cdots, (x_{n-1}, A(x_{n-1}))\}, \\
B&: \{(x_0, B(x_0)), (x_1, B(x_1)), \cdots, (x_{n-1}, B(x_{n-1}))\}, \\
C&: \{(x_0, A(x_0)B(x_0)), (x_1, A(x_1)B(x_1)), \cdots, (x_{n-1}, A(x_{n-1})B(x_{n-1}))\},
\end{aligned}
$$

where each evaluation is multiplied pointwise.

This suggests the following strategy for fast polynomial multiplication:

1. Evaluate polynomials at all $n$ points;
2. Perform fast pointwise multiplication in the evaluation representation ($O(n)$);
3. Convert back to the coefficient representation.

The challenge now is how to **evaluate** and **interpolate** the polynomials efficiently.
Naively, evaluating a polynomial at $n$ points would require $O(n^2)$ operations (we use
the $O(n)$ Horner's rule at each point):

$$
\begin{bmatrix}
A(1) \\
A(\omega) \\
A(\omega^2) \\
\vdots \\
A(\omega^{n-1})
\end{bmatrix} =
\begin{bmatrix}
1&1&1&\dots&1 \\
1&\omega&\omega^2&\dots&\omega^{n-1} \\
1&\omega^2&\omega^{2\cdot2}&\dots&\omega^{2\cdot(n-1)} \\
\vdots&\vdots&\vdots& &\vdots \\
1&\omega^{n-1}&\omega^{2(n-1)}&\cdots&\omega^{(n-1)^2}\\
\end{bmatrix} \cdot
\begin{bmatrix}
a_0 \\
a_1 \\
a_2 \\
\vdots \\
a_{n-1}
\end{bmatrix}.
$$

For convenience, we will denote the matrices above as:
$$\hat{\mathbf{A}} = \mathbf{V}_\omega \cdot \mathbf{A}. $$

($\hat{\mathbf{A}}$ is known as the *Discrete Fourier Transform* of $\mathbf{A}$;
$\mathbf{V}_\omega$ is also called the *Vandermonde matrix*.)

### The (radix-2) Cooley-Tukey algorithm
Our strategy is to divide a DFT of size $n$ into two interleaved DFTs of size $n/2$. Given
the polynomial $A(X) = a_0 + a_1X + a_2X^2 + \cdots + a_{n-1}X^{n-1},$ we split it up into
even and odd terms:

$$
\begin{aligned}
A_{\text{even}} &= a_0 + a_2X + \cdots + a_{n-2}X^{\frac{n}{2} - 1}, \\
A_{\text{odd}} &= a_1 + a_3X + \cdots + a_{n-1}X^{\frac{n}{2} - 1}. \\
\end{aligned}
$$

To recover the original polynomial, we do
$A(X) = A_{\text{even}} (X^2) + X A_{\text{odd}}(X^2).$

Trying this out on points $\omega_n^i$ and $\omega_n^{\frac{n}{2} + i}$,
$i \in [0..\frac{n}{2}-1],$ we start to notice some symmetries:

$$
\begin{aligned}
A(\omega_n^i) &= A_{\text{even}} ((\omega_n^i)^2) + \omega_n^i A_{\text{odd}}((\omega_n^i)^2), \\
A(\omega_n^{\frac{n}{2} + i}) &= A_{\text{even}} ((\omega_n^{\frac{n}{2} + i})^2) + \omega_n^{\frac{n}{2} + i} A_{\text{odd}}((\omega_n^{\frac{n}{2} + i})^2) \\
&= A_{\text{even}} ((-\omega_n^i)^2) - \omega_n^i A_{\text{odd}}((-\omega_n^i)^2) \leftarrow\text{(negation lemma)} \\
&= A_{\text{even}} ((\omega_n^i)^2) - \omega_n^i A_{\text{odd}}((\omega_n^i)^2).
\end{aligned}
$$

Notice that we are only evaluating $A_{\text{even}}(X)$ and $A_{\text{odd}}(X)$ over half
the domain $\{(\omega_n^0)^2, (\omega_n)^2, \cdots, (\omega_n^{\frac{n}{2} -1})^2\} = \{\omega_{n/2}^i\}, i = [0..\frac{n}{2}-1]$ (halving lemma).
This gives us all the terms we need to reconstruct $A(X)$ over the full domain
$\{\omega^0, \omega, \cdots, \omega^{n -1}\}$: which means we have transformed a
length-$n$ DFT into two length-$\frac{n}{2}$ DFTs. 

We choose $n = 2^k$ to be a power of two (by zero-padding if needed), and apply this
divide-and-conquer strategy recursively. By the Master Theorem[^master-thm], this gives us
an evaluation algorithm with $O(n\log_2n)$ operations, also known as the Fast Fourier
Transform (FFT).

### Inverse FFT
So we've evaluated our polynomials and multiplied them pointwise. What remains is to
convert the product from the evaluation representation back to coefficient representation.
To do this, we simply call the FFT on the evaluation representation. However, this time we
also:
- replace $\omega^i$ by $\omega^{-i}$ in the Vandermonde matrix, and
- multiply our final result by a factor of $1/n$.

In other words:
$$\mathbf{A} = \frac{1}{n} \mathbf{V}_{\omega^{-1}} \cdot \hat{\mathbf{A}}. $$

(To understand why the inverse FFT has a similar form to the FFT, refer to Slide 13-1 of
[^ifft]. The below image was also taken from [^ifft].)

![](https://i.imgur.com/lSw30zo.png)


## The Schwartz-Zippel lemma
The Schwartz-Zippel lemma informally states that "different polynomials are different at
most points." Formally, it can be written as follows:

> Let $p(x_1, x_2, \cdots, x_n)$ be a nonzero polynomial of $n$ variables with degree $d$.
> Let $S$ be a finite set of numbers with at least $d$ elements in it. If we choose random
> $\alpha_1, \alpha_2, \cdots, \alpha_n$ from $S$,
> $$\text{Pr}[p(\alpha_1, \alpha_2, \cdots, \alpha_n) = 0] \leq \frac{d}{|S|}.$$

In the familiar univariate case $p(X)$, this reduces to saying that a nonzero polynomial
of degree $d$ has at most $d$ roots.

The Schwartz-Zippel lemma is used in polynomial equality testing.  Given two multi-variate
polynomials $p_1(x_1,\cdots,x_n)$ and $p_2(x_1,\cdots,x_n)$ of degrees $d_1, d_2$
respectively, we can test if
$p_1(\alpha_1, \cdots, \alpha_n) - p_2(\alpha_1, \cdots, \alpha_n) = 0$ for random
$\alpha_1, \cdots, \alpha_n \leftarrow S,$ where the size of $S$ is at least
$|S| \geq (d_1 + d_2).$  If the two polynomials are identical, this will always be true,
whereas if the two polynomials are different then the equality holds with probability at
most $\frac{\max(d_1,d_2)}{|S|}$.

## Vanishing polynomial
Consider the order-$n$ multiplicative subgroup $\mathcal{H}$ with primitive root of unity
$\omega$. For all $\omega^i \in \mathcal{H}, i \in [n-1],$ we have
$(\omega^i)^n = (\omega^n)^i = (\omega^0)^i = 1.$ In other words, every element of
$\mathcal{H}$ fulfils the equation 

$$
\begin{aligned}
Z_H(X) &= X^n - 1 \\
&= (X-\omega^0)(X-\omega^1)(X-\omega^2)\cdots(X-\omega^{n-1}),
\end{aligned}
$$

meaning every element is a root of $Z_H(X).$ We call $Z_H(X)$ the **vanishing polynomial**
over $\mathcal{H}$ because it evaluates to zero on all elements of $\mathcal{H}.$

This comes in particularly handy when checking polynomial constraints. For instance, to
check that $A(X) + B(X) = C(X)$ over $\mathcal{H},$ we simply have to check that
$A(X) + B(X) - C(X)$ is some multiple of $Z_H(X)$. In other words, if dividing our
constraint by the vanishing polynomial still yields some polynomial
$\frac{A(X) + B(X) - C(X)}{Z_H(X)} = H(X),$ we are satisfied that $A(X) + B(X) - C(X) = 0$
over $\mathcal{H}.$

## Lagrange basis functions

> TODO: explain what a basis is in general (briefly).

Polynomials are commonly written in the monomial basis (e.g. $X, X^2, ... X^n$). However,
when working over a multiplicative subgroup of order $n$, we find a more natural expression
in the Lagrange basis.

Consider the order-$n$ multiplicative subgroup $\mathcal{H}$ with primitive root of unity
$\omega$. The Lagrange basis corresponding to this subgroup is a set of functions
$\{\mathcal{L}_i\}_{i = 0}^{n-1}$, where 

$$
\mathcal{L_i}(\omega^j) = \begin{cases}
1 & \text{if } i = j, \\
0 & \text{otherwise.}
\end{cases}
$$

We can write this more compactly as $\mathcal{L_i}(\omega^j) = \delta_{ij},$ where
$\delta$ is the Kronecker delta function. 

Now, we can write our polynomial as a linear combination of Lagrange basis functions,

$$A(X) = \sum_{i = 0}^{n-1} a_i\mathcal{L_i}(X), X \in \mathcal{H},$$

which is equivalent to saying that $A(X)$ evaluates to $a_0$ at $\omega^0$,
to $a_1$ at $\omega^1$, to $a_2$ at $\omega^2, \cdots,$ and so on.

When working over a multiplicative subgroup, the Lagrange basis function has a convenient
sparse representation of the form

$$
\mathcal{L}_i(X) = \frac{c_i\cdot(X^{n} - 1)}{X - \omega^i},
$$

where $c_i$ is the barycentric weight. (To understand how this form was derived, refer to
[^barycentric].) For $i = 0,$ we have
$c = 1/n \implies \mathcal{L}_0(X) = \frac{1}{n} \frac{(X^{n} - 1)}{X - 1}$.

Suppose we are given a set of evaluation points $\{x_0, x_1, \cdots, x_{n-1}\}$.
Since we cannot assume that the $x_i$'s form a multiplicative subgroup, we consider also
the Lagrange polynomials $\mathcal{L}_i$'s in the general case. Then we can construct:

$$
\mathcal{L}_i(X) = \prod_{j\neq i}\frac{X - x_j}{x_i - x_j}, i \in [0..n-1].
$$

Here, every $X = x_j \neq x_i$ will produce a zero numerator term $(x_j - x_j),$ causing
the whole product to evaluate to zero. On the other hand, $X= x_i$ will evaluate to
$\frac{x_i - x_j}{x_i - x_j}$ at every term, resulting in an overall product of one. This
gives the desired Kronecker delta behaviour $\mathcal{L_i}(x_j) = \delta_{ij}$ on the
set $\{x_0, x_1, \cdots, x_{n-1}\}$.

### Lagrange interpolation
Given a polynomial in its evaluation representation

$$A: \{(x_0, A(x_0)), (x_1, A(x_1)), \cdots, (x_{n-1}, A(x_{n-1}))\},$$

we can reconstruct its coefficient form in the Lagrange basis:

$$A(X) = \sum_{i = 0}^{n-1} A(x_i)\mathcal{L_i}(X), $$

where $X \in \{x_0, x_1,\cdots, x_{n-1}\}.$

## References
[^master-thm]: [Dasgupta, S., Papadimitriou, C. H., & Vazirani, U. V. (2008). "Algorithms" (ch. 2). New York: McGraw-Hill Higher Education.](https://people.eecs.berkeley.edu/~vazirani/algorithms/chap2.pdf)

[^ifft]: [Golin, M. (2016). "The Fast Fourier Transform and Polynomial Multiplication" [lecture notes], COMP 3711H Design and Analysis of Algorithms, Hong Kong University of Science and Technology.](http://www.cs.ust.hk/mjg_lib/Classes/COMP3711H_Fall16/lectures/FFT_Slides.pdf)

[^barycentric]: [Berrut, J. and Trefethen, L. (2004). "Barycentric Lagrange Interpolation."](https://people.maths.ox.ac.uk/trefethen/barycentric.pdf)


# 多项式

设 $A(X)$ 是定义在 $\mathbb{F}_p$ 上的多项式，其中 $X$ 是形式未知数。例如，

$$
A(X) = a_0 + a_1 X + a_2 X^2 + a_3 X^3
$$

定义了一个 3 次多项式。$a_0$ 称为常数项。$n-1$ 次多项式有 $n$ 个系数。我们经常需要将形式未知数 $X$ 替换为某个具体值 $x$，这被称为 $A(x)$。

> 在数学中，这通常被称为“在点 $x$ 处评估 $A(X)$”。这里的“点”源于多项式在几何中的使用形式 $y = A(x)$，其中 $(x, y)$ 是二维空间中的一个点的坐标。然而，我们处理的多项式几乎总是被约束为零，且 $x$ 是[某个域的元素](fields.md)。这不应与[椭圆曲线](curves.md)上的点混淆，尽管我们也会使用椭圆曲线，但不会在多项式评估的上下文中使用。

重要说明：

* 多项式的乘法产生一个乘积多项式，其次数是其因子次数的和。多项式除法则会减少次数。
  $$\deg(A(X)B(X)) = \deg(A(X)) + \deg(B(X)),$$
  $$\deg(A(X)/B(X)) = \deg(A(X)) -\deg(B(X)).$$
* 给定一个次数为 $n-1$ 的多项式 $A(X)$，如果我们在 $n$ 个不同的点处获得多项式的评估值，那么这些评估值完美地定义了该多项式。换句话说，给定这些评估值，我们可以通过多项式插值获得唯一的 $n-1$ 次多项式 $A(X)$。
* $[a_0, a_1, \cdots, a_{n-1}]$ 是多项式 $A(X)$ 的**系数表示**。等效地，我们可以使用其**评估表示**
  $$[(x_0, A(x_0)), (x_1, A(x_1)), \cdots, (x_{n-1, A(x_{n-1})}]$$
  在 $n$ 个不同的点处。这两种表示唯一地指定了同一个多项式。

> #### （旁注）霍纳法则
> 霍纳法则允许高效地评估一个次数为 $n-1$ 的多项式，仅使用 $n-1$ 次乘法和 $n-1$ 次加法。它是以下恒等式：
> $$\begin{aligned}a_0 &+ a_1X + a_2X^2 + \cdots + a_{n-1}X^{n-1} \\ &= a_0 + X\bigg( a_1 + X \Big( a_2 + \cdots + X(a_{n-2} + X a_{n-1}) \Big)\!\bigg),\end{aligned}$$

## 快速傅里叶变换 (FFT)
FFT 是一种高效地在多项式的系数表示和评估表示之间进行转换的方法。它在 $n$ 次单位根 $\{\omega^0, \omega^1, \cdots, \omega^{n-1}\}$ 处评估多项式，其中 $\omega$ 是一个 $n$ 次本原单位根。通过利用单位根的对称性，FFT 的每一轮将评估问题简化为原来的一半大小。最常用的是长度为 2 的幂次的多项式，即 $n = 2^k$，并递归地应用这种减半策略。

### 动机：快速多项式乘法
在系数表示中，将两个多项式 $A(X)\cdot B(X) = C(X)$ 相乘需要 $O(n^2)$ 次操作：

$$
\begin{aligned}
A(X) &= a_0 + a_1X + a_2X^2 + \cdots + a_{n-1}X^{n-1}, \\
B(X) &= b_0 + b_1X + b_2X^2 + \cdots + b_{n-1}X^{n-1}, \\
C(X) &= a_0\cdot (b_0 + b_1X + b_2X^2 + \cdots + b_{n-1}X^{n-1}) \\
&+ a_1X\cdot (b_0 + b_1X + b_2X^2 + \cdots + b_{n-1}X^{n-1})\\
&+ \cdots \\
&+ a_{n-1}X^{n-1} \cdot (b_0 + b_1X + b_2X^2 + \cdots + b_{n-1}X^{n-1}),
\end{aligned}
$$

其中第一个多项式的每一项都需要与第二个多项式的每一项相乘。

在评估表示中，多项式乘法仅需要 $O(n)$ 次操作：

$$
\begin{aligned}
A&: \{(x_0, A(x_0)), (x_1, A(x_1)), \cdots, (x_{n-1}, A(x_{n-1}))\}, \\
B&: \{(x_0, B(x_0)), (x_1, B(x_1)), \cdots, (x_{n-1}, B(x_{n-1}))\}, \\
C&: \{(x_0, A(x_0)B(x_0)), (x_1, A(x_1)B(x_1)), \cdots, (x_{n-1}, A(x_{n-1})B(x_{n-1}))\},
\end{aligned}
$$

其中每个评估值都是逐点相乘的。

这提出了以下快速多项式乘法的策略：

1. 在所有 $n$ 个点处评估多项式；
2. 在评估表示中执行快速的逐点乘法（$O(n)$）；
3. 转换回系数表示。

现在的挑战是如何高效地**评估**和**插值**多项式。朴素地在 $n$ 个点处评估多项式需要 $O(n^2)$ 次操作（我们在每个点处使用 $O(n)$ 的霍纳法则）：

$$
\begin{bmatrix}
A(1) \\
A(\omega) \\
A(\omega^2) \\
\vdots \\
A(\omega^{n-1})
\end{bmatrix} =
\begin{bmatrix}
1&1&1&\dots&1 \\
1&\omega&\omega^2&\dots&\omega^{n-1} \\
1&\omega^2&\omega^{2\cdot2}&\dots&\omega^{2\cdot(n-1)} \\
\vdots&\vdots&\vdots& &\vdots \\
1&\omega^{n-1}&\omega^{2(n-1)}&\cdots&\omega^{(n-1)^2}\\
\end{bmatrix} \cdot
\begin{bmatrix}
a_0 \\
a_1 \\
a_2 \\
\vdots \\
a_{n-1}
\end{bmatrix}.
$$

为了方便，我们将上述矩阵表示为：
$$\hat{\mathbf{A}} = \mathbf{V}_\omega \cdot \mathbf{A}. $$

（$\hat{\mathbf{A}}$ 被称为 $\mathbf{A}$ 的*离散傅里叶变换*；$\mathbf{V}_\omega$ 也被称为*范德蒙矩阵*。）

### (radix-2) Cooley-Tukey 算法
我们的策略是将大小为 $n$ 的 DFT 分成两个大小为 $n/2$ 的交错 DFT。给定多项式 $A(X) = a_0 + a_1X + a_2X^2 + \cdots + a_{n-1}X^{n-1}$，我们将其拆分为偶数和奇数项：

$$
\begin{aligned}
A_{\text{even}} &= a_0 + a_2X + \cdots + a_{n-2}X^{\frac{n}{2} - 1}, \\
A_{\text{odd}} &= a_1 + a_3X + \cdots + a_{n-1}X^{\frac{n}{2} - 1}. \\
\end{aligned}
$$

为了恢复原始多项式，我们执行 $A(X) = A_{\text{even}} (X^2) + X A_{\text{odd}}(X^2)$。

在点 $\omega_n^i$ 和 $\omega_n^{\frac{n}{2} + i}$ 处尝试这一点，$i \in [0..\frac{n}{2}-1]$，我们开始注意到一些对称性：

$$
\begin{aligned}
A(\omega_n^i) &= A_{\text{even}} ((\omega_n^i)^2) + \omega_n^i A_{\text{odd}}((\omega_n^i)^2), \\
A(\omega_n^{\frac{n}{2} + i}) &= A_{\text{even}} ((\omega_n^{\frac{n}{2} + i})^2) + \omega_n^{\frac{n}{2} + i} A_{\text{odd}}((\omega_n^{\frac{n}{2} + i})^2) \\
&= A_{\text{even}} ((-\omega_n^i)^2) - \omega_n^i A_{\text{odd}}((-\omega_n^i)^2) \leftarrow\text{(负元引理)} \\
&= A_{\text{even}} ((\omega_n^i)^2) - \omega_n^i A_{\text{odd}}((\omega_n^i)^2).
\end{aligned}
$$

注意到我们仅在域 $\{(\omega_n^0)^2, (\omega_n)^2, \cdots, (\omega_n^{\frac{n}{2} -1})^2\} = \{\omega_{n/2}^i\}, i = [0..\frac{n}{2}-1]$（折半引理）上评估 $A_{\text{even}}(X)$ 和 $A_{\text{odd}}(X)$。这为我们提供了重建 $A(X)$ 在完整域 $\{\omega^0, \omega, \cdots, \omega^{n -1}\}$ 上所需的所有项：这意味着我们将一个长度为 $n$ 的 DFT 转换为了两个长度为 $\frac{n}{2}$ 的 DFT。

我们选择 $n = 2^k$ 为 2 的幂（如有必要，通过零填充），并递归地应用这种分治策略。根据主定理[^master-thm]，这为我们提供了一个 $O(n\log_2n)$ 操作的评估算法，也称为快速傅里叶变换 (FFT)。

### 逆 FFT
我们已经评估了多项式并逐点相乘。剩下的就是将乘积从评估表示转换回系数表示。为此，我们只需在评估表示上调用 FFT。然而，这次我们还需要：
- 将 $\omega^i$ 替换为 $\omega^{-i}$ 在范德蒙矩阵中，并且
- 将最终结果乘以 $1/n$ 的因子。

换句话说：
$$\mathbf{A} = \frac{1}{n} \mathbf{V}_{\omega^{-1}} \cdot \hat{\mathbf{A}}. $$

（要理解为什么逆 FFT 的形式与 FFT 相似，请参阅 [^ifft] 的第 13-1 页。下图也取自 [^ifft]。）

![](https://i.imgur.com/lSw30zo.png)

## Schwartz-Zippel 引理
Schwartz-Zippel 引理非正式地指出“不同的多项式在大多数点上是不同的”。正式地，它可以写成如下形式：

> 设 $p(x_1, x_2, \cdots, x_n)$ 是一个 $n$ 个变量的非零多项式，次数为 $d$。设 $S$ 是一个有限集合，其中至少包含 $d$ 个元素。如果我们随机选择 $\alpha_1, \alpha_2, \cdots, \alpha_n$ 从 $S$ 中，
> $$\text{Pr}[p(\alpha_1, \alpha_2, \cdots, \alpha_n) = 0] \leq \frac{d}{|S|}.$$

在熟悉的单变量情况 $p(X)$ 中，这简化为说一个次数为 $d$ 的非零多项式最多有 $d$ 个根。

Schwartz-Zippel 引理用于多项式等式测试。给定两个多元多项式 $p_1(x_1,\cdots,x_n)$ 和 $p_2(x_1,\cdots,x_n)$，次数分别为 $d_1, d_2$，我们可以测试
$p_1(\alpha_1, \cdots, \alpha_n) - p_2(\alpha_1, \cdots, \alpha_n) = 0$ 对于随机 $\alpha_1, \cdots, \alpha_n \leftarrow S,$ 其中 $S$ 的大小至少为 $|S| \geq (d_1 + d_2)$。如果两个多项式相同，则这始终为真，而如果两个多项式不同，则等式成立的概率最多为 $\frac{\max(d_1,d_2)}{|S|}$。

## 消失多项式
考虑具有本原单位根 $\omega$ 的 $n$ 阶乘法子群 $\mathcal{H}$。对于所有 $\omega^i \in \mathcal{H}, i \in [n-1]$，我们有 $(\omega^i)^n = (\omega^n)^i = (\omega^0)^i = 1$。换句话说，$\mathcal{H}$ 的每个元素都满足方程

$$
\begin{aligned}
Z_H(X) &= X^n - 1 \\
&= (X-\omega^0)(X-\omega^1)(X-\omega^2)\cdots(X-\omega^{n-1}),
\end{aligned}
$$

这意味着每个元素都是 $Z_H(X)$ 的根。我们称 $Z_H(X)$ 为 $\mathcal{H}$ 上的**消失多项式**，因为它在 $\mathcal{H}$ 的所有元素上都评估为零。

这在检查多项式约束时特别有用。例如，要检查 $A(X) + B(X) = C(X)$ 在 $\mathcal{H}$ 上是否成立，我们只需检查 $A(X) + B(X) - C(X)$ 是 $Z_H(X)$ 的某个倍数。换句话说，如果我们的约束除以消失多项式仍然产生某个多项式 $\frac{A(X) + B(X) - C(X)}{Z_H(X)} = H(X)$，我们就满意 $A(X) + B(X) - C(X) = 0$ 在 $\mathcal{H}$ 上成立。

## 拉格朗日基函数

> TODO: 简要解释什么是基（一般情况下）。

多项式通常以单项式基（例如 $X, X^2, ... X^n$）书写。然而，当在一个 $n$ 阶乘法子群上工作时，我们发现拉格朗日基更自然。

考虑具有本原单位根 $\omega$ 的 $n$ 阶乘法子群 $\mathcal{H}$。对应于该子群的拉格朗日基是一组函数 $\{\mathcal{L}_i\}_{i = 0}^{n-1}$，其中

$$
\mathcal{L_i}(\omega^j) = \begin{cases}
1 & \text{如果 } i = j, \\
0 & \text{否则。}
\end{cases}
$$

我们可以更简洁地写成 $\mathcal{L_i}(\omega^j) = \delta_{ij}$，其中 $\delta$ 是克罗内克 delta 函数。

现在，我们可以将多项式写成拉格朗日基函数的线性组合，

$$A(X) = \sum_{i = 0}^{n-1} a_i\mathcal{L_i}(X), X \in \mathcal{H},$$

这等同于说 $A(X)$ 在 $\omega^0$ 处评估为 $a_0$，在 $\omega^1$ 处评估为 $a_1$，在 $\omega^2$ 处评估为 $a_2, \cdots,$ 以此类推。

当在乘法子群上工作时，拉格朗日基函数有一个方便的形式

$$
\mathcal{L}_i(X) = \frac{c_i\cdot(X^{n} - 1)}{X - \omega^i},
$$

其中 $c_i$ 是重心权重。（要了解如何推导这种形式，请参阅 [^barycentric]。）对于 $i = 0,$ 我们有
$c = 1/n \implies \mathcal{L}_0(X) = \frac{1}{n} \frac{(X^{n} - 1)}{X - 1}$。

假设我们有一组评估点 $\{x_0, x_1, \cdots, x_{n-1}\}$。由于我们不能假设 $x_i$ 形成一个乘法子群，我们也考虑一般情况下的拉格朗日多项式 $\mathcal{L}_i$。然后我们可以构造：

$$
\mathcal{L}_i(X) = \prod_{j\neq i}\frac{X - x_j}{x_i - x_j}, i \in [0..n-1].
$$

这里，每个 $X = x_j \neq x_i$ 都会产生一个零分子项 $(x_j - x_j)$，导致整个乘积评估为零。另一方面，$X= x_i$ 会在每一项处评估为 $\frac{x_i - x_j}{x_i - x_j}$，最终乘积为 1。这给出了在集合 $\{x_0, x_1, \cdots, x_{n-1}\}$ 上所需的克罗内克 delta 行为 $\mathcal{L_i}(x_j) = \delta_{ij}$。

### 拉格朗日插值
给定多项式的评估表示

$$A: \{(x_0, A(x_0)), (x_1, A(x_1)), \cdots, (x_{n-1}, A(x_{n-1}))\},$$

我们可以在拉格朗日基中重建其系数形式：

$$A(X) = \sum_{i = 0}^{n-1} A(x_i)\mathcal{L_i}(X), $$

其中 $X \in \{x_0, x_1,\cdots, x_{n-1}\}$。

## 参考文献
[^master-thm]: [Dasgupta, S., Papadimitriou, C. H., & Vazirani, U. V. (2008). "Algorithms" (ch. 2). New York: McGraw-Hill Higher Education.](https://people.eecs.berkeley.edu/~vazirani/algorithms/chap2.pdf)

[^ifft]: [Golin, M. (2016). "The Fast Fourier Transform and Polynomial Multiplication" [lecture notes], COMP 3711H Design and Analysis of Algorithms, Hong Kong University of Science and Technology.](http://www.cs.ust.hk/mjg_lib/Classes/COMP3711H_Fall16/lectures/FFT_Slides.pdf)

[^barycentric]: [Berrut, J. and Trefethen, L. (2004). "Barycentric Lagrange Interpolation."](https://people.maths.ox.ac.uk/trefethen/barycentric.pdf)