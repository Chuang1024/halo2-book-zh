We will use formulae for curve arithmetic using affine coordinates on short Weierstrass curves,
derived from section 4.1 of [Hüseyin Hışıl's thesis](https://core.ac.uk/download/pdf/10898289.pdf).

## Incomplete addition

- Inputs: $P = (x_p, y_p), Q = (x_q, y_q)$
- Output: $R = P \;⸭\; Q = (x_r, y_r)$

The formulae from Hışıl's thesis are:

- $x_3 = \left(\frac{y_1 - y_2}{x_1 - x_2}\right)^2 - x_1 - x_2$
- $y_3 = \frac{y_1 - y_2}{x_1 - x_2} \cdot (x_1 - x_3) - y_1.$

Rename $(x_1, y_1)$ to $(x_q, y_q)$, $(x_2, y_2)$ to $(x_p, y_p)$, and $(x_3, y_3)$ to $(x_r, y_r)$, giving

- $x_r = \left(\frac{y_q - y_p}{x_q - x_p}\right)^2 - x_q - x_p$
- $y_r = \frac{y_q - y_p}{x_q - x_p} \cdot (x_q - x_r) - y_q$

which is equivalent to

- $x_r + x_q + x_p = \left(\frac{y_p - y_q}{x_p - x_q}\right)^2$
- $y_r + y_q = \frac{y_p - y_q}{x_p - x_q} \cdot (x_q - x_r).$

Assuming $x_p \neq x_q$, we have

$
\begin{array}{lrrll}
&& x_r + x_q + x_p &=& \left(\frac{y_p - y_q}{x_p - x_q}\right)^2 \\[1.2ex]
&\Longleftrightarrow &(x_r + x_q + x_p) \cdot (x_p - x_q)^2 &=& (y_p - y_q)^2 \\[1ex]
&\Longleftrightarrow &(x_r + x_q + x_p) \cdot (x_p - x_q)^2 - (y_p - y_q)^2 &=& 0 \\[1.5ex]
\text{and} \\
&&y_r + y_q &=& \frac{y_p - y_q}{x_p - x_q} \cdot (x_q - x_r) \\[0.8ex]
&\Longleftrightarrow &(y_r + y_q) \cdot (x_p - x_q) &=& (y_p - y_q) \cdot (x_q - x_r) \\[1ex]
&\Longleftrightarrow &(y_r + y_q) \cdot (x_p - x_q) - (y_p - y_q) \cdot (x_q - x_r) &=& 0.
\end{array}
$

So we get the constraints:
- $(x_r + x_q + x_p) \cdot (x_p - x_q)^2 - (y_p - y_q)^2 = 0$
  - Note that this constraint is unsatisfiable for $P \;⸭\; (-P)$ (when $P \neq \mathcal{O}$),
    and so cannot be used with arbitrary inputs.
- $(y_r + y_q) \cdot (x_p - x_q) - (y_p - y_q) \cdot (x_q - x_r) = 0.$

### Constraints <a name="incomplete-addition-constraints">

$$
\begin{array}{|c|l|}
\hline
\text{Degree} & \text{Constraint} \\\hline
4 & q_\text{add-incomplete} \cdot \left( (x_r + x_q + x_p) \cdot (x_p - x_q)^2 - (y_p - y_q)^2 \right) = 0 \\\hline
3 & q_\text{add-incomplete} \cdot \left( (y_r + y_q) \cdot (x_p - x_q) - (y_p - y_q) \cdot (x_q - x_r) \right) = 0 \\\hline
\end{array}
$$

## Complete addition

$\hspace{1em} \begin{array}{rcll}
\mathcal{O} &+& \mathcal{O} &= \mathcal{O} \\[0.4ex]
\mathcal{O} &+& (x_q, y_q)  &= (x_q, y_q) \\[0.4ex]
 (x_p, y_p) &+& \mathcal{O} &= (x_p, y_p) \\[0.6ex]
   (x, y)   &+& (x, y)      &= [2] (x, y) \\[0.6ex]
   (x, y)   &+& (x, -y)     &= \mathcal{O} \\[0.4ex]
 (x_p, y_p) &+& (x_q, y_q)  &= (x_p, y_p) \;⸭\; (x_q, y_q), \text{ if } x_p \neq x_q.
\end{array}$

Suppose that we represent $\mathcal{O}$ as $(0, 0)$. ($0$ is not an $x$-coordinate of a valid point because we would need $y^2 = x^3 + 5$, and $5$ is not square in $\mathbb{F}_q$. Also $0$ is not a $y$-coordinate of a valid point because $-5$ is not a cube in $\mathbb{F}_q$.)

$$
\begin{aligned}
P + Q &= R\\
(x_p, y_p) + (x_q, y_q) &= (x_r, y_r) \\
                \lambda &= \frac{y_q - y_p}{x_q - x_p} \\
                    x_r &= \lambda^2 - x_p - x_q \\
                    y_r &= \lambda(x_p - x_r) - y_p
\end{aligned}
$$

For the doubling case, Hışıl's thesis tells us that $\lambda$ has to
instead be computed as $\frac{3x^2}{2y}$.

Define $\mathsf{inv0}(x) = \begin{cases} 0, &\text{if } x = 0 \\ 1/x, &\text{otherwise.} \end{cases}$

Witness $\alpha, \beta, \gamma, \delta, \lambda$ where:

$\hspace{1em}
\begin{array}{rl}
\alpha  \,\,=&\!\! \mathsf{inv0}(x_q - x_p) \\[0.4ex]
\beta   \,\,=&\!\! \mathsf{inv0}(x_p) \\[0.4ex]
\gamma  \,\,=&\!\! \mathsf{inv0}(x_q) \\[0.4ex]
\delta  \,\,=&\!\! \begin{cases}
                     \mathsf{inv0}(y_q + y_p), &\text{if } x_q = x_p \\
                     0, &\text{otherwise}
                   \end{cases} \\[2.5ex]
\lambda \,\,=&\!\! \begin{cases}
                     \frac{y_q - y_p}{x_q - x_p}, &\text{if } x_q \neq x_p \\[1.2ex]
                     \frac{3{x_p}^2}{2y_p} &\text{if } x_q = x_p \wedge y_p \neq 0 \\[0.8ex]
                     0, &\text{otherwise.}
                   \end{cases}
\end{array}
$

### Constraints <a name="complete-addition-constraints">

$$
\begin{array}{|c|lcl|l|}
\hline
\text{Degree} & \text{Constraint}\hspace{7em} &&& \text{Meaning} \\\hline
4 & q_\mathit{add} \cdot (x_q - x_p) \cdot ((x_q - x_p) \cdot \lambda - (y_q - y_p)) &=& 0 & x_q \neq x_p \implies \lambda = \frac{y_q - y_p}{x_q - x_p} \\\hline \\[-2.3ex]
5 & q_\mathit{add} \cdot (1 - (x_q - x_p) \cdot \alpha) \cdot \left(2y_p \cdot \lambda - 3{x_p}^2\right) &=& 0 & \begin{cases} x_q = x_p \wedge y_p \neq 0 \implies \lambda = \frac{3{x_p}^2}{2y_p} \\ x_q = x_p \wedge y_p = 0 \implies x_p = 0 \end{cases} \\\hline
6 & q_\mathit{add} \cdot x_p \cdot x_q \cdot (x_q - x_p) \cdot (\lambda^2 - x_p - x_q - x_r) &=& 0 & x_p \neq 0 \wedge x_q \neq 0 \wedge x_q \neq x_p \implies x_r = \lambda^2 - x_p - x_q \\
6 & q_\mathit{add} \cdot x_p \cdot x_q \cdot (x_q - x_p) \cdot (\lambda \cdot (x_p - x_r) - y_p - y_r) &=& 0 & x_p \neq 0 \wedge x_q \neq 0 \wedge x_q \neq x_p \implies y_r = \lambda \cdot (x_p - x_r) - y_p \\
6 & q_\mathit{add} \cdot x_p \cdot x_q \cdot (y_q + y_p) \cdot (\lambda^2 - x_p - x_q - x_r) &=& 0 & x_p \neq 0 \wedge x_q \neq 0 \wedge y_q \neq -y_p \implies x_r = \lambda^2 - x_p - x_q \\
6 & q_\mathit{add} \cdot x_p \cdot x_q \cdot (y_q + y_p) \cdot (\lambda \cdot (x_p - x_r) - y_p - y_r) &=& 0 & x_p \neq 0 \wedge x_q \neq 0 \wedge y_q \neq -y_p \implies y_r = \lambda \cdot (x_p - x_r) - y_p \\\hline
4 & q_\mathit{add} \cdot (1 - x_p \cdot \beta) \cdot (x_r - x_q) &=& 0 & x_p = 0 \implies x_r = x_q \\
4 & q_\mathit{add} \cdot (1 - x_p \cdot \beta) \cdot (y_r - y_q) &=& 0 & x_p = 0 \implies y_r = y_q \\\hline
4 & q_\mathit{add} \cdot (1 - x_q \cdot \gamma) \cdot (x_r - x_p) &=& 0 & x_q = 0 \implies x_r = x_p \\
4 & q_\mathit{add} \cdot (1 - x_q \cdot \gamma) \cdot (y_r - y_p) &=& 0 & x_q = 0 \implies y_r = y_p \\\hline
4 & q_\mathit{add} \cdot (1 - (x_q - x_p) \cdot \alpha - (y_q + y_p) \cdot \delta) \cdot x_r &=& 0 & x_q = x_p \wedge y_q = -y_p \implies x_r = 0 \\
4 & q_\mathit{add} \cdot (1 - (x_q - x_p) \cdot \alpha - (y_q + y_p) \cdot \delta) \cdot y_r &=& 0 & x_q = x_p \wedge y_q = -y_p \implies y_r = 0 \\\hline
\end{array}
$$

Max degree: 6

### Analysis of constraints
$$
\begin{array}{rl}
1. & (x_q - x_p) \cdot ((x_q - x_p) \cdot \lambda - (y_q - y_p)) = 0 \\
   & \\
   & \begin{aligned}
         \text{At least one of } &x_q - x_p = 0 \\
                      \text{or } &(x_q - x_p) \cdot \lambda - (y_q - y_p) = 0 \\
       \end{aligned} \\
   & \text{must be satisfied for the constraint to be satisfied.} \\
   & \\
   & \text{If } x_q - x_p \neq 0, \text{ then } (x_q - x_p) \cdot \lambda - (y_q - y_p) = 0, \text{ and} \\
   & \text{by rearranging both sides we get } \lambda = (y_q - y_p) / (x_q - x_p). \\
   & \\
   & \text{Therefore:} \\
   & \hspace{2em} x_q \neq x_p \implies \lambda = (y_q - y_p) / (x_q - x_p).\\
   & \\
2. & (1 - (x_q - x_p) \cdot \alpha) \cdot (2y_p \cdot \lambda - 3x_p^2) = 0 \\
   & \\
   & \begin{aligned}
         \text{At least one of } &(1 - (x_q - x_p) \cdot \alpha) = 0 \\
                      \text{or } &(2y_p \cdot \lambda - 3x_p^2) = 0
       \end{aligned} \\
   & \text{must be satisfied for the constraint to be satisfied.} \\
   & \\
   & \text{If } x_q = x_p, \text{ then } 1 - (x_q - x_p) \cdot \alpha = 0 \text{ has no solution for } \alpha, \\
   & \text{so it must be that } 2y_p \cdot \lambda - 3x_p^2 = 0. \\
   & \\
   & \text{If } x_q = x_p \text{ and } y_p = 0 \text{ then } x_p = 0, \text{ and the constraint is satisfied.}\\
   & \\
   & \text{If } x_q = x_p \text{ and } y_p \neq 0 \text{ then by rearranging both sides} \\
   & \text{we get } \lambda = 3x_p^2 / 2y_p. \\
   & \\
   & \text{Therefore:} \\
   & \hspace{2em} (x_q = x_p) \wedge y_p \neq 0 \implies \lambda = 3x_p^2 / 2y_p. \\
   & \\
3.\text{ a)} & x_p \cdot x_q \cdot (x_q - x_p) \cdot (\lambda^2 - x_p - x_q - x_r) = 0 \\
  \text{ b)} & x_p \cdot x_q \cdot (x_q - x_p) \cdot (\lambda \cdot (x_p - x_r) - y_p - y_r) = 0 \\
  \text{ c)} & x_p \cdot x_q \cdot (y_q + y_p) \cdot (\lambda^2 - x_p - x_q - x_r) = 0 \\
  \text{ d)} & x_p \cdot x_q \cdot (y_q + y_p) \cdot (\lambda \cdot (x_p - x_r) - y_p - y_r) = 0 \\
   & \\
   & \begin{aligned}
       \text{At least one of } &x_p = 0 \\
                    \text{or } &x_p = 0 \\
                    \text{or } &(x_q - x_p) = 0 \\
                    \text{or } &(\lambda^2 - x_p - x_q - x_r) = 0 \\
     \end{aligned} \\
   & \text{must be satisfied for constraint (a) to be satisfied.} \\
   & \\
   & \text{If } x_p \neq 0 \wedge x_q \neq 0 \wedge x_q \neq x_p, \\[1.5ex]
   & \text{• Constraint (a) imposes that } x_r = \lambda^2 - x_p - x_q. \\
   & \text{• Constraint (b) imposes that } y_r = \lambda \cdot (x_p - x_r) - y_p. \\
   & \\
   & \text{If } x_p \neq 0 \wedge x_q \neq 0 \wedge y_q \neq -y_p, \\[1.5ex]
   & \text{• Constraint (c) imposes that } x_r = \lambda^2 - x_p - x_q. \\
   & \text{• Constraint (d) imposes that } y_r = \lambda \cdot (x_p - x_r) - y_p. \\
   & \\
   & \text{Therefore:} \\
   & \begin{aligned}
                 &(x_p \neq 0) \wedge (x_q \neq 0) \wedge ((x_q \neq x_p) \vee (y_q \neq -y_p)) \\
        \implies &(x_r = \lambda^2 - x_p - x_q) \wedge (y_r = \lambda \cdot (x_p - x_r) - y_p).
     \end{aligned} \\
   & \\
