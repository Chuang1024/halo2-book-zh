## Recursion
> Alternative terms: Induction; Accumulation scheme; Proof-carrying data

However, the computation of $G$ requires a length-$2^k$ multiexponentiation
$\langle \mathbf{G}, \mathbf{s}\rangle,$ where $\mathbf{s}$ is composed of the round
challenges $u_1, \cdots, u_k$ arranged in a binary counting structure. This is the
linear-time computation that we want to amortise across a batch of proof instances.
Instead of computing $G,$ notice that we can express $G$ as a commitment to a polynomial

$$G = \text{Commit}(\sigma, g(X, u_1, \cdots, u_k)),$$

where $g(X, u_1, \cdots, u_k) := \prod_{i=1}^k (u_i + u_i^{-1}X^{2^{i-1}})$ is a
polynomial with degree $2^k - 1.$ 

|  |  | 
| -------- | -------- | 
| <img src="https://i.imgur.com/vMXKFDV.png" width=1900> | Since $G$ is a commitment, it can be checked in an inner product argument. The verifier circuit witnesses $G$ and brings $G, u_1, \cdots, u_k$ out as public inputs to the proof $\pi.$ The next verifier instance checks $\pi$ using the inner product argument; this includes checking that $G = \text{Commit}(g(X, u_1, \cdots, u_k))$ evaluates at some random point to the expected value for the given challenges $u_1, \cdots, u_k.$ Recall from the [previous section](#Polynomial-commitment-using-inner-product-argument) that this check only requires $\log d$ work. <br><br> At the end of checking $\pi$ and $G,$ the circuit is left with a new $G',$ along with the $u_1', \cdots, u_k'$ challenges sampled for the check. To fully accept $\pi$ as valid, we should perform a linear-time computation of $G' = \langle\mathbf{G}, \mathbf{s}'\rangle$. Once again, we delay this computation by witnessing $G'$ and bringing $G', u_1', \cdots, u_k'$ out as public inputs to the proof $\pi'.$ <br><br> This goes on from one proof instance to the next, until we are satisfied with the size of our batch of proofs. We finally perform a single linear-time computation, thus deciding the validity of the whole batch.   |

We recall from the section [Cycles of curves](curves.md#cycles-of-curves) that we can
instantiate this protocol over a two-cycle, where a proof produced by one curve is
efficiently verified in the circuit of the other curve. However, some of these verifier
checks can actually be efficiently performed in the native circuit; these are "deferred"
to the next native circuit (see diagram below) instead of being immediately passed over to
the other curve. 

![](https://i.imgur.com/l4HrYgE.png)

## 递归
> 替代术语：归纳；累积方案；证明携带数据

然而，计算 $G$ 需要一个长度为 $2^k$ 的多重指数运算 $\langle \mathbf{G}, \mathbf{s}\rangle$，其中 $\mathbf{s}$ 由轮次挑战 $u_1, \cdots, u_k$ 按二进制计数结构排列组成。这是我们希望在批量证明实例中分摊的线性时间计算。与其计算 $G$，不如注意到我们可以将 $G$ 表示为一个多项式的承诺

$$G = \text{Commit}(\sigma, g(X, u_1, \cdots, u_k)),$$

其中 $g(X, u_1, \cdots, u_k) := \prod_{i=1}^k (u_i + u_i^{-1}X^{2^{i-1}})$ 是一个次数为 $2^k - 1$ 的多项式。

|  |  | 
| -------- | -------- | 
| <img src="https://i.imgur.com/vMXKFDV.png" width=1900> | 由于 $G$ 是一个承诺，它可以在内积论证中进行检查。验证者电路见证 $G$，并将 $G, u_1, \cdots, u_k$ 作为公共输入带到证明 $\pi$ 中。下一个验证者实例使用内积论证检查 $\pi$；这包括检查 $G = \text{Commit}(g(X, u_1, \cdots, u_k))$ 在某个随机点处是否评估为给定挑战 $u_1, \cdots, u_k$ 的预期值。回顾[上一节](#使用内积论证的多项式承诺)，此检查仅需要 $\log d$ 的工作量。<br><br> 在检查 $\pi$ 和 $G$ 之后，电路会得到一个新的 $G'$，以及为检查而采样的挑战 $u_1', \cdots, u_k'$。为了完全接受 $\pi$ 为有效，我们应该执行 $G' = \langle\mathbf{G}, \mathbf{s}'\rangle$ 的线性时间计算。我们再次延迟此计算，通过见证 $G'$ 并将 $G', u_1', \cdots, u_k'$ 作为公共输入带到证明 $\pi'$ 中。<br><br> 这种操作从一个证明实例延续到下一个实例，直到我们对批量证明的大小感到满意。最终，我们执行一次线性时间计算，从而决定整个批量的有效性。   |

我们回顾[曲线循环](curves.md#cycles-of-curves)部分，我们可以在双循环上实例化此协议，其中一个曲线生成的证明在另一个曲线的电路中高效验证。然而，其中一些验证者检查实际上可以在本地电路中高效执行；这些检查被“推迟”到下一个本地电路（见下图），而不是立即传递到另一个曲线。

![](https://i.imgur.com/l4HrYgE.png)