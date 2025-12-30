# Proving system

The Halo 2 proving system can be broken down into five stages:

1. Commit to polynomials encoding the main components of the circuit:
   - Cell assignments.
   - Permuted values and products for each lookup argument.
   - Equality constraint permutations.
2. Construct the vanishing argument to constrain all circuit relations to zero:
   - Standard and custom gates.
   - Lookup argument rules.
   - Equality constraint permutation rules.
3. Evaluate the above polynomials at all necessary points:
   - All relative rotations used by custom gates across all columns.
   - Vanishing argument pieces.
4. Construct the multipoint opening argument to check that all evaluations are consistent
   with their respective commitments.
5. Run the inner product argument to create a polynomial commitment opening proof for the
   multipoint opening argument polynomial.

These stages are presented in turn across this section of the book.

## Example

To aid our explanations, we will at times refer to the following example constraint
system:

- Four advice columns $a, b, c, d$.
- One fixed column $f$.
- Three custom gates:
  - $a \cdot b \cdot c_{-1} - d = 0$
  - $f_{-1} \cdot c = 0$
  - $f \cdot d \cdot a = 0$

## tl;dr

The table below provides a (probably too) succinct description of the Halo 2 protocol.
This description will likely be replaced by the Halo 2 paper and security proof, but for
now serves as a summary of the following sub-sections.