4.\text{ a)} & (1 - x_p \cdot \beta) \cdot (x_r - x_q) = 0 \\
  \text{ b)} & (1 - x_p \cdot \beta) \cdot (y_r - y_q) = 0 \\
  & \\
  & \begin{aligned}
      \text{At least one of } 1 - x_p \cdot \beta &= 0 \\
                             \text{or } x_r - x_q &= 0
    \end{aligned} \\
  & \text{must be satisfied for constraint (a) to be satisfied.} \\
  & \\
  & \text{If } x_p = 0 \text{ then } 1 - x_p \cdot \beta = 0 \text{ has no solutions for } \beta, \\
  & \text{and so it must be that } x_r - x_q = 0. \\
  & \\
  & \text{Similarly, constraint (b) imposes that if } x_p = 0 \\
  & \text{then } y_r - y_q = 0. \\
  & \\
  & \text{Therefore:} \\
  & \hspace{2em} x_p = 0 \implies (x_r, y_r) = (x_q, y_q). \\
  & \\
5.\text{ a)} & (1 - x_q \cdot \gamma) \cdot (x_r - x_p) = 0 \\
  \text{ b)} & (1 - x_q \cdot \gamma) \cdot (y_r - y_p) = 0 \\
  & \\
  & \begin{aligned}
      \text{At least one of } 1 - x_q \cdot \gamma &= 0 \\
                             \text{or } x_r - x_p &= 0
    \end{aligned} \\
  & \text{must be satisfied for constraint (a) to be satisfied.} \\
  & \\
  & \text{If } x_q = 0 \text{ then } 1 - x_q \cdot \gamma = 0 \text{ has no solutions for } \gamma, \\
  & \text{and so it must be that } x_r - x_p = 0. \\
  & \\
  & \text{Similarly, constraint (b) imposes that if } x_q = 0 \\
  & \text{then } y_r - y_p = 0. \\
  & \\
  & \text{Therefore:} \\
  & \hspace{2em} x_q = 0 \implies (x_r, y_r) = (x_p, y_p). \\
  & \\
