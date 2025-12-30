# Developer tools

The `halo2` crate includes several utilities to help you design and implement your
circuits.

## Mock prover

`halo2_proofs::dev::MockProver` is a tool for debugging circuits, as well as cheaply verifying
their correctness in unit tests. The private and public inputs to the circuit are
constructed as would normally be done to create a proof, but `MockProver::run` instead
creates an object that will test every constraint in the circuit directly. It returns
granular error messages that indicate which specific constraint (if any) is not satisfied.

## Circuit visualizations

The `dev-graph` feature flag exposes several helper methods for creating graphical
representations of circuits.

On Debian systems, you will need the following additional packages:
```plaintext
sudo apt install cmake libexpat1-dev libfreetype6-dev libcairo2-dev
```

### Circuit layout

`halo2_proofs::dev::CircuitLayout` renders the circuit layout as a grid:

```rust,ignore,no_run
{{#include ../../../halo2_proofs/examples/circuit-layout.rs:dev-graph}}
```

- Columns are laid out from left to right as instance, advice and fixed. The order of
  columns is otherwise without meaning.
  - Instance columns have a white background.
  - Advice columns have a red background.
  - Fixed columns have a blue background.
- Regions are shown as labelled green boxes (overlaying the background colour). A region
  may appear as multiple boxes if some of its columns happen to not be adjacent.
- Cells that have been assigned to by the circuit will be shaded in grey. If any cells are
  assigned to more than once (which is usually a mistake), they will be shaded darker than
  the surrounding cells.

### Circuit structure

`halo2_proofs::dev::circuit_dot_graph` builds a [DOT graph string] representing the given
circuit, which can then be rendered with a variety of [layout programs]. The graph is built
from calls to `Layouter::namespace` both within the circuit, and inside the gadgets and
chips that it uses.

[DOT graph string]: https://graphviz.org/doc/info/lang.html
[layout programs]: https://en.wikipedia.org/wiki/DOT_(graph_description_language)#Layout_programs

```rust,ignore,no_run
fn main() {
    // Prepare the circuit you want to render.
    // You don't need to include any witness variables.
    let a = Fp::rand();
    let instance = Fp::one() + Fp::one();
    let lookup_table = vec![instance, a, a, Fp::zero()];
    let circuit: MyCircuit<Fp> = MyCircuit {
        a: None,
        lookup_table,
    };

    // Generate the DOT graph string.
    let dot_string = halo2_proofs::dev::circuit_dot_graph(&circuit);

    // Now you can either handle it in Rust, or just
    // print it out to use with command-line tools.
    print!("{}", dot_string);
}
```

## Cost estimator

The `cost-model` binary takes high-level parameters for a circuit design, and estimates
the verification cost, as well as resulting proof size.

```plaintext
Usage: cargo run --example cost-model -- [OPTIONS] k

Positional arguments:
  k                       2^K bound on the number of rows.

Optional arguments:
  -h, --help              Print this message.
  -a, --advice R[,R..]    An advice column with the given rotations. May be repeated.
  -i, --instance R[,R..]  An instance column with the given rotations. May be repeated.
  -f, --fixed R[,R..]     A fixed column with the given rotations. May be repeated.
  -g, --gate-degree D     Maximum degree of the custom gates.
  -l, --lookup N,I,T      A lookup over N columns with max input degree I and max table degree T. May be repeated.
  -p, --permutation N     A permutation over N columns. May be repeated.
```

For example, to estimate the cost of a circuit with three advice columns and one fixed
column (with various rotations), and a maximum gate degree of 4:

```plaintext
> cargo run --example cost-model -- -a 0,1 -a 0 -a-0,-1,1 -f 0 -g 4 11
    Finished dev [unoptimized + debuginfo] target(s) in 0.03s
     Running `target/debug/examples/cost-model -a 0,1 -a 0 -a 0,-1,1 -f 0 -g 4 11`
Circuit {
    k: 11,
    max_deg: 4,
    advice_columns: 3,
    lookups: 0,
    permutations: [],
    column_queries: 7,
    point_sets: 3,
    estimator: Estimator,
}
Proof size: 1440 bytes
Verification: at least 81.689ms
```



