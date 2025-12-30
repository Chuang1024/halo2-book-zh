# A simple example

Let's start with a simple circuit, to introduce you to the common APIs and how they are
used. The circuit will take a public input $c$, and will prove knowledge of two private
inputs $a$ and $b$ such that

$$a^2 \cdot b^2 = c.$$

## Define instructions

Firstly, we need to define the instructions that our circuit will rely on. Instructions
are the boundary between high-level [gadgets](../concepts/gadgets.md) and the low-level
circuit operations. Instructions may be as coarse or as granular as desired, but in
practice you want to strike a balance between an instruction being large enough to
effectively optimize its implementation, and small enough that it is meaningfully
reusable.

For our circuit, we will use three instructions:
- Load a private number into the circuit.
- Multiply two numbers.
- Expose a number as a public input to the circuit.

We also need a type for a variable representing a number. Instruction interfaces provide
associated types for their inputs and outputs, to allow the implementations to represent
these in a way that makes the most sense for their optimization goals.

```rust,ignore,no_run
{{#include ../../../halo2_proofs/examples/simple-example.rs:instructions}}
```

## Define a chip implementation

For our circuit, we will build a [chip](../concepts/chips.md) that provides the above
numeric instructions for a finite field.

```rust,ignore,no_run
{{#include ../../../halo2_proofs/examples/simple-example.rs:chip}}
```

Every chip needs to implement the `Chip` trait. This defines the properties of the chip
that a `Layouter` may rely on when synthesizing a circuit, as well as enabling any initial
state that the chip requires to be loaded into the circuit.

```rust,ignore,no_run
{{#include ../../../halo2_proofs/examples/simple-example.rs:chip-impl}}
```

## Configure the chip

The chip needs to be configured with the columns, permutations, and gates that will be
required to implement all of the desired instructions.

```rust,ignore,no_run
{{#include ../../../halo2_proofs/examples/simple-example.rs:chip-config}}
```

## Implement chip traits

```rust,ignore,no_run
{{#include ../../../halo2_proofs/examples/simple-example.rs:instructions-impl}}
```

## Build the circuit

Now that we have the instructions we need, and a chip that implements them, we can finally
build our circuit!

```rust,ignore,no_run
{{#include ../../../halo2_proofs/examples/simple-example.rs:circuit}}
```

## Testing the circuit

`halo2_proofs::dev::MockProver` can be used to test that the circuit is working correctly. The
private and public inputs to the circuit are constructed as we will do to create a proof,
but by passing them to `MockProver::run` we get an object that can test every constraint
in the circuit, and tell us exactly what is failing (if anything).

```rust,ignore,no_run
{{#include ../../../halo2_proofs/examples/simple-example.rs:test-circuit}}
```

## Full example

You can find the source code for this example
[here](https://github.com/zcash/halo2/tree/main/halo2_proofs/examples/simple-example.rs).


# 一个简单的示例

让我们从一个简单的电路开始，向您介绍常用的 API 及其使用方法。该电路将接收一个公共输入 $c$，并证明知道两个私有输入 $a$ 和 $b$，使得

$$a^2 \cdot b^2 = c.$$

## 定义指令

首先，我们需要定义电路将依赖的指令。指令是高级 [小工具](../concepts/gadgets.md) 和低级电路操作之间的边界。指令可以尽可能粗粒度或细粒度，但在实践中，您需要在指令足够大以有效优化其实现和足够小以有意义地重用之间找到平衡。

对于我们的电路，我们将使用三个指令：
- 将一个私有数加载到电路中。
- 将两个数相乘。
- 将一个数暴露为电路的公共输入。

我们还需要一个类型来表示一个数的变量。指令接口为其输入和输出提供关联类型，以允许实现以最符合其优化目标的方式表示这些内容。

```rust,ignore,no_run
{{#include ../../../halo2_proofs/examples/simple-example.rs:instructions}}
```

## 定义芯片实现

对于我们的电路，我们将构建一个 [芯片](../concepts/chips.md)，为有限域提供上述数值指令。

```rust,ignore,no_run
{{#include ../../../halo2_proofs/examples/simple-example.rs:chip}}
```

每个芯片都需要实现 `Chip` trait。这定义了芯片的属性，`Layouter` 在合成电路时可能依赖这些属性，并且还允许将芯片所需的任何初始状态加载到电路中。

```rust,ignore,no_run
{{#include ../../../halo2_proofs/examples/simple-example.rs:chip-impl}}
```

## 配置芯片

芯片需要配置所需的列、置换和门，以实现所有所需的指令。

```rust,ignore,no_run
{{#include ../../../halo2_proofs/examples/simple-example.rs:chip-config}}
```

## 实现芯片 trait

```rust,ignore,no_run
{{#include ../../../halo2_proofs/examples/simple-example.rs:instructions-impl}}
```

## 构建电路

现在我们有了所需的指令和实现它们的芯片，我们终于可以构建我们的电路了！

```rust,ignore,no_run
{{#include ../../../halo2_proofs/examples/simple-example.rs:circuit}}
```

## 测试电路

`halo2_proofs::dev::MockProver` 可用于测试电路是否正常工作。电路的私有和公共输入按照我们创建证明的方式进行构建，但通过将它们传递给 `MockProver::run`，我们得到一个对象，可以测试电路中的每个约束，并确切地告诉我们什么失败了（如果有的话）。

```rust,ignore,no_run
{{#include ../../../halo2_proofs/examples/simple-example.rs:test-circuit}}
```

## 完整示例

您可以在 [这里](https://github.com/zcash/halo2/tree/main/halo2_proofs/examples/simple-example.rs) 找到此示例的源代码。