6.\text{ a)} & (1 - (x_q - x_p) \cdot \alpha - (y_q + y_p) \cdot \delta) \cdot x_r = 0 \\
  \text{ b)} & (1 - (x_q - x_p) \cdot \alpha - (y_q + y_p) \cdot \delta) \cdot y_r = 0 \\
  & \\
  & \begin{aligned}
      \text{At least one of } &1 - (x_q - x_p) \cdot \alpha - (y_q + y_p) \cdot \delta = 0 \\
                   \text{or } &x_r = 0
    \end{aligned} \\
  & \text{must be satisfied for constraint (a) to be satisfied,} \\
  & \text{and similarly replacing } x_r \text{ by } y_r. \\
  & \\
  & \text{If } x_r \neq 0 \text{ or } y_r \neq 0, \text{ then it must be that } 1 - (x_q - x_p) \cdot \alpha - (y_q + y_p) \cdot \delta = 0. \\
  & \\
  & \text{However, if } x_q = x_p \wedge y_q = -y_p, \text{ then there are no solutions for } \alpha \text { and } \delta. \\
  & \\
  & \text{Therefore: } \\
  & \hspace{2em} x_q = x_p \wedge y_q = -y_p \implies (x_r, y_r) = (0, 0).
\end{array}
$$

#### Propositions:

$
\begin{array}{cl}
(1)& x_q \neq x_p \implies \lambda = (y_q - y_p) / (x_q - x_p) \\[0.8ex]
(2)& (x_q = x_p) \wedge y_p \neq 0 \implies \lambda = 3x_p^2 / 2y_p \\[0.8ex]
(3)& (x_p \neq 0) \wedge (x_q \neq 0) \wedge ((x_q \neq x_p) \vee (y_q \neq -y_p)) \\[0.4ex]
    &\implies (x_r = \lambda^2 - x_p - x_q) \wedge (y_r = \lambda \cdot (x_p - x_r) - y_p) \\[0.8ex]
(4)& x_p = 0 \implies (x_r, y_r) = (x_q, y_q) \\[0.8ex]
(5)& x_q = 0 \implies (x_r, y_r) = (x_p, y_p) \\[0.8ex]
(6)& x_q = x_p \wedge y_q = -y_p \implies (x_r, y_r) = (0, 0)
\end{array}
$

#### Cases:

$(x_p, y_p) + (x_q, y_q) = (x_r, y_r)$

Note that we rely on the fact that $0$ is not a valid $x$-coordinate or $y$-coordinate of a
point on the Pallas curve other than $\mathcal{O}$.

* $(0, 0) + (0, 0)$
    - Completeness:

        $
        \begin{array}{cl}
        (1)&\text{holds because } x_q = x_p \\
        (2)&\text{holds because } y_p = 0 \\
        (3)&\text{holds because } x_p = 0 \\
        (4)&\text{holds because } (x_r, y_r) = (x_q, y_q) = (0, 0) \\
        (5)&\text{holds because } (x_r, y_r) = (x_p, y_p) = (0, 0) \\
        (6)&\text{holds because } (x_r, y_r) = (0, 0). \\
        \end{array}
        $

    - Soundness: $(x_r, y_r) = (0, 0)$ is the only solution to $(6).$

