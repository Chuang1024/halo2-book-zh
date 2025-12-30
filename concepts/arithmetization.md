# PLONKish Arithmetization

The arithmetization used by Halo 2 comes from [PLONK](https://eprint.iacr.org/2019/953), or
more precisely its extension UltraPLONK that supports custom gates and lookup arguments. We'll
call it [***PLONKish***](https://twitter.com/feministPLT/status/1413815927704014850).

***PLONKish circuits*** are defined in terms of a rectangular matrix of values. We refer to
***rows***, ***columns***, and ***cells*** of this matrix with the conventional meanings.

A PLONKish circuit depends on a ***configuration***:

* A finite field $\mathbb{F}$, where cell values (for a given statement and witness) will be
  elements of $\mathbb{F}$.
* The number of columns in the matrix, and a specification of each column as being
  ***fixed***, ***advice***, or ***instance***. Fixed columns are fixed by the circuit;
  advice columns correspond to witness values; and instance columns are normally used for
  public inputs (technically, they can be used for any elements shared between the prover
  and verifier).

* A subset of the columns that can participate in equality constraints.

* A ***maximum constraint degree***.

* A sequence of ***polynomial constraints***. These are multivariate polynomials over
  $\mathbb{F}$ that must evaluate to zero *for each row*. The variables in a polynomial
  constraint may refer to a cell in a given column of the current row, or a given column of
  another row relative to this one (with wrap-around, i.e. taken modulo $n$). The maximum
  degree of each polynomial is given by the maximum constraint degree.

* A sequence of ***lookup arguments*** defined over tuples of ***input expressions***
  (which are multivariate polynomials as above) and ***table columns***.

A PLONKish circuit also defines:

* The number of rows $n$ in the matrix. $n$ must correspond to the size of a multiplicative
  subgroup of $\mathbb{F}^\times$; typically a power of two.

* A sequence of ***equality constraints***, which specify that two given cells must have equal
  values.

* The values of the fixed columns at each row.

From a circuit description we can generate a ***proving key*** and a ***verification key***,
which are needed for the operations of proving and verification for that circuit.

> Note that we specify the ordering of columns, polynomial constraints, lookup arguments, and
> equality constraints, even though these do not affect the meaning of the circuit. This makes
> it easier to define the generation of proving and verification keys as a deterministic
> process.

Typically, a configuration will define polynomial constraints that are switched off and on by
***selectors*** defined in fixed columns. For example, a constraint $q_i \cdot p(...) = 0$ can
be switched off for a particular row $i$ by setting $q_i = 0$. In this case we sometimes refer
to a set of constraints controlled by a set of selector columns that are designed to be used
together, as a ***gate***. Typically there will be a ***standard gate*** that supports generic
operations like field multiplication and division, and possibly also ***custom gates*** that
support more specialized operations.


# PLONKish 算术化

Halo 2 使用的算术化方法来自 [PLONK](https://eprint.iacr.org/2019/953)，更准确地说，是其扩展版本 UltraPLONK，支持自定义门和查找参数。我们称之为 [***PLONKish***](https://twitter.com/feministPLT/status/1413815927704014850)。

***PLONKish 电路*** 由一个矩形矩阵的值定义。我们用常规含义来指代该矩阵的***行***、***列***和***单元格***。

PLONKish 电路依赖于一个***配置***：

* 一个有限域 $\mathbb{F}$，其中单元格的值（对于给定的陈述和见证）将是 $\mathbb{F}$ 的元素。
* 矩阵中的列数，以及每列的规范，指定为***固定列***、***建议列***或***实例列***。固定列由电路固定；建议列对应于见证值；实例列通常用于公共输入（技术上，它们可以用于证明者和验证者之间共享的任何元素）。
* 可以参与等式约束的列的子集。
* 一个***最大约束次数***。
* 一系列***多项式约束***。这些是 $\mathbb{F}$ 上的多元多项式，必须*在每一行*评估为零。多项式约束中的变量可以引用当前行中给定列的单元格，或相对于当前行的另一行中给定列的单元格（带环绕，即模 $n$）。每个多项式的最大次数由最大约束次数给出。
* 一系列定义在***输入表达式***​（如上所述的多元多项式）和***表列***元组上的***查找参数***。

PLONKish 电路还定义：

* 矩阵中的行数 $n$。$n$ 必须对应于 $\mathbb{F}^\times$ 的乘法子群的大小；通常是 2 的幂。
* 一系列***等式约束***，指定两个给定的单元格必须具有相等的值。
* 每行固定列的值。

从电路描述中，我们可以生成一个***证明密钥***和一个***验证密钥***，这些密钥是该电路的证明和验证操作所需的。

> 注意，我们指定了列、多项式约束、查找参数和等式约束的顺序，即使这些不影响电路的含义。这使得定义证明密钥和验证密钥的生成过程更加确定。

通常，配置将定义由固定列中的***选择器***开关控制的多项式约束。例如，约束 $q_i \cdot p(...) = 0$ 可以通过将 $q_i$ 设置为 0 来为特定行 $i$ 关闭。在这种情况下，我们有时将一组由一组设计为一起使用的选择器列控制的约束称为***门***。通常，会有一个***标准门***支持通用操作，如域乘法和除法，可能还有***自定义门***支持更专业的操作。