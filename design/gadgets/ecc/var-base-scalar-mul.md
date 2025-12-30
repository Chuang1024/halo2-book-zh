# Variable-base scalar multiplication

In the Orchard circuit we need to check $\mathsf{pk_d} = [\mathsf{ivk}] \mathsf{g_d}$ where $\mathsf{ivk} \in [0, p)$ and the scalar field is $\mathbb{F}_q$ with $p < q$.

We have $p = 2^{254} + t_p$ and $q = 2^{254} + t_q$, for $t_p, t_q < 2^{128}$.

## Witness scalar
We're trying to compute $[\alpha] T$ for $\alpha \in [0, q)$. Set $k = \alpha + t_q$ and $n = 254$. Then we can compute

$\begin{array}{cl}
[2^{254} + (\alpha + t_q)] T &= [2^{254} + (\alpha + t_q) - (2^{254} + t_q)] T \\
                             &= [\alpha] T
\end{array}$

provided that $\alpha + t_q \in [0, 2^{n+1})$, i.e. $\alpha < 2^{n+1} - t_q$ which covers the whole range we need because in fact $2^{255} - t_q > q$.

Thus, given a scalar $\alpha$, we witness the boolean decomposition of $k = \alpha + t_q.$ (We use big-endian bit order for convenient input into the variable-base scalar multiplication algorithm.)

$$k = k_{254} \cdot 2^{254} + k_{253} \cdot 2^{253} + \cdots + k_0.$$