* $(x, y) + (0, 0)$ for $(x, y) \neq (0, 0)$
    - Completeness:

        $
        \begin{array}{cl}
        (1)&\text{holds because } x_q \neq x_p, \text{ therefore } \lambda = (y_q - y_p) / (x_q - x_p) \text{ is a solution} \\
        (2)&\text{holds because } x_q \neq x_p, \text{ therefore } \alpha = (x_q - x_p)^{-1} \text{ is a solution} \\
        (3)&\text{holds because } x_q = 0 \\
        (4)&\text{holds because } x_p \neq 0, \text{ therefore } \beta = x_p^{-1} \text{ is a solution} \\
        (5)&\text{holds because } (x_r, y_r) = (x_p, y_p) \\
        (6)&\text{holds because } x_q \neq x_p, \text{ therefore } \alpha = (x_q - x_p)^{-1} \text{ and } \delta = 0 \text{ is a solution.}
        \end{array}
        $

    - Soundness: $(x_r, y_r) = (x_p, y_p)$ is the only solution to $(5).$

* $(0, 0) + (x, y)$ for $(x, y) \neq (0, 0)$
    - Completeness:

        $
        \begin{array}{cl}
        (1)&\text{holds because } x_q \neq x_p, \text{ therefore } \lambda = (y_q - y_p) / (x_q - x_p) \text{ is a solution} \\
        (2)&\text{holds because } x_q \neq x_p, \text{ therefore } \alpha = (x_q - x_p)^{-1} \text{ is a solution} \\
        (3)&\text{holds because } x_p = 0 \\
        (4)&\text{holds because } x_p = 0 \text{ only when } (x_r, y_r) = (x_q, y_q) \\
        (5)&\text{holds because } x_q \neq 0, \text{ therefore } \gamma = x_q^{-1} \text{ is a solution}\\
        (6)&\text{holds because } x_q \neq x_p, \text{ therefore } \alpha = (x_q - x_p)^{-1} \text{ and } \delta = 0 \text{ is a solution.}
        \end{array}
        $

    - Soundness: $(x_r, y_r) = (x_q, y_q)$ is the only solution to $(4).$

* $(x, y) + (x, y)$ for $(x, y) \neq (0, 0)$
    - Completeness:

        $
        \begin{array}{cl}
        (1)&\text{holds because } x_q = x_p \\
        (2)&\text{holds because } x_q = x_p \wedge y_p \neq 0, \text{ therefore } \lambda = 3x_p^2 / 2y_p \text{ is a solution}\\
        (3)&\text{holds because } x_r = \lambda^2 - x_p - x_q \wedge y_r = \lambda \cdot (x_p - x_r) - y_p \text{ in this case} \\
        (4)&\text{holds because } x_p \neq 0, \text{ therefore } \beta = x_p^{-1} \text{ is a solution} \\
        (5)&\text{holds because } x_p \neq 0, \text{ therefore } \gamma = x_q^{-1} \text{ is a solution} \\
        (6)&\text{holds because } x_q = x_p \text{ and } y_q \neq -y_p, \text{ therefore } \alpha = 0 \text{ and } \delta = (y_q + y_p)^{-1} \text{ is a solution.} \\
        \end{array}
        $

    - Soundness: $\lambda$ is computed correctly, and $(x_r, y_r) = (\lambda^2 - x_p - x_q, \lambda \cdot (x_p - x_r) - y_p)$ is the only solution.

* $(x, y) + (x, -y)$ for $(x, y) \neq (0, 0)$
    - Completeness:

        $
        \begin{array}{cl}
        (1)&\text{holds because } x_q = x_p \\
        (2)&\text{holds because } x_q = x_p \wedge y_p \neq 0, \text{ therefore } \lambda = 3x_p^2 / 2y_p \text{ is a solution} \\
           &\text{(although } \lambda \text{ is not used in this case)} \\
        (3)&\text{holds because } x_q = x_p \text{ and } y_q = -y_p \\
        (4)&\text{holds because } x_p \neq 0, \text{ therefore } \beta = x_p^{-1} \text{ is a solution} \\
        (5)&\text{holds because } x_q \neq 0, \text{ therefore } \gamma = x_q^{-1} \text{ is a solution} \\
        (6)&\text{holds because } (x_r, y_r) = (0, 0) \\
        \end{array}
        $

    - Soundness: $(x_r, y_r) = (0, 0)$ is the only solution to $(6).$

* $(x_p, y_p) + (x_q, y_q)$ for $(x_p, y_p) \neq (0,0)$ and $(x_q, y_q) \neq (0, 0)$ and $x_p \neq x_q$
    - Completeness:

        $
        \begin{array}{cl}
        (1)&\text{holds because } x_q \neq x_p, \text{ therefore } \lambda = (y_q - y_p) / (x_q - x_p) \text{ is a solution} \\
        (2)&\text{holds because } x_q \neq x_p, \text{ therefore } \alpha = (x_q - x_p)^{-1} \text{ is a solution} \\
        (3)&\text{holds because } x_r = \lambda^2 - x_p - x_q \wedge y_r = \lambda \cdot (x_p - x_r) - y_p \text{ in this case} \\
        (4)&\text{holds because } x_p \neq 0, \text{ therefore } \beta = x_p^{-1} \text{ is a solution} \\
        (5)&\text{holds because } x_q \neq 0, \text{ therefore } \gamma = x_q^{-1} \text{ is a solution} \\
        (6)&\text{holds because } x_q \neq x_p, \text{ therefore } \alpha = (x_q - x_p)^{-1} \text{ and } \delta = 0 \text{ is a solution.}
        \end{array}
        $

    - Soundness: $\lambda$ is computed correctly, and $(x_r, y_r) = (\lambda^2 - x_p - x_q, \lambda \cdot (x_p - x_r) - y_p)$ is the only solution.


# 椭圆曲线算术

