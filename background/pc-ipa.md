# Polynomial commitment using inner product argument
We want to commit to some polynomial $p(X) \in \mathbb{F}_p[X]$, and be able to provably
evaluate the committed polynomial at arbitrary points. The naive solution would be for the
prover to simply send the polynomial's coefficients to the verifier: however, this
requires $O(n)$ communication. Our polynomial commitment scheme gets the job done using
$O(\log n)$ communication.

### `Setup`
Given a parameter $d = 2^k,$ we generate the common reference string
$\sigma = (\mathbb{G}, \mathbf{G}, H, \mathbb{F}_p)$ defining certain constants for this
scheme:
* $\mathbb{G}$ is a group of prime order $p;$
* $\mathbf{G} \in \mathbb{G}^d$ is a vector of $d$ random group elements;
* $H \in \mathbb{G}$ is a random group element; and
* $\mathbb{F}_p$ is the finite field of order $p.$

### `Commit`
The Pedersen vector commitment $\text{Commit}$ is defined as

$$\text{Commit}(\sigma, p(X); r) = \langle\mathbf{a}, \mathbf{G}\rangle + [r]H,$$

for some polynomial $p(X) \in \mathbb{F}_p[X]$ and some blinding factor
$r \in \mathbb{F}_p.$ Here, each element of the vector $\mathbf{a}_i \in \mathbb{F}_p$ is
the coefficient for the $i$th degree term of $p(X),$ and $p(X)$ is of maximal degree
$d - 1.$

### `Open` (prover) and `OpenVerify` (verifier)
The modified inner product argument is an argument of knowledge for the relation

$$\boxed{\{((P, x, v); (\mathbf{a}, r)): P = \langle\mathbf{a}, \mathbf{G}\rangle + [r]H, v = \langle\mathbf{a}, \mathbf{b}\rangle\}},$$

where $\mathbf{b} = (1, x, x^2, \cdots, x^{d-1})$ is composed of increasing powers of the
evaluation point $x.$ This allows a prover to demonstrate to a verifier that the
polynomial contained “inside” the commitment $P$ evaluates to $v$ at $x,$ and moreover,
that the committed polynomial has maximum degree $d − 1.$

The inner product argument proceeds in $k = \log_2 d$ rounds. For our purposes, it is
sufficient to know about its final outputs, while merely providing intuition about the
intermediate rounds. (Refer to Section 3 in the [Halo] paper for a full explanation.)

[Halo]: https://eprint.iacr.org/2019/1021.pdf

Before beginning the argument, the verifier selects a random group element $U$ and sends it
to the prover. We initialize the argument at round $k,$ with the vectors
$\mathbf{a}^{(k)} := \mathbf{a},$ $\mathbf{G}^{(k)} := \mathbf{G}$ and
$\mathbf{b}^{(k)} := \mathbf{b}.$ In each round $j = k, k-1, \cdots, 1$:

* the prover computes two values $L_j$ and $R_j$ by taking some inner product of
  $\mathbf{a}^{(j)}$ with $\mathbf{G}^{(j)}$ and $\mathbf{b}^{(j)}$. Note that are in some
  sense "cross-terms": the lower half of $\mathbf{a}$ is used with the higher half of
  $\mathbf{G}$ and $\mathbf{b}$, and vice versa:

$$
\begin{aligned}
L_j &= \langle\mathbf{a_{lo}^{(j)}}, \mathbf{G_{hi}^{(j)}}\rangle + [l_j]H + [\langle\mathbf{a_{lo}^{(j)}}, \mathbf{b_{hi}^{(j)}}\rangle] U\\
R_j &= \langle\mathbf{a_{hi}^{(j)}}, \mathbf{G_{lo}^{(j)}}\rangle + [r_j]H + [\langle\mathbf{a_{hi}^{(j)}}, \mathbf{b_{lo}^{(j)}}\rangle] U\\
\end{aligned}
$$

* the verifier issues a random challenge $u_j$;
* the prover uses $u_j$ to compress the lower and higher halves of $\mathbf{a}^{(j)}$,
  thus producing a new vector of half the original length 
  $$\mathbf{a}^{(j-1)} = \mathbf{a_{hi}^{(j)}} + \mathbf{a_{lo}^{(j)}}\cdot u_j^{-1}.$$
  The vectors $\mathbf{G}^{(j)}$ and $\mathbf{b}^{(j)}$ are similarly compressed to give
  $\mathbf{G}^{(j-1)}$ and $\mathbf{b}^{(j-1)}$ (using $u_j$ instead of $u_j^{-1}$).
* $\mathbf{a}^{(j-1)}$, $\mathbf{G}^{(j-1)}$ and $\mathbf{b}^{(j-1)}$ are input to the
  next round $j - 1.$

Note that at the end of the last round $j = 1,$ we are left with $a := \mathbf{a}^{(0)}$,
$G := \mathbf{G}^{(0)}$, $b := \mathbf{b}^{(0)},$ each of length 1. The intuition is that
these final scalars, together with the challenges $\{u_j\}$ and "cross-terms"
$\{L_j, R_j\}$ from each round, encode the compression in each round. Since the prover did
not know the challenges $U, \{u_j\}$ in advance, they would have been unable to manipulate
the round compressions. Thus, checking a constraint on these final terms should enforce
that the compression had been performed correctly, and that the original $\mathbf{a}$
satisfied the relation before undergoing compression.

Note that $G, b$ are simply rearrangements of the publicly known $\mathbf{G}, \mathbf{b},$
with the round challenges $\{u_j\}$ mixed in: this means the verifier can compute $G, b$
independently and verify that the prover had provided those same values.


# 使用内积论证的多项式承诺

