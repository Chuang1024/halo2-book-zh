# Halo 2 proofs

## Proofs as opaque byte streams

In proving system implementations like `bellman`, there is a concrete `Proof` struct that
encapsulates the proof data, is returned by a prover, and can be passed to a verifier.

`halo2` does not contain any proof-like structures, for several reasons:

- The Proof structures would contain vectors of (vectors of) curve points and scalars.
  This complicates serialization/deserialization of proofs because the lengths of these
  vectors depend on the configuration of the circuit. However, we didn't want to encode
  the lengths of vectors inside of proofs, because at runtime the circuit is fixed, and
  thus so are the proof sizes.
- It's easy to accidentally put stuff into a Proof structure that isn't also placed in the
  transcript, which is a hazard when developing and implementing a proving system.
- We needed to be able to create multiple PLONK proofs at the same time; these proofs
  share many different substructures when they are for the same circuit.

Instead, `halo2` treats proof objects as opaque byte streams. Creation and consumption of
these byte streams happens via the transcript:

- The `TranscriptWrite` trait represents something that we can write proof components to
  (at proving time).
- The `TranscriptRead` trait represents something that we can read proof components from
  (at verifying time).

Crucially, implementations of `TranscriptWrite` are responsible for simultaneously writing
to some `std::io::Write` buffer at the same time that they hash things into the transcript,
and similarly for `TranscriptRead`/`std::io::Read`.

As a bonus, treating proofs as opaque byte streams ensures that verification accounts for
the cost of deserialization, which isn't negligible due to point compression.

## Proof encoding

A Halo 2 proof, constructed over a curve $E(\mathbb{F}_p)$, is encoded as a stream of:

- Points $P \in E(\mathbb{F}_p)$ (for commitments to polynomials), and
- Scalars $s \in \mathbb{F}_q$ (for evaluations of polynomials, and blinding values).

For the Pallas and Vesta curves, both points and scalars have 32-byte encodings, meaning
that proofs are always a multiple of 32 bytes.

The `halo2` crate supports proving multiple instances of a circuit simultaneously, in
order to share common proof components and protocol logic.

In the encoding description below, we will use the following circuit-specific constants:

- $k$ - the size parameter of the circuit (which has $2^k$ rows).
- $A$ - the number of advice columns.
- $F$ - the number of fixed columns.
- $I$ - the number of instance columns.
- $L$ - the number of lookup arguments.
- $P$ - the number of permutation arguments.
- $\textsf{Col}_P$ - the number of columns involved in permutation argument $P$.
- $D$ - the maximum degree for the quotient polynomial.
- $Q_A$ - the number of advice column queries.
- $Q_F$ - the number of fixed column queries.
- $Q_I$ - the number of instance column queries.
- $M$ - the number of instances of the circuit that are being proven simultaneously.

As the proof encoding directly follows the transcript, we can break the encoding into
sections matching the Halo 2 protocol:

- PLONK commitments:
  - $A$ points (repeated $M$ times).
  - $2L$ points (repeated $M$ times).
  - $P$ points (repeated $M$ times).
  - $L$ points (repeated $M$ times).

- Vanishing argument:
  - $D - 1$ points.
  - $Q_I$ scalars (repeated $M$ times).
  - $Q_A$ scalars (repeated $M$ times).
  - $Q_F$ scalars.
  - $D - 1$ scalars.

- PLONK evaluations:
  - $(2 + \textsf{Col}_P) \times P$ scalars (repeated $M$ times).
  - $5L$ scalars (repeated $M$ times).

- Multiopening argument:
  - 1 point.
  - 1 scalar per set of points in the multiopening argument.

- Polynomial commitment scheme:
  - $1 + 2k$ points.
  - $2$ scalars.

# Halo 2 证明

## 证明作为不透明的字节流

在诸如 `bellman` 的证明系统实现中，存在一个具体的 `Proof` 结构体，它封装了证明数据，由证明者返回，并可以传递给验证者。

`halo2` 不包含任何类似证明的结构，原因如下：

- 证明结构将包含（向量的）曲线点和标量的向量。这使得证明的序列化/反序列化变得复杂，因为这些向量的长度取决于电路的配置。然而，我们不希望在证明中编码向量的长度，因为在运行时电路是固定的，因此证明的大小也是固定的。
- 很容易意外地将某些内容放入证明结构中，而这些内容并未被放入转录中，这在开发和实现证明系统时是一个隐患。
- 我们需要能够同时创建多个 PLONK 证明；这些证明在用于同一电路时共享许多不同的子结构。

相反，`halo2` 将证明对象视为不透明的字节流。这些字节流的创建和消费通过转录进行：

- `TranscriptWrite` 特性表示我们可以将证明组件写入的内容（在证明时）。
- `TranscriptRead` 特性表示我们可以从中读取证明组件的内容（在验证时）。

关键的是，`TranscriptWrite` 的实现负责在将内容哈希到转录中的同时，写入某个 `std::io::Write` 缓冲区，`TranscriptRead`/`std::io::Read` 也是如此。

作为奖励，将证明视为不透明的字节流确保了验证会考虑反序列化的成本，由于点压缩，这一成本不可忽视。

## 证明编码

在曲线 $E(\mathbb{F}_p)$ 上构建的 Halo 2 证明被编码为以下流：

- 点 $P \in E(\mathbb{F}_p)$（用于多项式的承诺），以及
- 标量 $s \in \mathbb{F}_q$（用于多项式的评估和盲值）。

对于 Pallas 和 Vesta 曲线，点和标量都有 32 字节的编码，这意味着证明始终是 32 字节的倍数。

`halo2` 库支持同时证明电路的多个实例，以共享公共的证明组件和协议逻辑。

在下面的编码描述中，我们将使用以下电路特定的常量：

- $k$ - 电路的大小参数（具有 $2^k$ 行）。
- $A$ - 建议列的数量。
- $F$ - 固定列的数量。
- $I$ - 实例列的数量。
- $L$ - 查找参数的数量。
- $P$ - 置换参数的数量。
- $\textsf{Col}_P$ - 置换参数 $P$ 中涉及的列数。
- $D$ - 商多项式的最大次数。
- $Q_A$ - 建议列查询的数量。
- $Q_F$ - 固定列查询的数量。
- $Q_I$ - 实例列查询的数量。
- $M$ - 同时证明的电路实例数量。

由于证明编码直接遵循转录，我们可以将编码分为与 Halo 2 协议匹配的部分：

- PLONK 承诺：
  - $A$ 个点（重复 $M$ 次）。
  - $2L$ 个点（重复 $M$ 次）。
  - $P$ 个点（重复 $M$ 次）。
  - $L$ 个点（重复 $M$ 次）。

- 消失参数：
  - $D - 1$ 个点。
  - $Q_I$ 个标量（重复 $M$ 次）。
  - $Q_A$ 个标量（重复 $M$ 次）。
  - $Q_F$ 个标量。
  - $D - 1$ 个标量。

- PLONK 评估：
  - $(2 + \textsf{Col}_P) \times P$ 个标量（重复 $M$ 次）。
  - $5L$ 个标量（重复 $M$ 次）。

- 多开放参数：
  - 1 个点。
  - 多开放参数中每组点 1 个标量。

- 多项式承诺方案：
  - $1 + 2k$ 个点。
  - $2$ 个标量。