我们将使用 [Hüseyin Hışıl 的论文](https://core.ac.uk/download/pdf/10898289.pdf) 第 4.1 节中推导出的短 Weierstrass 曲线上的仿射坐标公式。

## 不完全加法

- 输入：$P = (x_p, y_p), Q = (x_q, y_q)$
- 输出：$R = P \;⸭\; Q = (x_r, y_r)$

Hışıl 论文中的公式为：

- $x_3 = \left(\frac{y_1 - y_2}{x_1 - x_2}\right)^2 - x_1 - x_2$
- $y_3 = \frac{y_1 - y_2}{x_1 - x_2} \cdot (x_1 - x_3) - y_1.$

将 $(x_1, y_1)$ 重命名为 $(x_q, y_q)$，$(x_2, y_2)$ 重命名为 $(x_p, y_p)$，$(x_3, y_3)$ 重命名为 $(x_r, y_r)$，得到：

- $x_r = \left(\frac{y_q - y_p}{x_q - x_p}\right)^2 - x_q - x_p$
- $y_r = \frac{y_q - y_p}{x_q - x_p} \cdot (x_q - x_r) - y_q$

等价于：

- $x_r + x_q + x_p = \left(\frac{y_p - y_q}{x_p - x_q}\right)^2$
- $y_r + y_q = \frac{y_p - y_q}{x_p - x_q} \cdot (x_q - x_r).$

假设 $x_p \neq x_q$，我们有：

$
\begin{array}{lrrll}
&& x_r + x_q + x_p &=& \left(\frac{y_p - y_q}{x_p - x_q}\right)^2 \\[1.2ex]
&\Longleftrightarrow &(x_r + x_q + x_p) \cdot (x_p - x_q)^2 &=& (y_p - y_q)^2 \\[1ex]
&\Longleftrightarrow &(x_r + x_q + x_p) \cdot (x_p - x_q)^2 - (y_p - y_q)^2 &=& 0 \\[1.5ex]
\text{以及} \\
&&y_r + y_q &=& \frac{y_p - y_q}{x_p - x_q} \cdot (x_q - x_r) \\[0.8ex]
&\Longleftrightarrow &(y_r + y_q) \cdot (x_p - x_q) &=& (y_p - y_q) \cdot (x_q - x_r) \\[1ex]
&\Longleftrightarrow &(y_r + y_q) \cdot (x_p - x_q) - (y_p - y_q) \cdot (x_q - x_r) &=& 0.
\end{array}
$

因此，我们得到以下约束：
- $(x_r + x_q + x_p) \cdot (x_p - x_q)^2 - (y_p - y_q)^2 = 0$
  - 注意，此约束对于 $P \;⸭\; (-P)$（当 $P \neq \mathcal{O}$ 时）不可满足，
    因此不能用于任意输入。
- $(y_r + y_q) \cdot (x_p - x_q) - (y_p - y_q) \cdot (x_q - x_r) = 0.$

### 约束 <a name="incomplete-addition-constraints">

$$
\begin{array}{|c|l|}
\hline
\text{度数} & \text{约束} \\\hline
4 & q_\text{add-incomplete} \cdot \left( (x_r + x_q + x_p) \cdot (x_p - x_q)^2 - (y_p - y_q)^2 \right) = 0 \\\hline
3 & q_\text{add-incomplete} \cdot \left( (y_r + y_q) \cdot (x_p - x_q) - (y_p - y_q) \cdot (x_q - x_r) \right) = 0 \\\hline
\end{array}
$$

## 完全加法

$\hspace{1em} \begin{array}{rcll}
\mathcal{O} &+& \mathcal{O} &= \mathcal{O} \\[0.4ex]
\mathcal{O} &+& (x_q, y_q)  &= (x_q, y_q) \\[0.4ex]
 (x_p, y_p) &+& \mathcal{O} &= (x_p, y_p) \\[0.6ex]
   (x, y)   &+& (x, y)      &= [2] (x, y) \\[0.6ex]
   (x, y)   &+& (x, -y)     &= \mathcal{O} \\[0.4ex]
 (x_p, y_p) &+& (x_q, y_q)  &= (x_p, y_p) \;⸭\; (x_q, y_q), \text{ 如果 } x_p \neq x_q.
\end{array}$

假设我们将 $\mathcal{O}$ 表示为 $(0, 0)$。（$0$ 不是有效点的 $x$ 坐标，因为我们需要 $y^2 = x^3 + 5$，而 $5$ 在 $\mathbb{F}_q$ 中不是平方数。此外，$0$ 也不是有效点的 $y$ 坐标，因为 $-5$ 在 $\mathbb{F}_q$ 中不是立方数。）

$$
\begin{aligned}
P + Q &= R\\
(x_p, y_p) + (x_q, y_q) &= (x_r, y_r) \\
                \lambda &= \frac{y_q - y_p}{x_q - x_p} \\
                    x_r &= \lambda^2 - x_p - x_q \\
                    y_r &= \lambda(x_p - x_r) - y_p
\end{aligned}
$$

对于加倍情况，Hışıl 的论文告诉我们，$\lambda$ 必须改为计算为 $\frac{3x^2}{2y}$。

定义 $\mathsf{inv0}(x) = \begin{cases} 0, &\text{如果 } x = 0 \\ 1/x, &\text{否则。} \end{cases}$

见证 $\alpha, \beta, \gamma, \delta, \lambda$，其中：

$\hspace{1em}
\begin{array}{rl}
\alpha  \,\,=&\!\! \mathsf{inv0}(x_q - x_p) \\[0.4ex]
\beta   \,\,=&\!\! \mathsf{inv0}(x_p) \\[0.4ex]
\gamma  \,\,=&\!\! \mathsf{inv0}(x_q) \\[0.4ex]
\delta  \,\,=&\!\! \begin{cases}
                     \mathsf{inv0}(y_q + y_p), &\text{如果 } x_q = x_p \\
                     0, &\text{否则}
                   \end{cases} \\[2.5ex]
\lambda \,\,=&\!\! \begin{cases}
                     \frac{y_q - y_p}{x_q - x_p}, &\text{如果 } x_q \neq x_p \\[1.2ex]
                     \frac{3{x_p}^2}{2y_p} &\text{如果 } x_q = x_p \wedge y_p \neq 0 \\[0.8ex]
                     0, &\text{否则。}
                   \end{cases}
\end{array}
$

### 约束 <a name="complete-addition-constraints">

$$
\begin{array}{|c|lcl|l|}
\hline
\text{度数} & \text{约束}\hspace{7em} &&& \text{含义} \\\hline
4 & q_\mathit{add} \cdot (x_q - x_p) \cdot ((x_q - x_p) \cdot \lambda - (y_q - y_p)) &=& 0 & x_q \neq x_p \implies \lambda = \frac{y_q - y_p}{x_q - x_p} \\\hline \\[-2.3ex]
5 & q_\mathit{add} \cdot (1 - (x_q - x_p) \cdot \alpha) \cdot \left(2y_p \cdot \lambda - 3{x_p}^2\right) &=& 0 & \begin{cases} x_q = x_p \wedge y_p \neq 0 \implies \lambda = \frac{3{x_p}^2}{2y_p} \\ x_q = x_p \wedge y_p = 0 \implies x_p = 0 \end{cases} \\\hline
6 & q_\mathit{add} \cdot x_p \cdot x_q \cdot (x_q - x_p) \cdot (\lambda^2 - x_p - x_q - x_r) &=& 0 & x_p \neq 0 \wedge x_q \neq 0 \wedge x_q \neq x_p \implies x_r = \lambda^2 - x_p - x_q \\
6 & q_\mathit{add} \cdot x_p \cdot x_q \cdot (x_q - x_p) \cdot (\lambda \cdot (x_p - x_r) - y_p - y_r) &=& 0 & x_p \neq 0 \wedge x_q \neq 0 \wedge x_q \neq x_p \implies y_r = \lambda \cdot (x_p - x_r) - y_p \\
6 & q_\mathit{add} \cdot x_p \cdot x_q \cdot (y_q + y_p) \cdot (\lambda^2 - x_p - x_q - x_r) &=& 0 & x_p \neq 0 \wedge x_q \neq 0 \wedge y_q \neq -y_p \implies x_r = \lambda^2 - x_p - x_q \\
6 & q_\mathit{add} \cdot x_p \cdot x_q \cdot (y_q + y_p) \cdot (\lambda \cdot (x_p - x_r) - y_p - y_r) &=& 0 & x_p \neq 0 \wedge x_q \neq 0 \wedge y_q \neq -y_p \implies y_r = \lambda \cdot (x_p - x_r) - y_p \\\hline
4 & q_\mathit{add} \cdot (1 - x_p \cdot \beta) \cdot (x_r - x_q) &=& 0 & x_p = 0 \implies x_r = x_q \\
4 & q_\mathit{add} \cdot (1 - x_p \cdot \beta) \cdot (y_r - y_q) &=& 0 & x_p = 0 \implies y_r = y_q \\\hline
4 & q_\mathit{add} \cdot (1 - x_q \cdot \gamma) \cdot (x_r - x_p) &=& 0 & x_q = 0 \implies x_r = x_p \\
4 & q_\mathit{add} \cdot (1 - x_q \cdot \gamma) \cdot (y_r - y_p) &=& 0 & x_q = 0 \implies y_r = y_p \\\hline
4 & q_\mathit{add} \cdot (1 - (x_q - x_p) \cdot \alpha - (y_q + y_p) \cdot \delta) \cdot x_r &=& 0 & x_q = x_p \wedge y_q = -y_p \implies x_r = 0 \\
4 & q_\mathit{add} \cdot (1 - (x_q - x_p) \cdot \alpha - (y_q + y_p) \cdot \delta) \cdot y_r &=& 0 & x_q = x_p \wedge y_q = -y_p \implies y_r = 0 \\\hline
\end{array}
$$

最大度数：6

### 约束分析
$$
\begin{array}{rl}
1. & (x_q - x_p) \cdot ((x_q - x_p) \cdot \lambda - (y_q - y_p)) = 0 \\
   & \\
   & \begin{aligned}
         \text{至少满足以下之一：} &x_q - x_p = 0 \\
                      \text{或 } &(x_q - x_p) \cdot \lambda - (y_q - y_p) = 0 \\
       \end{aligned} \\
   & \text{才能使约束成立。} \\
   & \\
   & \text{如果 } x_q - x_p \neq 0, \text{ 则 } (x_q - x_p) \cdot \lambda - (y_q - y_p) = 0, \text{ 并且} \\
   & \text{通过重新排列两边，我们得到 } \lambda = (y_q - y_p) / (x_q - x_p). \\
   & \\
   & \text{因此：} \\
   & \hspace{2em} x_q \neq x_p \implies \lambda = (y_q - y_p) / (x_q - x_p).\\
   & \\
2. & (1 - (x_q - x_p) \cdot \alpha) \cdot (2y_p \cdot \lambda - 3x_p^2) = 0 \\
   & \\
   & \begin{aligned}
         \text{至少满足以下之一：} &(1 - (x_q - x_p) \cdot \alpha) = 0 \\
                      \text{或 } &(2y_p \cdot \lambda - 3x_p^2) = 0
       \end{aligned} \\
   & \text{才能使约束成立。} \\
   & \\
   & \text{如果 } x_q = x_p, \text{ 则 } 1 - (x_q - x_p) \cdot \alpha = 0 \text{ 无解，} \\
   & \text{因此必须满足 } 2y_p \cdot \lambda - 3x_p^2 = 0. \\
   & \\
   & \text{如果 } x_q = x_p \text{ 且 } y_p = 0 \text{ 则 } x_p = 0, \text{ 且约束成立。}\\
   & \\
   & \text{如果 } x_q = x_p \text{ 且 } y_p \neq 0 \text{ 则通过重新排列两边} \\
   & \text{我们得到 } \lambda = 3x_p^2 / 2y_p. \\
   & \\
   & \text{因此：} \\
   & \hspace{2em} (x_q = x_p) \wedge y_p \neq 0 \implies \lambda = 3x_p^2 / 2y_p. \\
   & \\
3.\text{ a)} & x_p \cdot x_q \cdot (x_q - x_p) \cdot (\lambda^2 - x_p - x_q - x_r) = 0 \\
  \text{ b)} & x_p \cdot x_q \cdot (x_q - x_p) \cdot (\lambda \cdot (x_p - x_r) - y_p - y_r) = 0 \\
  \text{ c)} & x_p \cdot x_q \cdot (y_q + y_p) \cdot (\lambda^2 - x_p - x_q - x_r) = 0 \\
  \text{ d)} & x_p \cdot x_q \cdot (y_q + y_p) \cdot (\lambda \cdot (x_p - x_r) - y_p - y_r) = 0 \\
   & \\
   & \begin{aligned}
       \text{至少满足以下之一：} &x_p = 0 \\
                    \text{或 } &x_p = 0 \\
                    \text{或 } &(x_q - x_p) = 0 \\
                    \text{或 } &(\lambda^2 - x_p - x_q - x_r) = 0 \\
     \end{aligned} \\
   & \text{才能使约束 (a) 成立。} \\
   & \\
   & \text{如果 } x_p \neq 0 \wedge x_q \neq 0 \wedge x_q \neq x_p, \\[1.5ex]
   & \text{• 约束 (a) 要求 } x_r = \lambda^2 - x_p - x_q. \\
   & \text{• 约束 (b) 要求 } y_r = \lambda \cdot (x_p - x_r) - y_p. \\
   & \\
   & \text{如果 } x_p \neq 0 \wedge x_q \neq 0 \wedge y_q \neq -y_p, \\[1.5ex]
   & \text{• 约束 (c) 要求 } x_r = \lambda^2 - x_p - x_q. \\
   & \text{• 约束 (d) 要求 } y_r = \lambda \cdot (x_p - x_r) - y_p. \\
   & \\
   & \text{因此：} \\
   & \begin{aligned}
                 &(x_p \neq 0) \wedge (x_q \neq 0) \wedge ((x_q \neq x_p) \vee (y_q \neq -y_p)) \\
        \implies &(x_r = \lambda^2 - x_p - x_q) \wedge (y_r = \lambda \cdot (x_p - x_r) - y_p).
     \end{aligned} \\
   & \\