我们希望承诺某个多项式 $p(X) \in \mathbb{F}_p[X]$，并能够在任意点上可证明地评估该承诺的多项式。朴素的解决方案是证明者简单地将多项式的系数发送给验证者，但这需要 $O(n)$ 的通信量。我们的多项式承诺方案通过 $O(\log n)$ 的通信量完成任务。

### `Setup`
给定参数 $d = 2^k$，我们生成公共参考字符串 $\sigma = (\mathbb{G}, \mathbf{G}, H, \mathbb{F}_p)$，定义该方案的常量：
* $\mathbb{G}$ 是一个素数阶为 $p$ 的群；
* $\mathbf{G} \in \mathbb{G}^d$ 是一个由 $d$ 个随机群元素组成的向量；
* $H \in \mathbb{G}$ 是一个随机群元素；以及
* $\mathbb{F}_p$ 是阶为 $p$ 的有限域。

### `Commit`
Pedersen 向量承诺 $\text{Commit}$ 定义为

$$\text{Commit}(\sigma, p(X); r) = \langle\mathbf{a}, \mathbf{G}\rangle + [r]H,$$

其中 $p(X) \in \mathbb{F}_p[X]$ 是某个多项式，$r \in \mathbb{F}_p$ 是盲化因子。这里，向量 $\mathbf{a}_i \in \mathbb{F}_p$ 的每个元素是 $p(X)$ 的第 $i$ 次项系数，且 $p(X)$ 的最大次数为 $d - 1$。

### `Open`（证明者）和 `OpenVerify`（验证者）
修改后的内积论证是以下关系的知识论证：

$$\boxed{\{((P, x, v); (\mathbf{a}, r)): P = \langle\mathbf{a}, \mathbf{G}\rangle + [r]H, v = \langle\mathbf{a}, \mathbf{b}\rangle\}},$$

其中 $\mathbf{b} = (1, x, x^2, \cdots, x^{d-1})$ 由评估点 $x$ 的递增幂组成。这使得证明者能够向验证者证明，承诺 $P$ 中“包含”的多项式在 $x$ 处的评估值为 $v$，并且承诺的多项式的最大次数为 $d - 1$。

内积论证进行 $k = \log_2 d$ 轮。对于我们的目的，了解其最终输出就足够了，同时仅提供关于中间轮的直觉。（完整解释请参阅 [Halo] 论文的第 3 节。）

[Halo]: https://eprint.iacr.org/2019/1021.pdf

在开始论证之前，验证者选择一个随机群元素 $U$ 并将其发送给证明者。我们在第 $k$ 轮初始化论证，向量为 $\mathbf{a}^{(k)} := \mathbf{a}$，$\mathbf{G}^{(k)} := \mathbf{G}$ 和 $\mathbf{b}^{(k)} := \mathbf{b}$。在每轮 $j = k, k-1, \cdots, 1$ 中：

* 证明者通过将 $\mathbf{a}^{(j)}$ 与 $\mathbf{G}^{(j)}$ 和 $\mathbf{b}^{(j)}$ 进行某种内积计算两个值 $L_j$ 和 $R_j$。注意，这些值在某种意义上是“交叉项”：$\mathbf{a}$ 的下半部分与 $\mathbf{G}$ 和 $\mathbf{b}$ 的上半部分一起使用，反之亦然：

$$
\begin{aligned}
L_j &= \langle\mathbf{a_{lo}^{(j)}}, \mathbf{G_{hi}^{(j)}}\rangle + [l_j]H + [\langle\mathbf{a_{lo}^{(j)}}, \mathbf{b_{hi}^{(j)}}\rangle] U\\
R_j &= \langle\mathbf{a_{hi}^{(j)}}, \mathbf{G_{lo}^{(j)}}\rangle + [r_j]H + [\langle\mathbf{a_{hi}^{(j)}}, \mathbf{b_{lo}^{(j)}}\rangle] U\\
\end{aligned}
$$

* 验证者发出一个随机挑战 $u_j$；
* 证明者使用 $u_j$ 压缩 $\mathbf{a}^{(j)}$ 的下半部分和上半部分，从而生成一个长度为原长度一半的新向量
  $$\mathbf{a}^{(j-1)} = \mathbf{a_{hi}^{(j)}} + \mathbf{a_{lo}^{(j)}}\cdot u_j^{-1}.$$
  向量 $\mathbf{G}^{(j)}$ 和 $\mathbf{b}^{(j)}$ 类似地压缩为 $\mathbf{G}^{(j-1)}$ 和 $\mathbf{b}^{(j-1)}$（使用 $u_j$ 而不是 $u_j^{-1}$）。
* $\mathbf{a}^{(j-1)}$、$\mathbf{G}^{(j-1)}$ 和 $\mathbf{b}^{(j-1)}$ 被输入到下一轮 $j - 1$。

注意，在最后一轮 $j = 1$ 结束时，我们得到 $a := \mathbf{a}^{(0)}$、$G := \mathbf{G}^{(0)}$ 和 $b := \mathbf{b}^{(0)}$，每个长度为 1。直觉是，这些最终标量，连同每轮的挑战 $\{u_j\}$ 和“交叉项” $\{L_j, R_j\}$，编码了每轮的压缩。由于证明者事先不知道挑战 $U, \{u_j\}$，他们无法操纵轮的压缩。因此，检查这些最终项的约束应该能够确保压缩已正确执行，并且原始的 $\mathbf{a}$ 在压缩之前满足关系。

注意，$G, b$ 只是公开已知的 $\mathbf{G}, \mathbf{b}$ 的重排，并混合了每轮的挑战 $\{u_j\}$：这意味着验证者可以独立计算 $G, b$ 并验证证明者是否提供了相同的值。