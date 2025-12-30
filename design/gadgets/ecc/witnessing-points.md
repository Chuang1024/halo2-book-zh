# Witnessing points

We represent elliptic curve points in the circuit in their affine representation $(x, y)$.
The identity is represented as the pseudo-coordinate $(0, 0)$, which we
[assume](../ecc.md#chip-assumptions) is not a valid point on the curve.

## Non-identity points

To constrain a coordinate pair $(x, y)$ as representing a valid point on the curve, we
directly check the curve equation. For Pallas and Vesta, this is:

$$y^2 = x^3 + 5$$

$$
\begin{array}{|c|l|}
\hline
\text{Degree} & \text{Constraint} \\\hline
4 & q_\text{point}^\text{non-id} \cdot (y^2 - x^3 - 5) = 0 \\\hline
\end{array}
$$

## Points including the identity

To allow $(x, y)$ to represent either a valid point on the curve, or the pseudo-coordinate
$(0, 0)$, we define a separate gate that enforces the curve equation check unless both $x$
and $y$ are zero.

$$
\begin{array}{|c|l|}
\hline
\text{Degree} & \text{Constraint} \\\hline
5 & (q_\text{point} \cdot x) \cdot (y^2 - x^3 - 5) = 0 \\\hline
5 & (q_\text{point} \cdot y) \cdot (y^2 - x^3 - 5) = 0 \\\hline
\end{array}
$$

# 见证点

我们在电路中以仿射表示 $(x, y)$ 来表示椭圆曲线点。  
单位元表示为伪坐标 $(0, 0)$，我们[假设](../ecc.md#chip-assumptions)这不是曲线上的有效点。

## 非单位元点

为了约束坐标对 $(x, y)$ 表示曲线上的有效点，我们直接检查曲线方程。对于 Pallas 和 Vesta，曲线方程为：

$$y^2 = x^3 + 5$$

$$
\begin{array}{|c|l|}
\hline
\text{度数} & \text{约束} \\\hline
4 & q_\text{point}^\text{non-id} \cdot (y^2 - x^3 - 5) = 0 \\\hline
\end{array}
$$

## 包括单位元的点

为了允许 $(x, y)$ 表示曲线上的有效点或伪坐标 $(0, 0)$，我们定义了一个单独的门，除非 $x$ 和 $y$ 都为零，否则强制执行曲线方程检查。

$$
\begin{array}{|c|l|}
\hline
\text{度数} & \text{约束} \\\hline
5 & (q_\text{point} \cdot x) \cdot (y^2 - x^3 - 5) = 0 \\\hline
5 & (q_\text{point} \cdot y) \cdot (y^2 - x^3 - 5) = 0 \\\hline
\end{array}
$$