4.\text{ a)} & (1 - x_p \cdot \beta) \cdot (x_r - x_q) = 0 \\
  \text{ b)} & (1 - x_p \cdot \beta) \cdot (y_r - y_q) = 0 \\
  & \\
  & \begin{aligned}
      \text{至少满足以下之一：} 1 - x_p \cdot \beta &= 0 \\
                             \text{或 } x_r - x_q &= 0
    \end{aligned} \\
  & \text{才能使约束 (a) 成立。} \\
  & \\
  & \text{如果 } x_p = 0 \text{ 则 } 1 - x_p \cdot \beta = 0 \text{ 无解，} \\
  & \text{因此必须满足 } x_r - x_q = 0. \\
  & \\
  & \text{类似地，约束 (b) 要求如果 } x_p = 0 \\
  & \text{则 } y_r - y_q = 0. \\
  & \\
  & \text{因此：} \\
  & \hspace{2em} x_p = 0 \implies (x_r, y_r) = (x_q, y_q). \\
  & \\
5.\text{ a)} & (1 - x_q \cdot \gamma) \cdot (x_r - x_p) = 0 \\
  \text{ b)} & (1 - x_q \cdot \gamma) \cdot (y_r - y_p) = 0 \\
  & \\
  & \begin{aligned}
      \text{至少满足以下之一：} 1 - x_q \cdot \gamma &= 0 \\
                             \text{或 } x_r - x_p &= 0
    \end{aligned} \\
  & \text{才能使约束 (a) 成立。} \\
  & \\
  & \text{如果 } x_q = 0 \text{ 则 } 1 - x_q \cdot \gamma = 0 \text{ 无解，} \\
  & \text{因此必须满足 } x_r - x_p = 0. \\
  & \\
  & \text{类似地，约束 (b) 要求如果 } x_q = 0 \\
  & \text{则 } y_r - y_p = 0. \\
  & \\
  & \text{因此：} \\
  & \hspace{2em} x_q = 0 \implies (x_r, y_r) = (x_p, y_p). \\
  & \\