| Prover                                                                      |         | Verifier                           |
| --------------------------------------------------------------------------- | ------- | ---------------------------------- |
|                                                                             | $\larr$ | $t(X) = (X^n - 1)$                 |
|                                                                             | $\larr$ | $F = [F_0, F_1, \dots, F_{m - 1}]$ |
| $\mathbf{A} = [A_0, A_1, \dots, A_{m - 1}]$                                 | $\rarr$ |                                    |
|                                                                             | $\larr$ | $\theta$                           |
| $\mathbf{L} = [(A'_0, S'_0), \dots, (A'_{m - 1}, S'_{m - 1})]$              | $\rarr$ |                                    |
|                                                                             | $\larr$ | $\beta, \gamma$                    |
| $\mathbf{Z_P} = [Z_{P,0}, Z_{P,1}, \ldots]$                                 | $\rarr$ |                                    |
| $\mathbf{Z_L} = [Z_{L,0}, Z_{L,1}, \ldots]$                                 | $\rarr$ |                                    |
|                                                                             | $\larr$ | $y$                                |
| $h(X) = \frac{\text{gate}_0(X) + \dots + y^i \cdot \text{gate}_i(X)}{t(X)}$ |         |                                    |
| $h(X) = h_0(X) + \dots + X^{n(d-1)} h_{d-1}(X)$                             |         |                                    |
| $\mathbf{H} = [H_0, H_1, \dots, H_{d-1}]$                                   | $\rarr$ |                                    |
|                                                                             | $\larr$ | $x$                                |
| $evals = [A_0(x), \dots, H_{d - 1}(x)]$                                     | $\rarr$ |                                    |
|                                                                             |         | Checks $h(x)$                      |
|                                                                             | $\larr$ | $x_1, x_2$                         |
| Constructs $h'(X)$ multipoint opening poly                                  |         |                                    |
| $U = \text{Commit}(h'(X))$                                                  | $\rarr$ |                                    |
|                                                                             | $\larr$ | $x_3$                              |
| $\mathbf{q}_\text{evals} = [Q_0(x_3), Q_1(x_3), \dots]$                     | $\rarr$ |                                    |
| $u_\text{eval} = U(x_3)$                                                    | $\rarr$ |                                    |
|                                                                             | $\larr$ | $x_4$                              |

Then the prover and verifier:

- Construct $\text{finalPoly}(X)$ as a linear combination of $\mathbf{Q}$ and $U$ using
  powers of $x_4$;
- Construct $\text{finalPolyEval}$ as the equivalent linear combination of
  $\mathbf{q}_\text{evals}$ and $u_\text{eval}$; and
- Perform $\text{InnerProduct}(\text{finalPoly}(X), x_3, \text{finalPolyEval}).$

> TODO: Write up protocol components that provide zero-knowledge.

# 证明系统

Halo 2 的证明系统可以分为五个阶段：

1. ​**承诺编码电路主要组件的多项式**：
   - 单元格赋值。
   - 每个查找参数的置换值和乘积。
   - 等式约束置换。
2. ​**构建归零论证以约束所有电路关系为零**：
   - 标准和自定义门。
   - 查找参数规则。
   - 等式约束置换规则。
3. ​**在所有必要的点上评估上述多项式**：
   - 所有自定义门在所有列中使用的相对旋转。
   - 归零论证的各个部分。
4. ​**构建多点打开论证以检查所有评估是否与其各自的承诺一致**。
5. ​**运行内积论证以为多点打开论证多项式创建多项式承诺打开证明**。

这些阶段将在本书的这一部分中依次介绍。

## 示例

为了帮助解释，我们有时会参考以下示例约束系统：

- 四个建议列 $a, b, c, d$。
- 一个固定列 $f$。
- 三个自定义门：
  - $a \cdot b \cdot c_{-1} - d = 0$
  - $f_{-1} \cdot c = 0$
  - $f \cdot d \cdot a = 0$

## 简要说明

下表提供了 Halo 2 协议的（可能过于）简洁描述。该描述可能会被 Halo 2 论文和安全证明所取代，但目前作为以下小节的总结。

| 证明者                                                                      |         | 验证者                           |
| --------------------------------------------------------------------------- | ------- | ---------------------------------- |
|                                                                             | $\larr$ | $t(X) = (X^n - 1)$                 |
|                                                                             | $\larr$ | $F = [F_0, F_1, \dots, F_{m - 1}]$ |
| $\mathbf{A} = [A_0, A_1, \dots, A_{m - 1}]$                                 | $\rarr$ |                                    |
|                                                                             | $\larr$ | $\theta$                           |
| $\mathbf{L} = [(A'_0, S'_0), \dots, (A'_{m - 1}, S'_{m - 1})]$              | $\rarr$ |                                    |
|                                                                             | $\larr$ | $\beta, \gamma$                    |
| $\mathbf{Z_P} = [Z_{P,0}, Z_{P,1}, \ldots]$                                 | $\rarr$ |                                    |
| $\mathbf{Z_L} = [Z_{L,0}, Z_{L,1}, \ldots]$                                 | $\rarr$ |                                    |
|                                                                             | $\larr$ | $y$                                |
| $h(X) = \frac{\text{gate}_0(X) + \dots + y^i \cdot \text{gate}_i(X)}{t(X)}$ |         |                                    |
| $h(X) = h_0(X) + \dots + X^{n(d-1)} h_{d-1}(X)$                             |         |                                    |
| $\mathbf{H} = [H_0, H_1, \dots, H_{d-1}]$                                   | $\rarr$ |                                    |
|                                                                             | $\larr$ | $x$                                |
| $evals = [A_0(x), \dots, H_{d - 1}(x)]$                                     | $\rarr$ |                                    |
|                                                                             |         | 检查 $h(x)$                      |
|                                                                             | $\larr$ | $x_1, x_2$                         |
| 构建 $h'(X)$ 多点打开多项式                                                  |         |                                    |
| $U = \text{Commit}(h'(X))$                                                  | $\rarr$ |                                    |
|                                                                             | $\larr$ | $x_3$                              |
| $\mathbf{q}_\text{evals} = [Q_0(x_3), Q_1(x_3), \dots]$                     | $\rarr$ |                                    |
| $u_\text{eval} = U(x_3)$                                                    | $\rarr$ |                                    |
|                                                                             | $\larr$ | $x_4$                              |

然后证明者和验证者：

- 使用 $x_4$ 的幂构造 $\text{finalPoly}(X)$ 作为 $\mathbf{Q}$ 和 $U$ 的线性组合；
- 构造 $\text{finalPolyEval}$ 作为 $\mathbf{q}_\text{evals}$ 和 $u_\text{eval}$ 的等价线性组合；
- 执行 $\text{InnerProduct}(\text{finalPoly}(X), x_3, \text{finalPolyEval}).$

> 待办：编写提供零知识的协议组件。