## Variable-base scalar multiplication
We use an optimized double-and-add algorithm, copied from ["Faster variable-base scalar multiplication in zk-SNARK circuits"](https://github.com/zcash/zcash/issues/3924) with some variable name changes:

```ignore
Acc := [2] T
for i from n-1 down to 0 {
    P := k_{i+1} ? T : −T
    Acc := (Acc + P) + Acc
}
return (k_0 = 0) ? (Acc - T) : Acc
```

It remains to check that the x-coordinates of each pair of points to be added are distinct.

When adding points in a prime-order group, we can rely on Theorem 3 from Appendix C of the [Halo paper](https://eprint.iacr.org/2019/1021.pdf), which says that if we have two such points with nonzero indices wrt a given odd-prime order base, where the indices taken in the range $-(q-1)/2..(q-1)/2$ are distinct disregarding sign, then they have different x-coordinates. This is helpful, because it is easier to reason about the indices of points occurring in the scalar multiplication algorithm than it is to reason about their x-coordinates directly.

So, the required check is equivalent to saying that the following "indexed version" of the above algorithm never asserts:

```ignore
acc := 2
for i from n-1 down to 0 {
    p = k_{i+1} ? 1 : −1
    assert acc ≠ ± p
    assert (acc + p) ≠ acc    // X
    acc := (acc + p) + acc
    assert 0 < acc ≤ (q-1)/2
}
if k_0 = 0 {
    assert acc ≠ 1
    acc := acc - 1
}
```

The maximum value of `acc` is:
```ignore
    <--- n 1s --->
  1011111...111111
= 1100000...000000 - 1
```
= $2^{n+1} + 2^n - 1$

> The assertion labelled X obviously cannot fail because $p \neq 0$. It is possible to see that acc is monotonically increasing except in the last conditional. It reaches its largest value when $k$ is maximal, i.e. $2^{n+1} + 2^n - 1$.

So to entirely avoid exceptional cases, we would need $2^{n+1} + 2^n - 1 < (q-1)/2$. But we can use $n$ larger by $c$ if the last $c$ iterations use [complete addition](./addition.md#Complete-addition).

The first $i$ for which the algorithm using **only** incomplete addition fails is going to be $252$, since $2^{252+1} + 2^{252} - 1 > (q - 1)/2$. We need $n = 254$ to make the wraparound technique above work.

```python
sage: q = 0x40000000000000000000000000000000224698fc0994a8dd8c46eb2100000001
sage: 2^253 + 2^252 - 1 < (q-1)//2
False
sage: 2^252 + 2^251 - 1 < (q-1)//2
True
```

So the last three iterations of the loop ($i = 2..0$) need to use [complete addition](./addition.md#Complete-addition), as does the conditional subtraction at the end. Writing this out using ⸭ for incomplete addition (as we do in the spec), we have:

```ignore
Acc := [2] T
for i from 253 down to 3 {
    P := k_{i+1} ? T : −T
    Acc := (Acc ⸭ P) ⸭ Acc
}
for i from 2 down to 0 {
    P := k_{i+1} ? T : −T
    Acc := (Acc + P) + Acc  // complete addition
}
return (k_0 = 0) ? (Acc + (-T)) : Acc  // complete addition
```

## Constraint program for optimized double-and-add
Define a running sum $\mathbf{z_j} = \sum_{i=j}^{n} (\mathbf{k}_{i} \cdot 2^{i-j})$, where $n = 254$ and:

$$
\begin{aligned}
    &\mathbf{z}_{n+1} = 0,\\
    &\mathbf{z}_{n} = \mathbf{k}_{n}, \hspace{2em}\text{(most significant bit)}\\
	&\mathbf{z}_0 = k.\\
\end{aligned}
$$

$\begin{array}{l}
\text{Initialize } A_{254} = [2] T. \\
\\
\text{for } i \text{ from } 254 \text{ down to } 4: \\
\hspace{1.5em} \BoolCheck{\mathbf{k}_i} = 0 \\
\hspace{1.5em} \mathbf{z}_{i} = 2\mathbf{z}_{i+1} + \mathbf{k}_{i} \\
\hspace{1.5em} x_{P,i} = x_T \\
\hspace{1.5em} y_{P,i} = (2 \mathbf{k}_i - 1) \cdot y_T  \hspace{2em}\text{(conditionally negate)} \\
\hspace{1.5em} \lambda_{1,i} \cdot (x_{A,i} - x_{P,i}) = y_{A,i} - y_{P,i} \\
\hspace{1.5em} x_{R,i} = \lambda_{1,i}^2 - x_{A,i} - x_{P,i} \\
\hspace{1.5em} (\lambda_{1,i} + \lambda_{2,i}) \cdot (x_{A,i} - x_{R,i}) = 2 y_{\mathsf{A},i} \\
\hspace{1.5em} \lambda_{2,i}^2 = x_{A,i-1} + x_{R,i} + x_{A,i} \\
\hspace{1.5em} \lambda_{2,i} \cdot (x_{A,i} - x_{A,i-1}) = y_{A,i} + y_{A,i-1}. \\
\end{array}$

The helper $\BoolCheck{x} = x \cdot (1 - x)$.
After substitution of $x_{P,i}, y_{P,i}, x_{R,i}, y_{A,i}$, and $y_{A,i-1}$, this becomes:

$\begin{array}{l}
\text{Initialize } A_{254} = [2] T. \\
\\
\text{for } i \text{ from } 254 \text{ down to } 4: \\
\hspace{1.5em} \text{// let } \mathbf{k}_{i} = \mathbf{z}_{i} - 2\mathbf{z}_{i+1} \\
\hspace{1.5em} \text{// let } y_{A,i} = \frac{(\lambda_{1,i} + \lambda_{2,i}) \cdot (x_{A,i} - (\lambda_{1,i}^2 - x_{A,i} - x_T))}{2} \\[2ex]
\hspace{1.5em} \BoolCheck{\mathbf{k}_i} = 0 \\
\hspace{1.5em} \lambda_{1,i} \cdot (x_{A,i} - x_T) = y_{A,i} - (2 \mathbf{k}_i - 1) \cdot y_T \\
\hspace{1.5em} \lambda_{2,i}^2 = x_{A,i-1} + \lambda_{1,i}^2 - x_T \\[1ex]
\hspace{1.5em} \begin{cases}
                 \lambda_{2,i} \cdot (x_{A,i} - x_{A,i-1}) = y_{A,i} + y_{A, i-1}, &\text{if } i > 4 \\[0.5ex]
                 \lambda_{2,4} \cdot (x_{A,4} - x_{A,3}) = y_{A,4} + y_{A,3}^\text{witnessed}, &\text{if } i = 4.
               \end{cases}
\end{array}$

Here, $y_{A,3}^\text{witnessed}$ is assigned to a cell. This is unlike previous $y_{A,i}$'s, which were implicitly derived from $\lambda_{1,i}, \lambda_{2,i}, x_{A,i}, x_T$, but never actually assigned.

The bits $\mathbf{k}_{3 \dots 1}$ are used in three further steps, using [complete addition](./addition.md#Complete-addition):

$\begin{array}{l}
\text{for } i \text{ from } 3 \text{ down to } 1: \\
\hspace{1.5em} \text{// let } \mathbf{k}_{i} = \mathbf{z}_{i} - 2\mathbf{z}_{i+1} \\[0.5ex]
\hspace{1.5em} \BoolCheck{\mathbf{k}_i} = 0 \\
\hspace{1.5em} (x_{A,i-1}, y_{A,i-1}) = \left((x_{A,i}, y_{A,i}) + (x_T, y_T)\right) + (x_{A,i}, y_{A,i})
\end{array}$

If the least significant bit $\mathbf{k_0} = 1,$ we set $B = \mathcal{O},$ otherwise we set ${B = -T}$. Then we return ${A + B}$ using complete addition.

Let $B = \begin{cases}
(0, 0), &\text{ if } \mathbf{k_0} = 1, \\
(x_T, -y_T), &\text{ otherwise.}
\end{cases}$

Output $(x_{A,0}, y_{A,0}) + B$.

(Note that $(0, 0)$ represents $\mathcal{O}$.)

## Incomplete addition

We need six advice columns to witness $(x_T, y_T, \lambda_1, \lambda_2, x_{A,i}, \mathbf{z}_i)$. However, since $(x_T, y_T)$ are the same, we can perform two incomplete additions in a single row, reusing the same $(x_T, y_T)$. We split the scalar bits used in incomplete addition into $hi$ and $lo$ halves and process them in parallel. This means that we effectively have two for loops:
- the first, covering the $hi$ half for $i$ from $254$ down to $130$, with a special case at $i = 130$; and
- the second, covering the $lo$ half for the remaining $i$ from $129$ down to $4$, with a special case at $i = 4$.

$$
\begin{array}{|c|c|c|c|c|c|c|c|c|c|c|c|c|c|c|c|}
\hline
    x_T     &    y_T      &          z^{hi}           &    x_A^{hi}        &  \lambda_1^{hi}  &  \lambda_2^{hi}  &  q_1^{hi}  &  q_2^{hi}   &  q_3^{hi}  &         z^{lo}        &  x_A^{lo}   &  \lambda_1^{lo}     &  \lambda_2^{lo}   &  q_1^{lo}  &  q_2^{lo}   &  q_3^{lo}  \\\hline
            &             &  \mathbf{z}_{255} = 0     &                    & y_{A,254}=2[T]_y &                  &     1      &     0       &     0      &   \mathbf{z}_{130}    &             &   y_{A,129}         &                   &     1      &     0       &     0      \\\hline
    x_T     &    y_T      &    \mathbf{z}_{254}       & x_{A,254} = 2[T]_x & \lambda_{1,254}  & \lambda_{2,254}  &     0      &     1       &     0      &   \mathbf{z}_{129}    & x_{A,129}   & \lambda_{1,129}     & \lambda_{2,129}   &     0      &     1       &     0      \\\hline
    x_T     &    y_T      &    \mathbf{z}_{253}       &     x_{A,253}      & \lambda_{1,253}  & \lambda_{2,253}  &     0      &     1       &     0      &   \mathbf{z}_{128}    & x_{A,128}   & \lambda_{1,128}     & \lambda_{2,128}   &     0      &     1       &     0      \\\hline
   \vdots   &   \vdots    &         \vdots            &      \vdots        &      \vdots      &      \vdots      &   \vdots   &   \vdots    &   \vdots   &        \vdots         &  \vdots     &      \vdots         &      \vdots       &   \vdots   &   \vdots    &   \vdots   \\\hline
    x_T     &    y_T      &    \mathbf{z}_{130}       &     x_{A,130}      & \lambda_{1,130}  & \lambda_{2,130}  &     0      &     0       &     1      &   \mathbf{z}_5        & x_{A,5}     & \lambda_{1,5}       & \lambda_{2,5}     &     0      &     1       &     0      \\\hline
    x_T     &    y_T      &                           &     x_{A,129}      &    y_{A,129}     &                  &            &             &            &   \mathbf{z}_4        & x_{A,4}     & \lambda_{1,4}       & \lambda_{2,4}     &     0      &     0       &     1      \\\hline
            &             &                           &                    &                  &                  &            &             &            &                       & x_{A,3}     &     y_{A,3}         &                   &            &             &            \\\hline

\end{array}
$$

For each $hi$ and $lo$ half, we have three sets of gates. Note that $i$ is going from $255..=3$; $i$ is NOT indexing the rows.

### $q_1 = 1$ <a name="incomplete-first-row-gate">
This gate is only used on the first row (before the for loop). We check that $\lambda_1, \lambda_2$ are initialized to values consistent with the initial $y_A.$
$$
\begin{array}{|c|l|}
\hline
\text{Degree} & \text{Constraint} \\\hline
4 & q_1 \cdot \left(y_{A,n}^\text{witnessed} - y_{A,n}\right) = 0 \\\hline
\end{array}
$$
where
$$
\begin{aligned}
y_{A,n} &= \frac{(\lambda_{1,n} + \lambda_{2,n}) \cdot (x_{A,n} - (\lambda_{1,n}^2 - x_{A,n} - x_T))}{2},\\
y_{A,n}^\text{witnessed} &\text{ is witnessed.}
\end{aligned}
$$

### $q_2 = 1$ <a name="incomplete-main-loop-gate">
This gate is used on all rows corresponding to the for loop except the last.

$$
\begin{array}{|c|l|}
\hline
\text{Degree} & \text{Constraint} \\\hline
2 & q_2 \cdot \left(x_{T,cur} - x_{T,next}\right) = 0 \\\hline
2 & q_2 \cdot \left(y_{T,cur} - y_{T,next}\right) = 0 \\\hline
3 & q_2 \cdot \BoolCheck{\mathbf{k}_i} = 0, \text{ where } \mathbf{k}_i = \mathbf{z}_{i} - 2\mathbf{z}_{i+1} \\\hline
4 & q_2 \cdot \left(\lambda_{1,i} \cdot (x_{A,i} - x_{T,i}) - y_{A,i} + (2\mathbf{k}_i - 1) \cdot y_{T,i}\right) = 0 \\\hline
3 & q_2 \cdot \left(\lambda_{2,i}^2 - x_{A,i-1} - x_{R,i} - x_{A,i}\right) = 0 \\\hline
3 & q_2 \cdot \left(\lambda_{2,i} \cdot (x_{A,i} - x_{A,i-1}) - y_{A,i} - y_{A,i-1}\right) = 0 \\\hline
\end{array}
$$
where
$$
\begin{aligned}
x_{R,i} &= \lambda_{1,i}^2 - x_{A,i} - x_T, \\
y_{A,i} &= \frac{(\lambda_{1,i} + \lambda_{2,i}) \cdot (x_{A,i} - (\lambda_{1,i}^2 - x_{A,i} - x_T))}{2}, \\
y_{A,i-1} &= \frac{(\lambda_{1,i-1} + \lambda_{2,i-1}) \cdot (x_{A,i-1} - (\lambda_{1,i-1}^2 - x_{A,i-1} - x_T))}{2}, \\
\end{aligned}
$$

### $q_3 = 1$ <a name="incomplete-last-row-gate">
This gate is used on the final iteration of the for loop, handling the special case where we check that the output $y_A$ has been witnessed correctly.
$$
\begin{array}{|c|l|}
\hline
\text{Degree} & \text{Constraint} \\\hline
3 & q_3 \cdot \BoolCheck{\mathbf{k}_i} = 0, \text{ where } \mathbf{k}_i = \mathbf{z}_{i} - 2\mathbf{z}_{i+1} \\\hline
4 & q_3 \cdot \left(\lambda_{1,i} \cdot (x_{A,i} - x_{T,i}) - y_{A,i} + (2\mathbf{k}_i - 1) \cdot y_{T,i}\right) = 0 \\\hline
3 & q_3 \cdot \left(\lambda_{2,i}^2 - x_{A,i-1} - x_{R,i} - x_{A,i}\right) = 0 \\\hline
3 & q_3 \cdot \left(\lambda_{2,i} \cdot (x_{A,i} - x_{A,i-1}) - y_{A,i} - y_{A,i-1}^\text{witnessed}\right) = 0 \\\hline
\end{array}
$$
where
$$
\begin{aligned}
x_{R,i} &= \lambda_{1,i}^2 - x_{A,i} - x_T, \\
y_{A,i} &= \frac{(\lambda_{1,i} + \lambda_{2,i}) \cdot (x_{A,i} - (\lambda_{1,i}^2 - x_{A,i} - x_T))}{2},\\
y_{A,i-1}^\text{witnessed} &\text{ is witnessed.}
\end{aligned}
$$

## Complete addition

We reuse the [complete addition](addition.md#complete-addition) constraints to implement
the final $c$ rounds of double-and-add. This requires two rows per round because we need
9 advice columns for each complete addition. In the 10th advice column we stash the other
cells that we need to correctly implement the double-and-add:

- The base $y$ coordinate, so we can conditionally negate it as input to one of the
  complete additions.
- The running sum, which we constrain over two rows instead of sequentially.

### Layout

$$
\begin{array}{|c|c|c|c|c|c|c|c|c|c|c|}
a_0 & a_1 & a_2 & a_3 &    a_4    &   a_5    &   a_6   &   a_7    &   a_8    & a_9     & q_\texttt{mul\_decompose\_var} \\\hline
x_T & y_p & x_A & y_A & \lambda_1 & \alpha_1 & \beta_1 & \gamma_1 & \delta_1 & z_{i+1} & 0 \\\hline
x_A & y_A & x_q & y_q & \lambda_2 & \alpha_2 & \beta_2 & \gamma_2 & \delta_2 & y_T     & 1 \\\hline
    &     & x_r & y_r &           &          &         &          &          & z_i     & 0 \\\hline
\end{array}
$$

### Constraints <a name="complete-gate">

In addition to the complete addition constraints, we define the following gate:

$$
\begin{array}{|c|l|}
\hline
\text{Degree} & \text{Constraint} \\\hline
  & q_\texttt{mul\_decompose\_var} \cdot \BoolCheck{\mathbf{k}_i} = 0 \\\hline
  & q_\texttt{mul\_decompose\_var} \cdot \Ternary{\mathbf{k}_i}{y_T - y_p}{y_T + y_p} = 0 \\\hline
\end{array}
$$
where $\mathbf{k}_i = \mathbf{z}_{i} - 2\mathbf{z}_{i+1}$.

## LSB

### Layout

$$
\begin{array}{|c|c|c|c|c|c|c|c|c|c|c|}
a_0 & a_1 & a_2 & a_3 &   a_4   &  a_5   &  a_6  &  a_7   &  a_8   & a_9 & q_\texttt{mul\_lsb} \\\hline
x_p & y_p & x_A & y_A & \lambda & \alpha & \beta & \gamma & \delta & z_1 & 1 \\\hline
x_T & y_T & x_r & y_r &         &        &       &        &        & z_0 & 0 \\\hline
\end{array}
$$

### Constraints <a name="lsb-gate">

$$
\begin{array}{|c|l|}
\hline
\text{Degree} & \text{Constraint} \\\hline
  & q_\texttt{mul\_lsb} \cdot \BoolCheck{\mathbf{k}_0} = 0 \\\hline
  & q_\texttt{mul\_lsb} \cdot \Ternary{\mathbf{k}_0}{x_p}{x_p - x_T} = 0 \\\hline
  & q_\texttt{mul\_lsb} \cdot \Ternary{\mathbf{k}_0}{y_p}{y_p + y_T} = 0 \\\hline
\end{array}
$$
where $\mathbf{k}_0 = \mathbf{z}_0 - 2\mathbf{z}_1$.

## Overflow check

$\mathbf{z}_i$ cannot overflow for any $i \geq 1$, because it is a weighted sum of bits only up to $2^{n-1} = 2^{253}$, which is smaller than $p$ (and also $q$).

However, $\mathbf{z}_0 = \alpha + t_q$ *can* overflow $[0, p)$.

> Note: for full-width scalar mul, it may not be possible to represent $\mathbf{z}_0$ in the base field (e.g. when the base field is Pasta's $\mathbb{F}_p$ and $p < q$).
> In that case, we need to special-case the row that would mention $\mathbf{z}_0$ so that it is correct for whatever representation we use for a full-width scalar.
> Our representation for $k$ will be the pair $(\mathbb{k}_{254}, k' = k - 2^{254} \cdot \mathbb{k}_{254})$.
> We'll use $k'$ in place of $\alpha + t_q$ for $\mathbf{z}_0$, constraining $k'$ to 254 bits so that it fits in an $\mathbb{F}_p$ element.
> Then we just have to generalize the argument below to work for $k' \in [0, 2 \cdot t_q)$ (because the maximum value of $\alpha + t_q$ is $q - 1 + t_q = 2^{254} + t_q - 1 + t_q$).

Since overflow can only occur in the final step that constrains $\mathbf{z}_0 = 2 \cdot \mathbf{z}_1 + \mathbf{k}_0$, we have $\mathbf{z}_0 = k \pmod{p}$. It is then sufficient to also check that $\mathbf{z}_0 = \alpha + t_q \pmod{p}$ (so that $k = \alpha + t_q \pmod{p}$) and that $k \in [t_q, p + t_q)$. These conditions together imply that $k = \alpha + t_q$ as an integer, and so $2^{254} + k = \alpha \pmod{q}$ as required.

> Note: the bits $\mathbf{k}_{254..0}$ do not represent a value reduced modulo $q$, but rather a representation of the unreduced $\alpha + t_q$.

### Optimized check for $k \in [t_q, p + t_q)$

Since $t_p + t_q < 2^{130}$ (also true if $p$ and $q$ are swapped), we have $$[t_q, p + t_q) = [t_q, t_q + 2^{130}) \;\cup\; [2^{130}, 2^{254}) \;\cup\; \big([2^{254}, 2^{254} + 2^{130}) \;\cap\; [p + t_q - 2^{130}, p + t_q)\big).$$

We may assume that $k = \alpha + t_q \pmod{p}$.

(This is true for the use of variable-base scalar mul in Orchard, where we know that $\alpha < p$. If is also true if we swap $p$ and $q$ so that we have $p > q$.
It is *not* true for a full-width scalar $\alpha \geq p$ when $p < q$.)

Therefore,
$\begin{array}{rcl}
k \in [t_q, p + t_q) &\Leftrightarrow& \big(k \in [t_q, t_q + 2^{130}) \;\vee\; k \in [2^{130}, 2^{254})\big) \;\vee\; \\
                     &               & \big(k \in [2^{254}, 2^{254} + 2^{130}) \;\wedge\; k \in [p + t_q - 2^{130}, p + t_q)\big) \\
                     \\
                     &\Leftrightarrow& \big(\mathbf{k}_{254} = 0 \implies (k \in [t_q, t_q + 2^{130}) \;\vee\; k \in [2^{130}, 2^{254}))\big) \;\wedge \\
                     &               & \big(\mathbf{k}_{254} = 1 \implies (k \in [2^{254}, 2^{254} + 2^{130}) \;\wedge\; k \in [p + t_q - 2^{130}, p + t_q))\big) \\
                     \\
                     &\Leftrightarrow& \big(\mathbf{k}_{254} = 0 \implies (\alpha \in [0, 2^{130}) \;\vee\; k \in [2^{130}, 2^{254}))\big) \;\wedge \\
                     &               & \big(\mathbf{k}_{254} = 1 \implies (k \in [2^{254}, 2^{254} + 2^{130}) \;\wedge\; (\alpha + 2^{130}) \bmod p \in [0, 2^{130}))\big) \;\;Ⓐ
\end{array}$

> Given $k \in [2^{254}, 2^{254} + 2^{130})$, we prove equivalence of $k \in [p + t_q - 2^{130}, p + t_q)$ and $(\alpha + 2^{130}) \bmod p \in [0, 2^{130})$ as follows:
> * shift the range by $2^{130} - p - t_q$ to give $k + 2^{130} - p - t_q \in [0, 2^{130})$;
> * observe that $k + 2^{130} - p - t_q$ is guaranteed to be in $[2^{130} - t_p - t_q, 2^{131} - t_p - t_q)$ and therefore cannot overflow or underflow modulo $p$;
> * using the fact that $k = \alpha + t_q \pmod{p}$, observe that $(k + 2^{130} - p - t_q) \bmod p = (\alpha + t_q + 2^{130} - p - t_q) \bmod p = (\alpha + 2^{130}) \bmod p$.
>
> (We can see in a different way that this is correct by observing that it checks whether $\alpha \bmod p \in [p - 2^{130}, p)$, so the upper bound is aligned as we would expect.)

Now, we can continue optimizing from $Ⓐ$:

$\begin{array}{rcl}
k \in [t_q, p + t_q) &\Leftrightarrow& \big(\mathbf{k}_{254} = 0 \implies (\alpha \in [0, 2^{130}) \;\vee\; k \in [2^{130}, 2^{254})\big) \;\wedge \\
                     &               & \big(\mathbf{k}_{254} = 1 \implies (k \in [2^{254}, 2^{254} + 2^{130}) \;\wedge\; (\alpha + 2^{130}) \bmod p \in [0, 2^{130}))\big) \\
                     \\
                     &\Leftrightarrow& \big(\mathbf{k}_{254} = 0 \implies (\alpha \in [0, 2^{130}) \;\vee\; \mathbf{k}_{253..130} \text{ are not all } 0)\big) \;\wedge \\
                     &               & \big(\mathbf{k}_{254} = 1 \implies (\mathbf{k}_{253..130} \text{ are all } 0 \;\wedge\; (\alpha + 2^{130}) \bmod p \in [0, 2^{130}))\big)
\end{array}$

Constraining $\mathbf{k}_{253..130}$ to be all-$0$ or not-all-$0$ can be implemented almost "for free", as follows.

Recall that $\mathbf{z}_i = \sum_{h=i}^{n} (\mathbf{k}_{h} \cdot 2^{h-i})$, so we have:

$\begin{array}{rcl}
                                 \mathbf{z}_{130} &=& \sum_{h=130}^{254} (\mathbf{k}_h \cdot 2^{h-130}) \\
                                 \mathbf{z}_{130} &=& \mathbf{k}_{254} \cdot 2^{254-130} + \sum_{h=130}^{253} (\mathbf{k}_h \cdot 2^{h-130}) \\
\mathbf{z}_{130} - \mathbf{k}_{254} \cdot 2^{124} &=& \sum_{h=130}^{253} (\mathbf{k}_h \cdot 2^{h-130})
\end{array}$

So $\mathbf{k}_{253..130}$ are all $0$ exactly when $\mathbf{z}_{130} = \mathbf{k}_{254} \cdot 2^{124}$.

Finally, we can merge the $130$-bit decompositions for the $\mathbf{k}_{254} = 0$ and $\mathbf{k}_{254} = 1$ cases by checking that $(\alpha + \mathbf{k}_{254} \cdot 2^{130}) \bmod p \in [0, 2^{130})$.

### Overflow check constraints

Let $s = \alpha + \mathbf{k}_{254} \cdot 2^{130}$. The constraints for the overflow check are:

$$
\begin{aligned}
\mathbf{z}_0 &= \alpha + t_q \pmod{p} \\
\mathbf{k}_{254} = 1 \implies \big(\mathbf{z}_{130} &= 2^{124} \;\wedge\; s \bmod p \in [0, 2^{130})\big) \\
\mathbf{k}_{254} = 0 \implies \big(\mathbf{z}_{130} &\neq 0 \;\vee\; s \bmod p \in [0, 2^{130})\big)
\end{aligned}
$$

Define $\mathsf{inv0}(x) = \begin{cases} 0, &\text{if } x = 0 \\ 1/x, &\text{otherwise.} \end{cases}$

Witness $\eta = \mathsf{inv0}(\mathbf{z}_{130})$, and decompose $s \bmod p$ as $\mathbf{s}_{129..0}$.

Then the needed gates are:

$$
\begin{array}{|c|l|}
\hline
\text{Degree} & \text{Constraint} \\\hline
2 & q_\texttt{mul\_overflow} \cdot \left(s - (\alpha + \mathbf{k}_{254} \cdot 2^{130})\right) = 0 \\\hline
2 & q_\texttt{mul\_overflow} \cdot \left(\mathbf{z}_0 - \alpha - t_q\right) = 0 \\\hline
3 & q_\texttt{mul\_overflow} \cdot \left(\mathbf{k}_{254} \cdot (\mathbf{z}_{130} - 2^{124})\right) = 0 \\\hline
3 & q_\texttt{mul\_overflow} \cdot \left(\mathbf{k}_{254} \cdot (s - \sum\limits_{i=0}^{129} 2^i \cdot \mathbf{s}_i)/2^{130}\right) = 0 \\\hline
5 & q_\texttt{mul\_overflow} \cdot \left((1 - \mathbf{k}_{254}) \cdot (1 - \mathbf{z}_{130} \cdot \eta) \cdot (s - \sum\limits_{i=0}^{129} 2^i \cdot \mathbf{s}_i)/2^{130}\right) = 0 \\\hline
\end{array}
$$
where $(s - \sum\limits_{i=0}^{129} 2^i \cdot \mathbf{s}_i)/2^{130}$ can be computed by another running sum. Note that the factor of $1/2^{130}$ has no effect on the constraint, since the RHS is zero.

#### Running sum range check
We make use of a $10$-bit [lookup range check](../decomposition.md#lookup-decomposition) in the circuit to subtract the low $130$ bits of $\mathbf{s}$. The range check subtracts the first $13 \cdot 10$ bits of $\mathbf{s},$ and right-shifts the result to give $(s - \sum\limits_{i=0}^{129} 2^i \cdot \mathbf{s}_i)/2^{130}.$

## Overflow check (general)
Recall that we defined $\mathbf{z}_j = \sum_{i=j}^n (\mathbf{k}_i \cdot 2^{i−j})$, where $n = 254$.

$\mathbf{z}_j$ cannot overflow for any $j \geq 1$, because it is a weighted sum of bits only up to and including $2^{n-j}$. When $n = 254$ and $j = 1$ this sum can be at most $2^{254} - 1$, which is smaller than $p$ (and also $q$).

However, for full-width scalar mul, it may not be possible to represent $\mathbf{z}_0$ in the base field (e.g. when the base field is Pasta's $\mathbb{F}_p$ and $p < q$). In that case $\mathbf{z}_0 = \alpha + t_q$ *can* overflow $[0, p)$.

So, we need to special-case the row that would mention $\mathbf{z}_0$ so that it is correct for whatever representation we use for a full-width scalar.

Our representation for $k$ will be the pair $(\mathbf{k}_{254}, k' = k - 2^{254} \cdot \mathbf{k}_{254})$. We'll use $k'$ in place of $\alpha + t_q$ for $\mathbf{z}_0$, constraining $k'$ to 254 bits so that it fits in an $\mathbb{F}_p$ element.

Then we just have to generalize the [overflow check used for variable-base scalar mul in the Orchard circuit](https://zcash.github.io/halo2/design/gadgets/ecc/var-base-scalar-mul.html#overflow-check) to work for $k' \in [0, 2 \cdot t_q)$ (because the maximum value of $\alpha + t_q$ is $q - 1 + t_q = 2^{254} + t_q - 1 + t_q$).


> Note: the bits $\mathbf{k}_{254..0}$ do not represent a value reduced modulo $q$, but rather a representation of the unreduced $\alpha + t_q$.

Overflow can only occur in the final step that constrains $\mathbf{z}_0 = 2 \cdot \mathbf{z}_1 + \mathbf{k}_0$, and only if $\mathbf{z}_1$ has the bit with weight $2^{253}$ set (i.e. if $\mathbf{k}_{254} = 1$). If we instead set $\mathbf{z}_0 = 2 \cdot \mathbf{z}_1 - 2^{254} \cdot \mathbf{k}_{254} + \mathbf{k}_0$, now $\mathbf{z}_0$ cannot overflow and should be equal to $k'$.

It is then sufficient to also check that $\mathbf{z}_0 + 2^{254} \cdot \mathbf{k}_{254} = \alpha + t_q$ as an integer where $\alpha \in [0, q)$.

Represent $\alpha$ as $2^{254} \cdot \boldsymbol\alpha_{254} + 2^{253} \cdot \boldsymbol\alpha_{253} + \alpha''$ where we constrain $\alpha'' \in [0, 2^{253})$ and $\boldsymbol\alpha_{253}$ and $\boldsymbol\alpha_{254}$ to boolean. For this to be a canonical representation we also need $\boldsymbol \alpha_{254} = 1 \implies (\boldsymbol \alpha_{253} = 0 \;\wedge\; \alpha'' \in [0, t_q))$.

Let $\alpha' = 2^{253} \cdot \boldsymbol\alpha_{253} + \alpha''$.

If $\boldsymbol\alpha_{254} = 1$:
* constrain $\mathbf{k}_{254} = 1$ and $\mathbf{z}_0 = \alpha' + t_q$. This cannot overflow because in this case $\alpha' \in [0, t_q)$ and so $\mathbf{z}_0 \in [0, 2 \cdot t_q)$.

If $\boldsymbol\alpha_{254} = 0$:
* we should have $\mathbf{k}_{254} = 1$ iff $\alpha' \in [2^{254} - t_q, 2^{254})$, i.e. witness $\mathbf{k}_{254}$ as boolean and then
  * If $\mathbf{k}_{254} = 0$ then constrain $\alpha' \not\in [2^{254} - t_q, 2^{254})$.
    * This can be done by constraining either $\boldsymbol\alpha_{253} = 0$ or $\alpha'' + t_q \in [0, 2^{253})$. ($\alpha'' + t_q$ cannot overflow.)
  * If $\mathbf{k}_{254} = 1$ then constrain $\alpha' \in [2^{254} - t_q, 2^{254})$.
    * This can be done by constraining $\alpha' - (2^{254} - t_q) \in [0, 2^{130})$ and $\alpha' - 2^{254} + 2^{130} \in [0, 2^{130})$.

### Overflow check constraints (general)

Represent $\alpha$ as $2^{254} \cdot \boldsymbol\alpha_{254} + 2^{253} \cdot \boldsymbol\alpha_{253} + \alpha''$ as above.

The constraints for the overflow check are:

$$
\begin{aligned}
&\alpha'' \in [0, 2^{253}) \\
&\boldsymbol\alpha_{253} \in \{0,1\} \\
&\boldsymbol\alpha_{254} \in \{0,1\} \\
&\mathbf{k}_{254} \in \{0,1\} \\
\boldsymbol \alpha_{254} = 1 \implies &\boldsymbol \alpha_{253} = 0 \\
\boldsymbol \alpha_{254} = 1 \implies &\alpha'' \in [0, 2^{130}) \;❁ \\
\boldsymbol \alpha_{254} = 1 \implies &\alpha'' + 2^{130} - t_q \in [0, 2^{130}) \;❁ \\
\boldsymbol\alpha_{254} = 1 \implies &\mathbf{k}_{254} = 1 \\
\boldsymbol\alpha_{254} = 1 \implies &\mathbf{z}_0 = \alpha' + t_q \\
\boldsymbol\alpha_{254} = 0 \;\wedge\; \boldsymbol\alpha_{253} = 1 \;\wedge\; \mathbf{k}_{254} = 0 &\implies \alpha'' + t_q \in [0, 2^{253}) \\
\boldsymbol\alpha_{254} = 0 \;\wedge\; \mathbf{k}_{254} = 1 &\implies \alpha' - 2^{254} + t_q \in [0, 2^{130}) \;❁ \\
\boldsymbol\alpha_{254} = 0 \;\wedge\; \mathbf{k}_{254} = 1 &\implies \alpha' - 2^{254} + 2^{130} \in [0, 2^{130}) \;❁ \\
\end{aligned}
$$

Note that the four 130-bit constraints marked $❁$ are in two pairs that occur in disjoint cases. We can therefore combine them into two 130-bit constraints using a new witness variable $u$; the other constraint always being on $u + 2^{130} - t_q$:

$$
\begin{aligned}
\boldsymbol \alpha_{254} = 1 \implies &u = \alpha'' \\
\boldsymbol\alpha_{254} = 0 \;\wedge\; \mathbf{k}_{254} = 1 \implies &u = \alpha' - 2^{254} + t_q \\
&u \in [0, 2^{130}) \\
&u + 2^{130} - t_q \in [0, 2^{130}) \\
\end{aligned}
$$

($u$ is unconstrained and can be witnessed as $0$ in the case $\boldsymbol\alpha_{254} = 0 \;\wedge\; \mathbf{k}_{254} = 0$.)

### Cost

* 25 10-bit and one 3-bit range check, to constrain $\alpha''$ to 253 bits;
* 25 10-bit and one 3-bit range check, to constrain $\alpha'' + t_q$ to 253 bits when $\boldsymbol\alpha_{254} = 0 \;\wedge\; \boldsymbol\alpha_{253} = 1 \;\wedge\; \mathbf{k}_{254} = 0$;
* two times 13 10-bit range checks.



# 变基标量乘法

在 Orchard 电路中，我们需要检查 $\mathsf{pk_d} = [\mathsf{ivk}] \mathsf{g_d}$，其中 $\mathsf{ivk} \in [0, p)$ 且标量域为 $\mathbb{F}_q$，且 $p < q$。

我们有 $p = 2^{254} + t_p$ 和 $q = 2^{254} + t_q$，其中 $t_p, t_q < 2^{128}$。

## 见证标量

我们尝试计算 $[\alpha] T$，其中 $\alpha \in [0, q)$。设 $k = \alpha + t_q$ 且 $n = 254$。然后我们可以计算

$\begin{array}{cl}
[2^{254} + (\alpha + t_q)] T &= [2^{254} + (\alpha + t_q) - (2^{254} + t_q)] T \\
                             &= [\alpha] T
\end{array}$

前提是 $\alpha + t_q \in [0, 2^{n+1})$，即 $\alpha < 2^{n+1} - t_q$，这覆盖了我们所需的整个范围，因为实际上 $2^{255} - t_q > q$。

因此，给定标量 $\alpha$，我们见证 $k = \alpha + t_q$ 的布尔分解。（我们使用大端位序以便于输入到变基标量乘法算法中。）

$$k = k_{254} \cdot 2^{254} + k_{253} \cdot 2^{253} + \cdots + k_0.$$

## 变基标量乘法

我们使用优化的双加算法，从 ["Faster variable-base scalar multiplication in zk-SNARK circuits"](https://github.com/zcash/zcash/issues/3924) 复制而来，并做了一些变量名的更改：

```ignore
Acc := [2] T
for i from n-1 down to 0 {
    P := k_{i+1} ? T : −T
    Acc := (Acc + P) + Acc
}
return (k_0 = 0) ? (Acc - T) : Acc
```

仍需检查每对要相加的点的 $x$ 坐标是否不同。

在素数阶群中相加点时，我们可以依赖 [Halo 论文](https://eprint.iacr.org/2019/1021.pdf) 附录 C 中的定理 3，该定理指出，如果我们有两个点相对于给定的奇素数阶基的非零索引，且这些索引在 $-(q-1)/2..(q-1)/2$ 范围内不考虑符号时不同，则它们的 $x$ 坐标不同。这很有帮助，因为推理标量乘法算法中点的索引比直接推理它们的 $x$ 坐标更容易。

因此，所需的检查等同于以下“索引版本”的算法从不断言：

```ignore
acc := 2
for i from n-1 down to 0 {
    p = k_{i+1} ? 1 : −1
    assert acc ≠ ± p
    assert (acc + p) ≠ acc    // X
    acc := (acc + p) + acc
    assert 0 < acc ≤ (q-1)/2
}
if k_0 = 0 {
    assert acc ≠ 1
    acc := acc - 1
}
```

`acc` 的最大值为：
```ignore
    <--- n 1s --->
  1011111...111111
= 1100000...000000 - 1
```
= $2^{n+1} + 2^n - 1$

> 标记为 X 的断言显然不能失败，因为 $p \neq 0$。可以看到，`acc` 是单调递增的，除了最后一个条件。当 $k$ 最大时，即 $2^{n+1} + 2^n - 1$ 时，`acc` 达到其最大值。

因此，为了完全避免特殊情况，我们需要 $2^{n+1} + 2^n - 1 < (q-1)/2$。但如果最后 $c$ 次迭代使用 [完全加法](./addition.md#Complete-addition)，我们可以使用更大的 $n$。

仅使用不完全加法的算法首次失败的 $i$ 将是 $252$，因为 $2^{252+1} + 2^{252} - 1 > (q - 1)/2$。我们需要 $n = 254$ 才能使上述环绕技术生效。

```python
sage: q = 0x40000000000000000000000000000000224698fc0994a8dd8c46eb2100000001
sage: 2^253 + 2^252 - 1 < (q-1)//2
False
sage: 2^252 + 2^251 - 1 < (q-1)//2
True
```

因此，循环的最后三次迭代（$i = 2..0$）需要使用 [完全加法](./addition.md#Complete-addition)，最后的条件减法也需要。用 ⸭ 表示不完全加法（如规范中所示），我们得到：

```ignore
Acc := [2] T
for i from 253 down to 3 {
    P := k_{i+1} ? T : −T
    Acc := (Acc ⸭ P) ⸭ Acc
}
for i from 2 down to 0 {
    P := k_{i+1} ? T : −T
    Acc := (Acc + P) + Acc  // 完全加法
}
return (k_0 = 0) ? (Acc + (-T)) : Acc  // 完全加法
```

## 优化双加算法的约束程序

定义一个运行和 $\mathbf{z_j} = \sum_{i=j}^{n} (\mathbf{k}_{i} \cdot 2^{i-j})$，其中 $n = 254$ 且：

$$
\begin{aligned}
    &\mathbf{z}_{n+1} = 0,\\
    &\mathbf{z}_{n} = \mathbf{k}_{n}, \hspace{2em}\text{(最高有效位)}\\
	&\mathbf{z}_0 = k.\\
\end{aligned}
$$

$\begin{array}{l}
\text{初始化 } A_{254} = [2] T. \\
\\
\text{对于 } i \text{ 从 } 254 \text{ 到 } 4: \\
\hspace{1.5em} \BoolCheck{\mathbf{k}_i} = 0 \\
\hspace{1.5em} \mathbf{z}_{i} = 2\mathbf{z}_{i+1} + \mathbf{k}_{i} \\
\hspace{1.5em} x_{P,i} = x_T \\
\hspace{1.5em} y_{P,i} = (2 \mathbf{k}_i - 1) \cdot y_T  \hspace{2em}\text{(条件取反)} \\
\hspace{1.5em} \lambda_{1,i} \cdot (x_{A,i} - x_{P,i}) = y_{A,i} - y_{P,i} \\
\hspace{1.5em} x_{R,i} = \lambda_{1,i}^2 - x_{A,i} - x_{P,i} \\
\hspace{1.5em} (\lambda_{1,i} + \lambda_{2,i}) \cdot (x_{A,i} - x_{R,i}) = 2 y_{\mathsf{A},i} \\
\hspace{1.5em} \lambda_{2,i}^2 = x_{A,i-1} + x_{R,i} + x_{A,i} \\
\hspace{1.5em} \lambda_{2,i} \cdot (x_{A,i} - x_{A,i-1}) = y_{A,i} + y_{A,i-1}. \\
\end{array}$

辅助函数 $\BoolCheck{x} = x \cdot (1 - x)$。
在代入 $x_{P,i}, y_{P,i}, x_{R,i}, y_{A,i}$ 和 $y_{A,i-1}$ 后，这变为：

$\begin{array}{l}
\text{初始化 } A_{254} = [2] T. \\
\\
\text{对于 } i \text{ 从 } 254 \text{ 到 } 4: \\
\hspace{1.5em} \text{// 令 } \mathbf{k}_{i} = \mathbf{z}_{i} - 2\mathbf{z}_{i+1} \\
\hspace{1.5em} \text{// 令 } y_{A,i} = \frac{(\lambda_{1,i} + \lambda_{2,i}) \cdot (x_{A,i} - (\lambda_{1,i}^2 - x_{A,i} - x_T))}{2} \\[2ex]
\hspace{1.5em} \BoolCheck{\mathbf{k}_i} = 0 \\
\hspace{1.5em} \lambda_{1,i} \cdot (x_{A,i} - x_T) = y_{A,i} - (2 \mathbf{k}_i - 1) \cdot y_T \\
\hspace{1.5em} \lambda_{2,i}^2 = x_{A,i-1} + \lambda_{1,i}^2 - x_T \\[1ex]
\hspace{1.5em} \begin{cases}
                 \lambda_{2,i} \cdot (x_{A,i} - x_{A,i-1}) = y_{A,i} + y_{A, i-1}, &\text{如果 } i > 4 \\[0.5ex]
                 \lambda_{2,4} \cdot (x_{A,4} - x_{A,3}) = y_{A,4} + y_{A,3}^\text{witnessed}, &\text{如果 } i = 4.
               \end{cases}
\end{array}$

这里，$y_{A,3}^\text{witnessed}$ 被分配到单元格中。这与之前的 $y_{A,i}$ 不同，后者是从 $\lambda_{1,i}, \lambda_{2,i}, x_{A,i}, x_T$ 隐式导出的，但从未实际分配。

位 $\mathbf{k}_{3 \dots 1}$ 用于接下来的三个步骤，使用 [完全加法](./addition.md#Complete-addition)：

$\begin{array}{l}
\text{对于 } i \text{ 从 } 3 \text{ 到 } 1: \\
\hspace{1.5em} \text{// 令 } \mathbf{k}_{i} = \mathbf{z}_{i} - 2\mathbf{z}_{i+1} \\[0.5ex]
\hspace{1.5em} \BoolCheck{\mathbf{k}_i} = 0 \\
\hspace{1.5em} (x_{A,i-1}, y_{A,i-1}) = \left((x_{A,i}, y_{A,i}) + (x_T, y_T)\right) + (x_{A,i}, y_{A,i})
\end{array}$

如果最低有效位 $\mathbf{k_0} = 1,$ 我们设 $B = \mathcal{O},$ 否则我们设 ${B = -T}$。然后我们使用完全加法返回 ${A + B}$。

设 $B = \begin{cases}
(0, 0), &\text{ 如果 } \mathbf{k_0} = 1, \\
(x_T, -y_T), &\text{ 否则。}
\end{cases}$

输出 $(x_{A,0}, y_{A,0}) + B$。

（注意，$(0, 0)$ 表示 $\mathcal{O}$。）


## 不完全加法

我们需要六个建议列来见证 $(x_T, y_T, \lambda_1, \lambda_2, x_{A,i}, \mathbf{z}_i)$。然而，由于 $(x_T, y_T)$ 是相同的，我们可以在单行中执行两个不完全加法，重复使用相同的 $(x_T, y_T)$。我们将用于不完全加法的标量位分成 $hi$ 和 $lo$ 两半，并并行处理它们。这意味着我们实际上有两个 for 循环：
- 第一个循环覆盖 $hi$ 半部分，$i$ 从 $254$ 到 $130$，在 $i = 130$ 时有特殊情况；
- 第二个循环覆盖剩余的 $lo$ 半部分，$i$ 从 $129$ 到 $4$，在 $i = 4$ 时有特殊情况。

$$
\begin{array}{|c|c|c|c|c|c|c|c|c|c|c|c|c|c|c|c|}
\hline
    x_T     &    y_T      &          z^{hi}           &    x_A^{hi}        &  \lambda_1^{hi}  &  \lambda_2^{hi}  &  q_1^{hi}  &  q_2^{hi}   &  q_3^{hi}  &         z^{lo}        &  x_A^{lo}   &  \lambda_1^{lo}     &  \lambda_2^{lo}   &  q_1^{lo}  &  q_2^{lo}   &  q_3^{lo}  \\\hline
            &             &  \mathbf{z}_{255} = 0     &                    & y_{A,254}=2[T]_y &                  &     1      &     0       &     0      &   \mathbf{z}_{130}    &             &   y_{A,129}         &                   &     1      &     0       &     0      \\\hline
    x_T     &    y_T      &    \mathbf{z}_{254}       & x_{A,254} = 2[T]_x & \lambda_{1,254}  & \lambda_{2,254}  &     0      &     1       &     0      &   \mathbf{z}_{129}    & x_{A,129}   & \lambda_{1,129}     & \lambda_{2,129}   &     0      &     1       &     0      \\\hline
    x_T     &    y_T      &    \mathbf{z}_{253}       &     x_{A,253}      & \lambda_{1,253}  & \lambda_{2,253}  &     0      &     1       &     0      &   \mathbf{z}_{128}    & x_{A,128}   & \lambda_{1,128}     & \lambda_{2,128}   &     0      &     1       &     0      \\\hline
   \vdots   &   \vdots    &         \vdots            &      \vdots        &      \vdots      &      \vdots      &   \vdots   &   \vdots    &   \vdots   &        \vdots         &  \vdots     &      \vdots         &      \vdots       &   \vdots   &   \vdots    &   \vdots   \\\hline
    x_T     &    y_T      &    \mathbf{z}_{130}       &     x_{A,130}      & \lambda_{1,130}  & \lambda_{2,130}  &     0      &     0       &     1      &   \mathbf{z}_5        & x_{A,5}     & \lambda_{1,5}       & \lambda_{2,5}     &     0      &     1       &     0      \\\hline
    x_T     &    y_T      &                           &     x_{A,129}      &    y_{A,129}     &                  &            &             &            &   \mathbf{z}_4        & x_{A,4}     & \lambda_{1,4}       & \lambda_{2,4}     &     0      &     0       &     1      \\\hline
            &             &                           &                    &                  &                  &            &             &            &                       & x_{A,3}     &     y_{A,3}         &                   &            &             &            \\\hline
\end{array}
$$

对于每个 $hi$ 和 $lo$ 半部分，我们有三组门。注意，$i$ 从 $255..=3$；$i$ 不是索引行。

### $q_1 = 1$ <a name="incomplete-first-row-gate">
此门仅在循环前的第一行使用。我们检查 $\lambda_1, \lambda_2$ 是否初始化为与初始 $y_A$ 一致的值。
$$
\begin{array}{|c|l|}
\hline
\text{度数} & \text{约束} \\\hline
4 & q_1 \cdot \left(y_{A,n}^\text{witnessed} - y_{A,n}\right) = 0 \\\hline
\end{array}
$$
其中
$$
\begin{aligned}
y_{A,n} &= \frac{(\lambda_{1,n} + \lambda_{2,n}) \cdot (x_{A,n} - (\lambda_{1,n}^2 - x_{A,n} - x_T))}{2},\\
y_{A,n}^\text{witnessed} &\text{ 被见证。}
\end{aligned}
$$

### $q_2 = 1$ <a name="incomplete-main-loop-gate">
此门用于循环中除最后一行外的所有行。

$$
\begin{array}{|c|l|}
\hline
\text{度数} & \text{约束} \\\hline
2 & q_2 \cdot \left(x_{T,cur} - x_{T,next}\right) = 0 \\\hline
2 & q_2 \cdot \left(y_{T,cur} - y_{T,next}\right) = 0 \\\hline
3 & q_2 \cdot \BoolCheck{\mathbf{k}_i} = 0, \text{ 其中 } \mathbf{k}_i = \mathbf{z}_{i} - 2\mathbf{z}_{i+1} \\\hline
4 & q_2 \cdot \left(\lambda_{1,i} \cdot (x_{A,i} - x_{T,i}) - y_{A,i} + (2\mathbf{k}_i - 1) \cdot y_{T,i}\right) = 0 \\\hline
3 & q_2 \cdot \left(\lambda_{2,i}^2 - x_{A,i-1} - x_{R,i} - x_{A,i}\right) = 0 \\\hline
3 & q_2 \cdot \left(\lambda_{2,i} \cdot (x_{A,i} - x_{A,i-1}) - y_{A,i} - y_{A,i-1}\right) = 0 \\\hline
\end{array}
$$
其中
$$
\begin{aligned}
x_{R,i} &= \lambda_{1,i}^2 - x_{A,i} - x_T, \\
y_{A,i} &= \frac{(\lambda_{1,i} + \lambda_{2,i}) \cdot (x_{A,i} - (\lambda_{1,i}^2 - x_{A,i} - x_T))}{2}, \\
y_{A,i-1} &= \frac{(\lambda_{1,i-1} + \lambda_{2,i-1}) \cdot (x_{A,i-1} - (\lambda_{1,i-1}^2 - x_{A,i-1} - x_T))}{2}, \\
\end{aligned}
$$

### $q_3 = 1$ <a name="incomplete-last-row-gate">
此门用于循环的最后一次迭代，处理特殊情况，检查输出的 $y_A$ 是否被正确见证。
$$
\begin{array}{|c|l|}
\hline
\text{度数} & \text{约束} \\\hline
3 & q_3 \cdot \BoolCheck{\mathbf{k}_i} = 0, \text{ 其中 } \mathbf{k}_i = \mathbf{z}_{i} - 2\mathbf{z}_{i+1} \\\hline
4 & q_3 \cdot \left(\lambda_{1,i} \cdot (x_{A,i} - x_{T,i}) - y_{A,i} + (2\mathbf{k}_i - 1) \cdot y_{T,i}\right) = 0 \\\hline
3 & q_3 \cdot \left(\lambda_{2,i}^2 - x_{A,i-1} - x_{R,i} - x_{A,i}\right) = 0 \\\hline
3 & q_3 \cdot \left(\lambda_{2,i} \cdot (x_{A,i} - x_{A,i-1}) - y_{A,i} - y_{A,i-1}^\text{witnessed}\right) = 0 \\\hline
\end{array}
$$
其中
$$
\begin{aligned}
x_{R,i} &= \lambda_{1,i}^2 - x_{A,i} - x_T, \\
y_{A,i} &= \frac{(\lambda_{1,i} + \lambda_{2,i}) \cdot (x_{A,i} - (\lambda_{1,i}^2 - x_{A,i} - x_T))}{2},\\
y_{A,i-1}^\text{witnessed} &\text{ 被见证。}
\end{aligned}
$$

## 完全加法

我们重用 [完全加法](addition.md#complete-addition) 约束来实现双加算法的最后 $c$ 轮。这需要每轮两行，因为每次完全加法需要 9 个建议列。在第 10 个建议列中，我们存储了正确实现双加算法所需的其他单元格：
- 基点的 $y$ 坐标，以便我们可以有条件地将其取反作为完全加法的输入；
- 运行和，我们在两行中而不是顺序地约束它。

### 布局

$$
\begin{array}{|c|c|c|c|c|c|c|c|c|c|c|}
a_0 & a_1 & a_2 & a_3 &    a_4    &   a_5    &   a_6   &   a_7    &   a_8    & a_9     & q_\texttt{mul\_decompose\_var} \\\hline
x_T & y_p & x_A & y_A & \lambda_1 & \alpha_1 & \beta_1 & \gamma_1 & \delta_1 & z_{i+1} & 0 \\\hline
x_A & y_A & x_q & y_q & \lambda_2 & \alpha_2 & \beta_2 & \gamma_2 & \delta_2 & y_T     & 1 \\\hline
    &     & x_r & y_r &           &          &         &          &          & z_i     & 0 \\\hline
\end{array}
$$

### 约束 <a name="complete-gate">

除了完全加法约束外，我们还定义了以下门：

$$
\begin{array}{|c|l|}
\hline
\text{度数} & \text{约束} \\\hline
  & q_\texttt{mul\_decompose\_var} \cdot \BoolCheck{\mathbf{k}_i} = 0 \\\hline
  & q_\texttt{mul\_decompose\_var} \cdot \Ternary{\mathbf{k}_i}{y_T - y_p}{y_T + y_p} = 0 \\\hline
\end{array}
$$
其中 $\mathbf{k}_i = \mathbf{z}_{i} - 2\mathbf{z}_{i+1}$。

## LSB

### 布局

$$
\begin{array}{|c|c|c|c|c|c|c|c|c|c|c|}
a_0 & a_1 & a_2 & a_3 &   a_4   &  a_5   &  a_6  &  a_7   &  a_8   & a_9 & q_\texttt{mul\_lsb} \\\hline
x_p & y_p & x_A & y_A & \lambda & \alpha & \beta & \gamma & \delta & z_1 & 1 \\\hline
x_T & y_T & x_r & y_r &         &        &       &        &        & z_0 & 0 \\\hline
\end{array}
$$

### 约束 <a name="lsb-gate">

$$
\begin{array}{|c|l|}
\hline
\text{度数} & \text{约束} \\\hline
  & q_\texttt{mul\_lsb} \cdot \BoolCheck{\mathbf{k}_0} = 0 \\\hline
  & q_\texttt{mul\_lsb} \cdot \Ternary{\mathbf{k}_0}{x_p}{x_p - x_T} = 0 \\\hline
  & q_\texttt{mul\_lsb} \cdot \Ternary{\mathbf{k}_0}{y_p}{y_p + y_T} = 0 \\\hline
\end{array}
$$
其中 $\mathbf{k}_0 = \mathbf{z}_0 - 2\mathbf{z}_1$。

## 溢出检查

对于任何 $i \geq 1$，$\mathbf{z}_i$ 不会溢出，因为它是仅到 $2^{n-1} = 2^{253}$ 的加权和，这小于 $p$（也小于 $q$）。

然而，$\mathbf{z}_0 = \alpha + t_q$ *可以* 溢出 $[0, p)$。

> 注意：对于全宽度标量乘法，可能无法在基域中表示 $\mathbf{z}_0$（例如，当基域是 Pasta 的 $\mathbb{F}_p$ 且 $p < q$ 时）。
> 在这种情况下，我们需要特殊处理提到 $\mathbf{z}_0$ 的行，以便它对于我们用于全宽度标量的任何表示都是正确的。
> 我们对 $k$ 的表示将是 $(\mathbb{k}_{254}, k' = k - 2^{254} \cdot \mathbb{k}_{254})$。
> 我们将使用 $k'$ 代替 $\alpha + t_q$ 作为 $\mathbf{z}_0$，将 $k'$ 约束为 254 位，以便它适合 $\mathbb{F}_p$ 元素。
> 然后我们只需将下面的参数推广到适用于 $k' \in [0, 2 \cdot t_q)$（因为 $\alpha + t_q$ 的最大值是 $q - 1 + t_q = 2^{254} + t_q - 1 + t_q$）。

由于溢出只能发生在约束 $\mathbf{z}_0 = 2 \cdot \mathbf{z}_1 + \mathbf{k}_0$ 的最后一步，我们有 $\mathbf{z}_0 = k \pmod{p}$。然后只需检查 $\mathbf{z}_0 = \alpha + t_q \pmod{p}$（以便 $k = \alpha + t_q \pmod{p}$）和 $k \in [t_q, p + t_q)$。这些条件共同意味着 $k = \alpha + t_q$ 作为整数，因此 $2^{254} + k = \alpha \pmod{q}$ 符合要求。

> 注意：位 $\mathbf{k}_{254..0}$ 不代表模 $q$ 减少的值，而是未减少的 $\alpha + t_q$ 的表示。

### 优化检查 $k \in [t_q, p + t_q)$

由于 $t_p + t_q < 2^{130}$（如果 $p$ 和 $q$ 交换，这也成立），我们有 $$[t_q, p + t_q) = [t_q, t_q + 2^{130}) \;\cup\; [2^{130}, 2^{254}) \;\cup\; \big([2^{254}, 2^{254} + 2^{130}) \;\cap\; [p + t_q - 2^{130}, p + t_q)\big).$$

我们可以假设 $k = \alpha + t_q \pmod{p}$。

（这对于 Orchard 中的变基标量乘法成立，因为我们知道 $\alpha < p$。如果我们将 $p$ 和 $q$ 交换，使得 $p > q$，这也成立。
对于全宽度标量 $\alpha \geq p$ 且 $p < q$ 时，这不成立。）

因此，
$\begin{array}{rcl}
k \in [t_q, p + t_q) &\Leftrightarrow& \big(k \in [t_q, t_q + 2^{130}) \;\vee\; k \in [2^{130}, 2^{254})\big) \;\vee\; \\
                     &               & \big(k \in [2^{254}, 2^{254} + 2^{130}) \;\wedge\; k \in [p + t_q - 2^{130}, p + t_q)\big) \\
                     \\
                     &\Leftrightarrow& \big(\mathbf{k}_{254} = 0 \implies (k \in [t_q, t_q + 2^{130}) \;\vee\; k \in [2^{130}, 2^{254}))\big) \;\wedge \\
                     &               & \big(\mathbf{k}_{254} = 1 \implies (k \in [2^{254}, 2^{254} + 2^{130}) \;\wedge\; k \in [p + t_q - 2^{130}, p + t_q))\big) \\
                     \\
                     &\Leftrightarrow& \big(\mathbf{k}_{254} = 0 \implies (\alpha \in [0, 2^{130}) \;\vee\; k \in [2^{130}, 2^{254}))\big) \;\wedge \\
                     &               & \big(\mathbf{k}_{254} = 1 \implies (k \in [2^{254}, 2^{254} + 2^{130}) \;\wedge\; (\alpha + 2^{130}) \bmod p \in [0, 2^{130}))\big) \;\;Ⓐ
\end{array}$

> 给定 $k \in [2^{254}, 2^{254} + 2^{130})$，我们证明 $k \in [p + t_q - 2^{130}, p + t_q)$ 和 $(\alpha + 2^{130}) \bmod p \in [0, 2^{130})$ 的等价性如下：
> * 将范围偏移 $2^{130} - p - t_q$，得到 $k + 2^{130} - p - t_q \in [0, 2^{130})$；
> * 观察到 $k + 2^{130} - p - t_q$ 保证在 $[2^{130} - t_p - t_q, 2^{131} - t_p - t_q)$ 范围内，因此不会在模 $p$ 下溢出或下溢；
> * 利用 $k = \alpha + t_q \pmod{p}$ 的事实，观察到 $(k + 2^{130} - p - t_q) \bmod p = (\alpha + t_q + 2^{130} - p - t_q) \bmod p = (\alpha + 2^{130}) \bmod p$。
>
> （我们可以通过另一种方式看到这是正确的，观察到它检查 $\alpha \bmod p \in [p - 2^{130}, p)$，因此上限如我们所期望的那样对齐。）

现在，我们可以从 $Ⓐ$ 继续优化：

$\begin{array}{rcl}
k \in [t_q, p + t_q) &\Leftrightarrow& \big(\mathbf{k}_{254} = 0 \implies (\alpha \in [0, 2^{130}) \;\vee\; k \in [2^{130}, 2^{254})\big) \;\wedge \\
                     &               & \big(\mathbf{k}_{254} = 1 \implies (k \in [2^{254}, 2^{254} + 2^{130}) \;\wedge\; (\alpha + 2^{130}) \bmod p \in [0, 2^{130}))\big) \\
                     \\
                     &\Leftrightarrow& \big(\mathbf{k}_{254} = 0 \implies (\alpha \in [0, 2^{130}) \;\vee\; \mathbf{k}_{253..130} \text{ 不全为 } 0)\big) \;\wedge \\
                     &               & \big(\mathbf{k}_{254} = 1 \implies (\mathbf{k}_{253..130} \text{ 全为 } 0 \;\wedge\; (\alpha + 2^{130}) \bmod p \in [0, 2^{130}))\big)
\end{array}$

将 $\mathbf{k}_{253..130}$ 约束为全 $0$ 或不全 $0$ 几乎可以“免费”实现，如下所示。

回想 $\mathbf{z}_i = \sum_{h=i}^{n} (\mathbf{k}_{h} \cdot 2^{h-i})$，因此我们有：

$\begin{array}{rcl}
                                 \mathbf{z}_{130} &=& \sum_{h=130}^{254} (\mathbf{k}_h \cdot 2^{h-130}) \\
                                 \mathbf{z}_{130} &=& \mathbf{k}_{254} \cdot 2^{254-130} + \sum_{h=130}^{253} (\mathbf{k}_h \cdot 2^{h-130}) \\
\mathbf{z}_{130} - \mathbf{k}_{254} \cdot 2^{124} &=& \sum_{h=130}^{253} (\mathbf{k}_h \cdot 2^{h-130})
\end{array}$

因此，$\mathbf{k}_{253..130}$ 全为 $0$ 当且仅当 $\mathbf{z}_{130} = \mathbf{k}_{254} \cdot 2^{124}$。

最后，我们可以通过检查 $(\alpha + \mathbf{k}_{254} \cdot 2^{130}) \bmod p \in [0, 2^{130})$ 来合并 $\mathbf{k}_{254} = 0$ 和 $\mathbf{k}_{254} = 1$ 情况的 $130$ 位分解。

### 溢出检查约束

设 $s = \alpha + \mathbf{k}_{254} \cdot 2^{130}$。溢出检查的约束为：

$$
\begin{aligned}
\mathbf{z}_0 &= \alpha + t_q \pmod{p} \\
\mathbf{k}_{254} = 1 \implies \big(\mathbf{z}_{130} &= 2^{124} \;\wedge\; s \bmod p \in [0, 2^{130})\big) \\
\mathbf{k}_{254} = 0 \implies \big(\mathbf{z}_{130} &\neq 0 \;\vee\; s \bmod p \in [0, 2^{130})\big)
\end{aligned}
$$

定义 $\mathsf{inv0}(x) = \begin{cases} 0, &\text{如果 } x = 0 \\ 1/x, &\text{否则。} \end{cases}$

见证 $\eta = \mathsf{inv0}(\mathbf{z}_{130})$，并将 $s \bmod p$ 分解为 $\mathbf{s}_{129..0}$。

所需的门为：

$$
\begin{array}{|c|l|}
\hline
\text{度数} & \text{约束} \\\hline
2 & q_\texttt{mul\_overflow} \cdot \left(s - (\alpha + \mathbf{k}_{254} \cdot 2^{130})\right) = 0 \\\hline
2 & q_\texttt{mul\_overflow} \cdot \left(\mathbf{z}_0 - \alpha - t_q\right) = 0 \\\hline
3 & q_\texttt{mul\_overflow} \cdot \left(\mathbf{k}_{254} \cdot (\mathbf{z}_{130} - 2^{124})\right) = 0 \\\hline
3 & q_\texttt{mul\_overflow} \cdot \left(\mathbf{k}_{254} \cdot (s - \sum\limits_{i=0}^{129} 2^i \cdot \mathbf{s}_i)/2^{130}\right) = 0 \\\hline
5 & q_\texttt{mul\_overflow} \cdot \left((1 - \mathbf{k}_{254}) \cdot (1 - \mathbf{z}_{130} \cdot \eta) \cdot (s - \sum\limits_{i=0}^{129} 2^i \cdot \mathbf{s}_i)/2^{130}\right) = 0 \\\hline
\end{array}
$$
其中 $(s - \sum\limits_{i=0}^{129} 2^i \cdot \mathbf{s}_i)/2^{130}$ 可以通过另一个运行和计算。注意，$1/2^{130}$ 的因子对约束没有影响，因为 RHS 为零。

#### 运行和范围检查
我们在电路中使用 $10$ 位 [查找范围检查](../decomposition.md#lookup-decomposition) 来减去 $\mathbf{s}$ 的低 $130$ 位。范围检查减去 $\mathbf{s}$ 的前 $13 \cdot 10$ 位，并将结果右移，得到 $(s - \sum\limits_{i=0}^{129} 2^i \cdot \mathbf{s}_i)/2^{130}.$

## 溢出检查（通用）
回想我们定义了 $\mathbf{z}_j = \sum_{i=j}^n (\mathbf{k}_i \cdot 2^{i−j})$，其中 $n = 254$。

对于任何 $j \geq 1$，$\mathbf{z}_j$ 不会溢出，因为它是仅到 $2^{n-j}$ 的加权和。当 $n = 254$ 且 $j = 1$ 时，这个和最多为 $2^{254} - 1$，这小于 $p$（也小于 $q$）。

然而，对于全宽度标量乘法，可能无法在基域中表示 $\mathbf{z}_0$（例如，当基域是 Pasta 的 $\mathbb{F}_p$ 且 $p < q$ 时）。在这种情况下，$\mathbf{z}_0 = \alpha + t_q$ *可以* 溢出 $[0, p)$。

因此，我们需要特殊处理提到 $\mathbf{z}_0$ 的行，以便它对于我们用于全宽度标量的任何表示都是正确的。

我们对 $k$ 的表示将是 $(\mathbf{k}_{254}, k' = k - 2^{254} \cdot \mathbf{k}_{254})$。我们将使用 $k'$ 代替 $\alpha + t_q$ 作为 $\mathbf{z}_0$，将 $k'$ 约束为 254 位，以便它适合 $\mathbb{F}_p$ 元素。

然后我们只需将 [Orchard 电路中用于变基标量乘法的溢出检查](https://zcash.github.io/halo2/design/gadgets/ecc/var-base-scalar-mul.html#overflow-check) 推广到适用于 $k' \in [0, 2 \cdot t_q)$（因为 $\alpha + t_q$ 的最大值是 $q - 1 + t_q = 2^{254} + t_q - 1 + t_q$）。


> 注意：位 $\mathbf{k}_{254..0}$ 不代表模 $q$ 减少的值，而是未减少的 $\alpha + t_q$ 的表示。

溢出只能发生在约束 $\mathbf{z}_0 = 2 \cdot \mathbf{z}_1 + \mathbf{k}_0$ 的最后一步，并且仅当 $\mathbf{z}_1$ 设置了权重为 $2^{253}$ 的位（即 $\mathbf{k}_{254} = 1$）时。如果我们改为设置 $\mathbf{z}_0 = 2 \cdot \mathbf{z}_1 - 2^{254} \cdot \mathbf{k}_{254} + \mathbf{k}_0$，现在 $\mathbf{z}_0$ 不会溢出，并且应该等于 $k'$。

然后只需检查 $\mathbf{z}_0 + 2^{254} \cdot \mathbf{k}_{254} = \alpha + t_q$ 作为整数，其中 $\alpha \in [0, q)$。

将 $\alpha$ 表示为 $2^{254} \cdot \boldsymbol\alpha_{254} + 2^{253} \cdot \boldsymbol\alpha_{253} + \alpha''$，其中我们约束 $\alpha'' \in [0, 2^{253})$ 且 $\boldsymbol\alpha_{253}$ 和 $\boldsymbol\alpha_{254}$ 为布尔值。为了使这是一个规范表示，我们还需要 $\boldsymbol \alpha_{254} = 1 \implies (\boldsymbol \alpha_{253} = 0 \;\wedge\; \alpha'' \in [0, t_q))$。

设 $\alpha' = 2^{253} \cdot \boldsymbol\alpha_{253} + \alpha''$。

如果 $\boldsymbol\alpha_{254} = 1$：
* 约束 $\mathbf{k}_{254} = 1$ 且 $\mathbf{z}_0 = \alpha' + t_q$。这不会溢出，因为在这种情况下 $\alpha' \in [0, t_q)$，因此 $\mathbf{z}_0 \in [0, 2 \cdot t_q)$。

如果 $\boldsymbol\alpha_{254} = 0$：
* 我们应该有 $\mathbf{k}_{254} = 1$ 当且仅当 $\alpha' \in [2^{254} - t_q, 2^{254})$，即见证 $\mathbf{k}_{254}$ 为布尔值，然后
  * 如果 $\mathbf{k}_{254} = 0$，则约束 $\alpha' \not\in [2^{254} - t_q, 2^{254})$。
    * 这可以通过约束 $\boldsymbol\alpha_{253} = 0$ 或 $\alpha'' + t_q \in [0, 2^{253})$ 来实现。（$\alpha'' + t_q$ 不会溢出。）
  * 如果 $\mathbf{k}_{254} = 1$，则约束 $\alpha' \in [2^{254} - t_q, 2^{254})$。
    * 这可以通过约束 $\alpha' - (2^{254} - t_q) \in [0, 2^{130})$ 和 $\alpha' - 2^{254} + 2^{130} \in [0, 2^{130})$ 来实现。

### 溢出检查约束（通用）

将 $\alpha$ 表示为 $2^{254} \cdot \boldsymbol\alpha_{254} + 2^{253} \cdot \boldsymbol\alpha_{253} + \alpha''$，如上所述。

溢出检查的约束为：

$$
\begin{aligned}
&\alpha'' \in [0, 2^{253}) \\
&\boldsymbol\alpha_{253} \in \{0,1\} \\
&\boldsymbol\alpha_{254} \in \{0,1\} \\
&\mathbf{k}_{254} \in \{0,1\} \\
\boldsymbol \alpha_{254} = 1 \implies &\boldsymbol \alpha_{253} = 0 \\
\boldsymbol \alpha_{254} = 1 \implies &\alpha'' \in [0, 2^{130}) \;❁ \\
\boldsymbol \alpha_{254} = 1 \implies &\alpha'' + 2^{130} - t_q \in [0, 2^{130}) \;❁ \\
\boldsymbol\alpha_{254} = 1 \implies &\mathbf{k}_{254} = 1 \\
\boldsymbol\alpha_{254} = 1 \implies &\mathbf{z}_0 = \alpha' + t_q \\
\boldsymbol\alpha_{254} = 0 \;\wedge\; \boldsymbol\alpha_{253} = 1 \;\wedge\; \mathbf{k}_{254} = 0 &\implies \alpha'' + t_q \in [0, 2^{253}) \\
\boldsymbol\alpha_{254} = 0 \;\wedge\; \mathbf{k}_{254} = 1 &\implies \alpha' - 2^{254} + t_q \in [0, 2^{130}) \;❁ \\
\boldsymbol\alpha_{254} = 0 \;\wedge\; \mathbf{k}_{254} = 1 &\implies \alpha' - 2^{254} + 2^{130} \in [0, 2^{130}) \;❁ \\
\end{aligned}
$$

注意，标记为 $❁$ 的四个 130 位约束分为两组，发生在不相交的情况下。因此，我们可以使用一个新的见证变量 $u$ 将它们合并为两个 130 位约束；另一个约束始终在 $u + 2^{130} - t_q$ 上：

$$
\begin{aligned}
\boldsymbol \alpha_{254} = 1 \implies &u = \alpha'' \\
\boldsymbol\alpha_{254} = 0 \;\wedge\; \mathbf{k}_{254} = 1 \implies &u = \alpha' - 2^{254} + t_q \\
&u \in [0, 2^{130}) \\
&u + 2^{130} - t_q \in [0, 2^{130}) \\
\end{aligned}
$$

（在 $\boldsymbol\alpha_{254} = 0 \;\wedge\; \mathbf{k}_{254} = 0$ 的情况下，$u$ 不受约束，可以见证为 $0$。）

### 成本

* 25 个 10 位和 1 个 3 位范围检查，将 $\alpha''$ 约束为 253 位；
* 25 个 10 位和 1 个 3 位范围检查，当 $\boldsymbol\alpha_{254} = 0 \;\wedge\; \boldsymbol\alpha_{253} = 1 \;\wedge\; \mathbf{k}_{254} = 0$ 时，将 $\alpha'' + t_q$ 约束为 253 位；
* 两次 13 个 10 位范围检查。