6.\text{ a)} & (1 - (x_q - x_p) \cdot \alpha - (y_q + y_p) \cdot \delta) \cdot x_r = 0 \\
  \text{ b)} & (1 - (x_q - x_p) \cdot \alpha - (y_q + y_p) \cdot \delta) \cdot y_r = 0 \\
  & \\
  & \begin{aligned}
      \text{至少满足以下之一：} &1 - (x_q - x_p) \cdot \alpha - (y_q + y_p) \cdot \delta = 0 \\
                   \text{或 } &x_r = 0
    \end{aligned} \\
  & \text{才能使约束 (a) 成立，} \\
  & \text{类似地替换 } x_r \text{ 为 } y_r. \\
  & \\
  & \text{如果 } x_r \neq 0 \text{ 或 } y_r \neq 0, \text{ 则必须满足 } 1 - (x_q - x_p) \cdot \alpha - (y_q + y_p) \cdot \delta = 0. \\
  & \\
  & \text{然而，如果 } x_q = x_p \wedge y_q = -y_p, \text{ 则无解。} \\
  & \\
  & \text{因此： } \\
  & \hspace{2em} x_q = x_p \wedge y_q = -y_p \implies (x_r, y_r) = (0, 0).
\end{array}
$$

#### 命题：

$
\begin{array}{cl}
(1)& x_q \neq x_p \implies \lambda = (y_q - y_p) / (x_q - x_p) \\[0.8ex]
(2)& (x_q = x_p) \wedge y_p \neq 0 \implies \lambda = 3x_p^2 / 2y_p \\[0.8ex]
(3)& (x_p \neq 0) \wedge (x_q \neq 0) \wedge ((x_q \neq x_p) \vee (y_q \neq -y_p)) \\[0.4ex]
    &\implies (x_r = \lambda^2 - x_p - x_q) \wedge (y_r = \lambda \cdot (x_p - x_r) - y_p) \\[0.8ex]
(4)& x_p = 0 \implies (x_r, y_r) = (x_q, y_q) \\[0.8ex]
(5)& x_q = 0 \implies (x_r, y_r) = (x_p, y_p) \\[0.8ex]
(6)& x_q = x_p \wedge y_q = -y_p \implies (x_r, y_r) = (0, 0)
\end{array}
$

#### 情况：

$(x_p, y_p) + (x_q, y_q) = (x_r, y_r)$

注意，我们依赖于 $0$ 不是 Pallas 曲线上除 $\mathcal{O}$ 外的点的有效 $x$ 坐标或 $y$ 坐标这一事实。

* $(0, 0) + (0, 0)$
    - 完备性：

        $
        \begin{array}{cl}
        (1)&\text{成立，因为 } x_q = x_p \\
        (2)&\text{成立，因为 } y_p = 0 \\
        (3)&\text{成立，因为 } x_p = 0 \\
        (4)&\text{成立，因为 } (x_r, y_r) = (x_q, y_q) = (0, 0) \\
        (5)&\text{成立，因为 } (x_r, y_r) = (x_p, y_p) = (0, 0) \\
        (6)&\text{成立，因为 } (x_r, y_r) = (0, 0). \\
        \end{array}
        $

    - 合理性：$(x_r, y_r) = (0, 0)$ 是 $(6)$ 的唯一解。

