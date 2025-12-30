# Decomposition
Given a field element $\alpha$, these gadgets decompose it into $W$ $K$-bit windows $$\alpha = k_0 + 2^{K} \cdot k_1 + 2^{2K} \cdot k_2 + \cdots + 2^{(W-1)K} \cdot k_{W-1}$$ where each $k_i$ a $K$-bit value.

This is done using a running sum $z_i, i \in [0..W).$ We initialize the running sum $z_0 = \alpha,$ and compute subsequent terms $z_{i+1} = \frac{z_i - k_i}{2^{K}}.$ This gives us:

$$
\begin{aligned}
z_0 &= \alpha \\
    &= k_0 + 2^{K} \cdot k_1 + 2^{2K} \cdot k_2 +  2^{3K} \cdot k_3 + \cdots, \\
z_1 &= (z_0 - k_0) / 2^K \\
    &= k_1 + 2^{K} \cdot k_2 +  2^{2K} \cdot k_3 + \cdots, \\
z_2 &= (z_1 - k_1) / 2^K \\
    &= k_2 +  2^{K} \cdot k_3 + \cdots, \\
    &\vdots \\
\downarrow &\text{ (in strict mode)} \\
z_W &= (z_{W-1} - k_{W-1}) / 2^K \\
    &= 0 \text{ (because } z_{W-1} = k_{W-1} \text{)}
\end{aligned}
$$

### Strict mode
Strict mode constrains the running sum output $z_{W}$ to be zero, thus range-constraining the field element to be within $W \cdot K$ bits.

In strict mode, we are also assured that $z_{W-1} = k_{W-1}$ gives us the last window in the decomposition.
## Lookup decomposition
This gadget makes use of a $K$-bit lookup table to decompose a field element $\alpha$ into $K$-bit words. Each $K$-bit word $k_i = z_i - 2^K \cdot z_{i+1}$ is range-constrained by a lookup in the $K$-bit table.

The region layout for the lookup decomposition uses a single advice column $z$, and two selectors $q_{lookup}$ and $q_{running}.$
$$
\begin{array}{|c|c|c|}
\hline
    z    & q_\mathit{lookup} & q_\mathit{running} \\\hline
\hline
  z_0    &     1      &       1     \\\hline
  z_1    &     1      &       1     \\\hline
\vdots   &   \vdots   &     \vdots  \\\hline
z_{n-1}  &     1      &       1     \\\hline
z_n      &     0      &       0     \\\hline
\end{array}
$$
### Short range check
Using two $K$-bit lookups, we can range-constrain a field element $\alpha$ to be $n$ bits, where $n \leq K.$ To do this:

1. Constrain $0 \leq \alpha < 2^K$ to be within $K$ bits using a $K$-bit lookup.
2. Constrain $0 \leq \alpha \cdot 2^{K - n} < 2^K$ to be within $K$ bits using a $K$-bit lookup.

The short variant of the lookup decomposition introduces a $q_{bitshift}$ selector. The same advice column $z$ has here been renamed to $\textsf{word}$ for clarity:
$$
\begin{array}{|c|c|c|c|}
\hline
\textsf{word} & q_\mathit{lookup} & q_\mathit{running} & q_\mathit{bitshift} \\\hline
\hline
\alpha        &     1      &      0      &       0      \\\hline
\alpha'       &     1      &      0      &       1      \\\hline
2^{K-n}       &     0      &      0      &       0      \\\hline
\end{array}
$$

where $\alpha' = \alpha \cdot 2^{K - n}.$ Note that $2^{K-n}$ is assigned to a fixed column at keygen, and copied in at proving time. This is used in the gate enabled by the $q_\mathit{bitshift}$ selector to check that $\alpha$ was shifted correctly:
$$
\begin{array}{|c|l|}
\hline
\text{Degree} & \text{Constraint} \\\hline
       2      & q_\mathit{bitshift} \cdot ((\alpha \cdot 2^{K - n}) - \alpha') \\\hline
\end{array}
$$

### Combined lookup expression
Since the lookup decomposition and its short variant both make use of the same lookup table, we combine their lookup input expressions into a single one:

$$q_\mathit{lookup} \cdot \left(q_\mathit{running} \cdot (z_i - 2^K \cdot z_{i+1}) + (1 - q_\mathit{running}) \cdot \textsf{word} \right)$$

where $z_i$ and $\textsf{word}$ are the same cell (but distinguished here for clarity of usage).

## Short range decomposition
For a short range (for instance, $[0, \texttt{range})$ where $\texttt{range} \leq 8$), we can range-constrain each word using a degree-$\texttt{range}$ polynomial constraint instead of a lookup: $$\RangeCheck{word}{range} = \texttt{word} \cdot (1 - \texttt{word}) \cdots (\texttt{range} - 1 - \texttt{word}).$$

# 分解

给定一个域元素 $\alpha$，这些 gadget 将其分解为 $W$ 个 $K$ 位的窗口：  
$$\alpha = k_0 + 2^{K} \cdot k_1 + 2^{2K} \cdot k_2 + \cdots + 2^{(W-1)K} \cdot k_{W-1}$$  
其中每个 $k_i$ 是一个 $K$ 位的值。

这是通过使用一个运行和 $z_i, i \in [0..W)$ 来完成的。我们初始化运行和 $z_0 = \alpha$，并计算后续项 $z_{i+1} = \frac{z_i - k_i}{2^{K}}$。这给出了：

$$
\begin{aligned}
z_0 &= \alpha \\
    &= k_0 + 2^{K} \cdot k_1 + 2^{2K} \cdot k_2 +  2^{3K} \cdot k_3 + \cdots, \\
z_1 &= (z_0 - k_0) / 2^K \\
    &= k_1 + 2^{K} \cdot k_2 +  2^{2K} \cdot k_3 + \cdots, \\
z_2 &= (z_1 - k_1) / 2^K \\
    &= k_2 +  2^{K} \cdot k_3 + \cdots, \\
    &\vdots \\
\downarrow &\text{（在严格模式下）} \\
z_W &= (z_{W-1} - k_{W-1}) / 2^K \\
    &= 0 \text{（因为 } z_{W-1} = k_{W-1} \text{)}
