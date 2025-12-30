# Comparison to other work

## BCMS20 Appendix A.2

Appendix A.2 of [BCMS20] describes a polynomial commitment scheme that is similar to the
one described in [BGH19] (BCMS20 being a generalization of the original Halo paper). Halo
2 builds on both of these works, and thus itself uses a polynomial commitment scheme that
is very similar to the one in BCMS20.

[BGH19]: https://eprint.iacr.org/2019/1021
[BCMS20]: https://eprint.iacr.org/2020/499

The following table provides a mapping between the variable names in BCMS20, and the
equivalent objects in Halo 2 (which builds on the nomenclature from the Halo paper):

|     BCMS20     |       Halo 2        |
| :------------: | :-----------------: |
|      $S$       |         $H$         |
|      $H$       |         $U$         |
|      $C$       |    `msm` or $P$     |
|    $\alpha$    |       $\iota$       |
|    $\xi_0$     |         $z$         |
|    $\xi_i$     |    `challenge_i`    |
|      $H'$      |       $[z] U$       |
|   $\bar{p}$    |      `s_poly`       |
| $\bar{\omega}$ |   `s_poly_blind`    |
|   $\bar{C}$    | `s_poly_commitment` |
|     $h(X)$     |       $g(X)$        |
|      $U$       |         $G$         |
|   $\omega'$    |   `blind` / $\xi$   |
|  $\mathbf{c}$  |    $\mathbf{a}$     |
|      $c$       | $a = \mathbf{a}_0$  |
|      $v'$      |        $ab$         |

Halo 2's polynomial commitment scheme differs from Appendix A.2 of BCMS20 in two ways:

1. Step 8 of the $\text{Open}$ algorithm computes a "non-hiding" commitment $C'$ prior to
   the inner product argument, which opens to the same value as $C$ but is a commitment to
   a randomly-drawn polynomial. The remainder of the protocol involves no blinding. By
   contrast, in Halo 2 we blind every single commitment that we make (even for instance
   and fixed polynomials, though using a blinding factor of 1 for the fixed polynomials);
   this makes the protocol simpler to reason about. As a consequence of this, the verifier
   needs to handle the cumulative blinding factor at the end of the protocol, and so there
   is no need to derive an equivalent to $C'$ at the start of the protocol.

   - $C'$ is also an input to the random oracle for $\xi_0$; in Halo 2 we utilize a
     transcript that has already committed to the equivalent components of $C'$ prior to
     sampling $z$.

2. The $\text{PC}_\text{DL}.\text{SuccinctCheck}$ subroutine (Figure 2 of BCMS20) computes
   the initial group element $C_0$ by adding $[v] H' = [v \xi_0] H$, which requires two
   scalar multiplications. Instead, we subtract $[v] G_0$ from the original commitment $P$,
   so that we're effectively opening the polynomial at the point to the value zero. The
   computation $[v] G_0$ is more efficient in the context of recursion because $G_0$ is a
   fixed base (so we can use lookup tables).

# 与其他工作的比较

## BCMS20 附录 A.2

[BCMS20] 的附录 A.2 描述了一种多项式承诺方案，该方案与 [BGH19] 中描述的方案类似（BCMS20 是对原始 Halo 论文的推广）。Halo 2 在这两项工作的基础上进行了构建，因此它本身使用的多项式承诺方案与 BCMS20 中的方案非常相似。

[BGH19]: https://eprint.iacr.org/2019/1021  
[BCMS20]: https://eprint.iacr.org/2020/499

下表提供了 BCMS20 中的变量名称与 Halo 2 中等效对象的映射关系（Halo 2 的命名法基于 Halo 论文）：

|     BCMS20     |       Halo 2        |
| :------------: | :-----------------: |
|      $S$       |         $H$         |
|      $H$       |         $U$         |
|      $C$       |    `msm` 或 $P$     |
|    $\alpha$    |       $\iota$       |
|    $\xi_0$     |         $z$         |
|    $\xi_i$     |    `challenge_i`    |
|      $H'$      |       $[z] U$       |
|   $\bar{p}$    |      `s_poly`       |
| $\bar{\omega}$ |   `s_poly_blind`    |
|   $\bar{C}$    | `s_poly_commitment` |
|     $h(X)$     |       $g(X)$        |
|      $U$       |         $G$         |
|   $\omega'$    |   `blind` / $\xi$   |
|  $\mathbf{c}$  |    $\mathbf{a}$     |
|      $c$       | $a = \mathbf{a}_0$  |
|      $v'$      |        $ab$         |

Halo 2 的多项式承诺方案与 BCMS20 附录 A.2 中的方案有以下两点不同：

1. $\text{Open}$ 算法的第 8 步在开始内积论证之前计算了一个“非隐藏”承诺 $C'$，该承诺与 $C$ 打开相同的值，但承诺的是随机绘制的多项式。协议的其余部分不涉及盲化。相比之下，在 Halo 2 中，我们对每一个承诺都进行了盲化（即使是实例和固定多项式，尽管对固定多项式使用了盲化因子 1）；这使得协议更易于推理。因此，验证者需要在协议结束时处理累积的盲化因子，因此在协议开始时无需推导与 $C'$ 等效的值。

   - $C'$ 也是 $\xi_0$ 随机预言机的输入；在 Halo 2 中，我们使用了一个在采样 $z$ 之前已经提交了与 $C'$ 等效组件的转录本。

2. $\text{PC}_\text{DL}.\text{SuccinctCheck}$ 子程序（BCMS20 的图 2）通过添加 $[v] H' = [v \xi_0] H$ 来计算初始群元素 $C_0$，这需要两次标量乘法。相反，我们从原始承诺 $P$ 中减去 $[v] G_0$，这样我们实际上是在该点将多项式打开为值零。在递归上下文中，计算 $[v] G_0$ 更高效，因为 $G_0$ 是一个固定基点（因此我们可以使用查找表）。