* $(x, y) + (0, 0)$ 对于 $(x, y) \neq (0, 0)$
    - 完备性：

        $
        \begin{array}{cl}
        (1)&\text{成立，因为 } x_q \neq x_p, \text{ 因此 } \lambda = (y_q - y_p) / (x_q - x_p) \text{ 是一个解} \\
        (2)&\text{成立，因为 } x_q \neq x_p, \text{ 因此 } \alpha = (x_q - x_p)^{-1} \text{ 是一个解} \\
        (3)&\text{成立，因为 } x_q = 0 \\
        (4)&\text{成立，因为 } x_p \neq 0, \text{ 因此 } \beta = x_p^{-1} \text{ 是一个解} \\
        (5)&\text{成立，因为 } (x_r, y_r) = (x_p, y_p) \\
        (6)&\text{成立，因为 } x_q \neq x_p, \text{ 因此 } \alpha = (x_q - x_p)^{-1} \text{ 且 } \delta = 0 \text{ 是一个解。}
        \end{array}
        $

    - 合理性：$(x_r, y_r) = (x_p, y_p)$ 是 $(5)$ 的唯一解。

* $(0, 0) + (x, y)$ 对于 $(x, y) \neq (0, 0)$
    - 完备性：

        $
        \begin{array}{cl}
        (1)&\text{成立，因为 } x_q \neq x_p, \text{ 因此 } \lambda = (y_q - y_p) / (x_q - x_p) \text{ 是一个解} \\
        (2)&\text{成立，因为 } x_q \neq x_p, \text{ 因此 } \alpha = (x_q - x_p)^{-1} \text{ 是一个解} \\
        (3)&\text{成立，因为 } x_p = 0 \\
        (4)&\text{成立，因为 } x_p = 0 \text{ 仅当 } (x_r, y_r) = (x_q, y_q) \\
        (5)&\text{成立，因为 } x_q \neq 0, \text{ 因此 } \gamma = x_q^{-1} \text{ 是一个解}\\
        (6)&\text{成立，因为 } x_q \neq x_p, \text{ 因此 } \alpha = (x_q - x_p)^{-1} \text{ 且 } \delta = 0 \text{ 是一个解。}
        \end{array}
        $

    - 合理性：$(x_r, y_r) = (x_q, y_q)$ 是 $(4)$ 的唯一解。

* $(x, y) + (x, y)$ 对于 $(x, y) \neq (0, 0)$
    - 完备性：

        $
        \begin{array}{cl}
        (1)&\text{成立，因为 } x_q = x_p \\
        (2)&\text{成立，因为 } x_q = x_p \wedge y_p \neq 0, \text{ 因此 } \lambda = 3x_p^2 / 2y_p \text{ 是一个解}\\
        (3)&\text{成立，因为 } x_r = \lambda^2 - x_p - x_q \wedge y_r = \lambda \cdot (x_p - x_r) - y_p \text{ 在这种情况下} \\
        (4)&\text{成立，因为 } x_p \neq 0, \text{ 因此 } \beta = x_p^{-1} \text{ 是一个解} \\
        (5)&\text{成立，因为 } x_p \neq 0, \text{ 因此 } \gamma = x_q^{-1} \text{ 是一个解} \\
        (6)&\text{成立，因为 } x_q = x_p \text{ 且 } y_q \neq -y_p, \text{ 因此 } \alpha = 0 \text{ 且 } \delta = (y_q + y_p)^{-1} \text{ 是一个解。} \\
        \end{array}
        $

    - 合理性：$\lambda$ 计算正确，且 $(x_r, y_r) = (\lambda^2 - x_p - x_q, \lambda \cdot (x_p - x_r) - y_p)$ 是唯一解。

* $(x, y) + (x, -y)$ 对于 $(x, y) \neq (0, 0)$
    - 完备性：

        $
        \begin{array}{cl}
        (1)&\text{成立，因为 } x_q = x_p \\
        (2)&\text{成立，因为 } x_q = x_p \wedge y_p \neq 0, \text{ 因此 } \lambda = 3x_p^2 / 2y_p \text{ 是一个解} \\
           &\text{（尽管 } \lambda \text{ 在这种情况下未被使用）} \\
        (3)&\text{成立，因为 } x_q = x_p \text{ 且 } y_q = -y_p \\
        (4)&\text{成立，因为 } x_p \neq 0, \text{ 因此 } \beta = x_p^{-1} \text{ 是一个解} \\
        (5)&\text{成立，因为 } x_q \neq 0, \text{ 因此 } \gamma = x_q^{-1} \text{ 是一个解} \\
        (6)&\text{成立，因为 } (x_r, y_r) = (0, 0) \\
        \end{array}
        $

    - 合理性：$(x_r, y_r) = (0, 0)$ 是 $(6)$ 的唯一解。

* $(x_p, y_p) + (x_q, y_q)$ 对于 $(x_p, y_p) \neq (0,0)$ 且 $(x_q, y_q) \neq (0, 0)$ 且 $x_p \neq x_q$
    - 完备性：

        $
        \begin{array}{cl}
        (1)&\text{成立，因为 } x_q \neq x_p, \text{ 因此 } \lambda = (y_q - y_p) / (x_q - x_p) \text{ 是一个解} \\
        (2)&\text{成立，因为 } x_q \neq x_p, \text{ 因此 } \alpha = (x_q - x_p)^{-1} \text{ 是一个解} \\
        (3)&\text{成立，因为 } x_r = \lambda^2 - x_p - x_q \wedge y_r = \lambda \cdot (x_p - x_r) - y_p \text{ 在这种情况下} \\
        (4)&\text{成立，因为 } x_p \neq 0, \text{ 因此 } \beta = x_p^{-1} \text{ 是一个解} \\
        (5)&\text{成立，因为 } x_q \neq 0, \text{ 因此 } \gamma = x_q^{-1} \text{ 是一个解} \\
        (6)&\text{成立，因为 } x_q \neq x_p, \text{ 因此 } \alpha = (x_q - x_p)^{-1} \text{ 且 } \delta = 0 \text{ 是一个解。}
        \end{array}
        $

    - 合理性：$\lambda$ 计算正确，且 $(x_r, y_r) = (\lambda^2 - x_p - x_q, \lambda \cdot (x_p - x_r) - y_p)$ 是唯一解。