# 开发者工具

`halo2` crate 包含了多个实用工具，以帮助您设计和实现电路。

## 模拟证明器

`halo2_proofs::dev::MockProver` 是一个用于调试电路的工具，也可以在单元测试中廉价地验证其正确性。电路的私有和公共输入按照通常创建证明的方式进行构建，但 `MockProver::run` 会创建一个对象，直接测试电路中的每个约束。它返回详细的错误信息，指示哪个特定的约束（如果有）未被满足。

## 电路可视化

`dev-graph` 功能标志暴露了几个辅助方法，用于创建电路的图形表示。

在 Debian 系统上，您需要以下额外的包：
```plaintext
sudo apt install cmake libexpat1-dev libfreetype6-dev libcairo2-dev
```

### 电路布局

`halo2_proofs::dev::CircuitLayout` 将电路布局渲染为一个网格：

```rust,ignore,no_run
{{#include ../../../halo2_proofs/examples/circuit-layout.rs:dev-graph}}
```

- 列从左到右依次为实例列、建议列和固定列。列的顺序没有其他含义。
  - 实例列背景为白色。
  - 建议列背景为红色。
  - 固定列背景为蓝色。
- 区域显示为带标签的绿色框（覆盖背景颜色）。如果某些列不连续，区域可能显示为多个框。
- 由电路分配的单元格将显示为灰色。如果任何单元格被多次分配（通常是错误），它们将比周围单元格显示得更暗。

### 电路结构

`halo2_proofs::dev::circuit_dot_graph` 构建一个表示给定电路的 [DOT 图字符串]，然后可以使用各种 [布局程序] 进行渲染。该图从电路内部以及其使用的小工具和芯片中的 `Layouter::namespace` 调用构建。

[DOT 图字符串]: https://graphviz.org/doc/info/lang.html
[布局程序]: https://en.wikipedia.org/wiki/DOT_(graph_description_language)#Layout_programs

```rust,ignore,no_run
fn main() {
    // 准备您要渲染的电路。
    // 您不需要包含任何见证变量。
    let a = Fp::rand();
    let instance = Fp::one() + Fp::one();
    let lookup_table = vec![instance, a, a, Fp::zero()];
    let circuit: MyCircuit<Fp> = MyCircuit {
        a: None,
        lookup_table,
    };

    // 生成 DOT 图字符串。
    let dot_string = halo2_proofs::dev::circuit_dot_graph(&circuit);

    // 您现在可以在 Rust 中处理它，或者
    // 直接打印出来以使用命令行工具。
    print!("{}", dot_string);
}
```

## 成本估算器

`cost-model` 二进制文件接收电路设计的高级参数，并估算验证成本以及生成的证明大小。

```plaintext
用法: cargo run --example cost-model -- [选项] k

位置参数:
  k                       2^K 行数的上限。

可选参数:
  -h, --help              打印此消息。
  -a, --advice R[,R..]    具有给定旋转的建议列。可重复。
  -i, --instance R[,R..]  具有给定旋转的实例列。可重复。
  -f, --fixed R[,R..]     具有给定旋转的固定列。可重复。
  -g, --gate-degree D     自定义门的最大次数。
  -l, --lookup N,I,T      在 N 列上的查找，最大输入次数 I 和最大表次数 T。可重复。
  -p, --permutation N     在 N 列上的置换。可重复。
```

例如，估算具有三个建议列和一个固定列（具有各种旋转）且最大门次数为 4 的电路的成本：

```plaintext
> cargo run --example cost-model -- -a 0,1 -a 0 -a-0,-1,1 -f 0 -g 4 11
    完成 dev [未优化 + debuginfo] 目标 0.03s
    运行 `target/debug/examples/cost-model -a 0,1 -a 0 -a 0,-1,1 -f 0 -g 4 11`
Circuit {
    k: 11,
    max_deg: 4,
    advice_columns: 3,
    lookups: 0,
    permutations: [],
    column_queries: 7,
    point_sets: 3,
    estimator: Estimator,
}
证明大小: 1440 字节
验证: 至少 81.689ms

```