# Chips

The previous section gives a fairly low-level description of a circuit. When implementing circuits we will
typically use a higher-level API which aims for the desirable characteristics of auditability,
efficiency, modularity, and expressiveness.

Some of the terminology and concepts used in this API are taken from an analogy with
integrated circuit design and layout. [As for integrated circuits](https://opencores.org/),
the above desirable characteristics are easier to obtain by composing ***chips*** that provide
efficient pre-built implementations of particular functionality.

For example, we might have chips that implement particular cryptographic primitives such as a
hash function or cipher, or algorithms like scalar multiplication or pairings.

In PLONKish circuits, it is possible to build up arbitrary logic just from standard gates that do
field multiplication and addition. However, very significant efficiency gains can be obtained by
using custom gates.

Using our API, we define chips that "know" how to use particular sets of custom gates. This
creates an abstraction layer that isolates the implementation of a high-level circuit from the
complexity of using custom gates directly.

> Even if we sometimes need to "wear two hats", by implementing both a high-level circuit and
> the chips that it uses, the intention is that this separation will result in code that is
> easier to understand, audit, and maintain/reuse. This is partly because some potential
> implementation errors are ruled out by construction.

Gates in PLONKish circuits refer to cells by ***relative references***, i.e. to the cell in a given
column, and the row at a given offset relative to the one in which the gate's selector is set. We
call this an ***offset reference*** when the offset is nonzero (i.e. offset references are a subset
of relative references).

Relative references contrast with ***absolute references*** used in equality constraints,
which can point to any cell.

The motivation for offset references is to reduce the number of columns needed in the
configuration, which reduces proof size. If we did not have offset references then we would
need a column to hold each value referred to by a custom gate, and we would need to use
equality constraints to copy values from other cells of the circuit into that column. With
offset references, we not only need fewer columns; we also do not need equality constraints to
be supported for all of those columns, which improves efficiency.

In R1CS (another arithmetization which may be more familiar to some readers, but don't worry
if it isn't), a circuit consists of a "sea of gates" with no semantically significant ordering.
Because of offset references, the order of rows in a PLONKish circuit, on the other hand, *is*
significant. We're going to make some simplifying assumptions and define some abstractions to
tame the resulting complexity: the aim will be that, [at the gadget level](gadgets.md) where
we do most of our circuit construction, we will not have to deal with relative references or
with gate layout explicitly.

We will partition a circuit into ***regions***, where each region contains a disjoint subset
of cells, and relative references only ever point *within* a region. Part of the responsibility
of a chip implementation is to ensure that gates that make offset references are laid out in
the correct positions in a region.

Given the set of regions and their ***shapes***, we will use a separate ***floor planner***
to decide where (i.e. at what starting row) each region is placed. There is a default floor
planner that implements a very general algorithm, but you can write your own floor planner if
you need to.

Floor planning will in general leave gaps in the matrix, because the gates in a given row did
not use all available columns. These are filled in —as far as possible— by gates that do
not require offset references, which allows them to be placed on any row.

Chips can also define lookup tables. If more than one table is defined for the same lookup
argument, we can use a ***tag column*** to specify which table is used on each row. It is also
possible to perform a lookup in the union of several tables (limited by the polynomial degree
bound).

## Composing chips
In order to combine functionality from several chips, we compose them in a tree. The top-level
chip defines a set of fixed, advice, and instance columns, and then specifies how they
should be distributed between lower-level chips.

In the simplest case, each lower-level chips will use columns disjoint from the other chips.
However, it is allowed to share a column between chips. It is important to optimize the number
of advice columns in particular, because that affects proof size.

The result (possibly after optimization) is a PLONKish configuration. Our circuit implementation
will be parameterized on a chip, and can use any features of the supported lower-level chips via
the top-level chip.

Our hope is that less expert users will normally be able to find an existing chip that
supports the operations they need, or only have to make minor modifications to an existing
chip. Expert users will have full control to do the kind of
[circuit optimizations](https://zips.z.cash/protocol/canopy.pdf#circuitdesign)
[that ECC is famous  for](https://electriccoin.co/blog/cultivating-sapling-faster-zksnarks/) 🙂.

# 芯片

上一节对电路的描述较为底层。在实现电路时，我们通常会使用一个更高层次的 API，旨在实现可审计性、效率、模块化和表达性等理想特性。

该 API 中使用的一些术语和概念来自集成电路设计和布局的类比。[与集成电路类似](https://opencores.org/)，通过组合提供特定功能的高效预构建实现的***芯片***，更容易实现上述理想特性。

例如，我们可能有实现特定加密原语（如哈希函数或密码）或算法（如标量乘法或配对）的芯片。

在 PLONKish 电路中，仅使用标准门（进行域乘法和加法）就可以构建任意逻辑。然而，使用自定义门可以显著提高效率。

使用我们的 API，我们定义“知道”如何使用特定自定义门集的芯片。这创建了一个抽象层，将高级电路的实现与直接使用自定义门的复杂性隔离开来。

> 即使我们有时需要“戴两顶帽子”，即同时实现高级电路和其使用的芯片，其意图是这种分离将使代码更易于理解、审计和维护/重用。部分原因是一些潜在的实现错误在构建时被排除在外。

PLONKish 电路中的门通过***相对引用***来引用单元格，即引用给定列中的单元格，以及相对于设置门选择器的行的给定偏移量的行。当偏移量非零时，我们称之为***偏移引用***​（即偏移引用是相对引用的子集）。

相对引用与等式约束中使用的***绝对引用***形成对比，后者可以指向任何单元格。

偏移引用的动机是减少配置中所需的列数，从而减少证明大小。如果没有偏移引用，我们将需要一列来保存自定义门引用的每个值，并且需要使用等式约束将电路的其他单元格中的值复制到该列中。通过偏移引用，我们不仅需要更少的列；还不需要支持所有这些列的等式约束，这提高了效率。

在 R1CS（另一种算术化方法，可能对某些读者更熟悉，但如果不熟悉也不用担心）中，电路由“门海”组成，没有语义上的显著顺序。由于偏移引用，PLONKish 电路中的行顺序*是*有意义的。我们将做一些简化假设并定义一些抽象来管理由此产生的复杂性：目标是，在[小工具层面](gadgets.md)（我们进行大部分电路构建的地方），我们不需要显式处理相对引用或门布局。

我们将电路划分为***区域***，其中每个区域包含一个不相交的单元格子集，并且相对引用仅指向区域*内部*。芯片实现的部分职责是确保进行偏移引用的门在区域中布局在正确的位置。

给定区域集及其***形状***，我们将使用单独的***布局规划器***来决定每个区域的放置位置（即从哪一行开始）。有一个默认的布局规划器实现了一个非常通用的算法，但如果你需要，可以编写自己的布局规划器。

布局规划通常会在矩阵中留下间隙，因为给定行中的门并未使用所有可用列。这些间隙尽可能地被不需要偏移引用的门填充，这允许它们放置在任何行上。

芯片还可以定义查找表。如果为同一个查找参数定义了多个表，我们可以使用***标签列***来指定每行使用哪个表。也可以在多个表的联合中执行查找（受多项式次数限制）。

## 组合芯片
为了组合多个芯片的功能，我们将它们组合成一个树形结构。顶层芯片定义了一组固定列、建议列和实例列，然后指定它们应如何分配给下层芯片。

在最简单的情况下，每个下层芯片将使用与其他芯片不相交的列。然而，允许在芯片之间共享列。优化建议列的数量尤为重要，因为这会影响证明大小。

结果（可能在优化之后）是一个 PLONKish 配置。我们的电路实现将参数化在芯片上，并可以通过顶层芯片使用任何支持的下层芯片的功能。

我们希望，经验较少的用户通常能够找到支持其所需操作的现有芯片，或只需对现有芯片进行少量修改。专家用户将完全控制进行[电路优化](https://zips.z.cash/protocol/canopy.pdf#circuitdesign)，[就像 ECC 闻名的那样](https://electriccoin.co/blog/cultivating-sapling-faster-zksnarks/) 🙂。