\end{aligned}
$$

### 严格模式
严格模式将运行和输出 $z_{W}$ 约束为零，从而将域元素范围约束在 $W \cdot K$ 位内。

在严格模式下，我们还确保 $z_{W-1} = k_{W-1}$ 给出了分解中的最后一个窗口。

## 查找表分解
此 gadget 使用一个 $K$ 位的查找表将域元素 $\alpha$ 分解为 $K$ 位的字。每个 $K$ 位的字 $k_i = z_i - 2^K \cdot z_{i+1}$ 通过在 $K$ 位查找表中的查找进行范围约束。

查找表分解的区域布局使用一个建议列 $z$ 和两个选择器 $q_{lookup}$ 和 $q_{running}$。

$$
\begin{array}{|c|c|c|}
\hline
    z    & q_\mathit{lookup} & q_\mathit{running} \\\hline
\hline
  z_0    &     1      &       1     \\\hline
  z_1    &     1      &       1     \\\hline
\vdots   &   \vdots   &     \vdots  \\\hline
z_{n-1}  &     1      &       1     \\\hline
z_n      &     0      &       0     \\\hline
\end{array}
$$

### 短范围检查
使用两个 $K$ 位的查找，我们可以将域元素 $\alpha$ 范围约束为 $n$ 位，其中 $n \leq K$。为此：

1. 使用 $K$ 位查找将 $0 \leq \alpha < 2^K$ 约束在 $K$ 位内。
2. 使用 $K$ 位查找将 $0 \leq \alpha \cdot 2^{K - n} < 2^K$ 约束在 $K$ 位内。

查找表分解的短范围变体引入了一个 $q_{bitshift}$ 选择器。为清晰起见，建议列 $z$ 在此重命名为 $\textsf{word}$：

$$
\begin{array}{|c|c|c|c|}
\hline
\textsf{word} & q_\mathit{lookup} & q_\mathit{running} & q_\mathit{bitshift} \\\hline
\hline
\alpha        &     1      &      0      &       0      \\\hline
\alpha'       &     1      &      0      &       1      \\\hline
2^{K-n}       &     0      &      0      &       0      \\\hline
\end{array}
$$

其中 $\alpha' = \alpha \cdot 2^{K - n}$。注意 $2^{K-n}$ 在密钥生成时被分配到固定列，并在证明时复制。这用于由 $q_\mathit{bitshift}$ 选择器启用的门中，以检查 $\alpha$ 是否正确移位：

$$
\begin{array}{|c|l|}
\hline
\text{度数} & \text{约束} \\\hline
       2      & q_\mathit{bitshift} \cdot ((\alpha \cdot 2^{K - n}) - \alpha') \\\hline
\end{array}
$$

### 组合查找表达式
由于查找表分解及其短范围变体都使用相同的查找表，我们将它们的查找输入表达式组合成一个：

$$q_\mathit{lookup} \cdot \left(q_\mathit{running} \cdot (z_i - 2^K \cdot z_{i+1}) + (1 - q_\mathit{running}) \cdot \textsf{word} \right)$$

其中 $z_i$ 和 $\textsf{word}$ 是相同的单元格（但在此区分以明确使用方式）。

## 短范围分解
对于短范围（例如，$[0, \texttt{range})$，其中 $\texttt{range} \leq 8$），我们可以使用一个度数为 $\texttt{range}$ 的多项式约束来范围约束每个字，而不是使用查找表：  
$$\RangeCheck{word}{range} = \texttt{word} \cdot (1 - \texttt{word}) \cdots (\texttt{range} - 1 - \texttt{word}).$$