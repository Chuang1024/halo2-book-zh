# Gadgets

When implementing a circuit, we could use the features of the chips we've selected directly.
Typically, though, we will use them via ***gadgets***. This indirection is useful because,
for reasons of efficiency and limitations imposed by PLONKish circuits, the chip interfaces will
often be dependent on low-level implementation details. The gadget interface can provide a more
convenient and stable API that abstracts away from extraneous detail.

For example, consider a hash function such as SHA-256. The interface of a chip supporting
SHA-256 might be dependent on internals of the hash function design such as the separation
between message schedule and compression function. The corresponding gadget interface can
provide a more convenient and familiar `update`/`finalize` API, and can also handle parts
of the hash function that do not need chip support, such as padding. This is similar to how
[accelerated](https://software.intel.com/content/www/us/en/develop/articles/intel-sha-extensions.html)
[instructions](https://developer.arm.com/documentation/ddi0514/g/introduction/about-the-cortex-a57-processor-cryptography-engine)
for cryptographic primitives on CPUs are typically accessed via software libraries, rather
than directly.

Gadgets can also provide modular and reusable abstractions for circuit programming
at a higher level, similar to their use in libraries such as
[libsnark](https://github.com/christianlundkvist/libsnark-tutorial) and
[bellman](https://electriccoin.co/blog/bellman-zksnarks-in-rust/). As well as abstracting
*functions*, they can also abstract *types*, such as elliptic curve points or integers of
specific sizes.

# 小工具（Gadgets）

在实现电路时，我们可以直接使用所选芯片的功能。然而，通常情况下，我们会通过***小工具***来使用它们。这种间接性非常有用，因为出于效率考虑以及 PLONKish 电路的限制，芯片接口通常会依赖于低级的实现细节。小工具接口可以提供更方便、更稳定的 API，从而抽象掉无关的细节。

例如，考虑像 SHA-256 这样的哈希函数。支持 SHA-256 的芯片接口可能会依赖于哈希函数设计的内部细节，例如消息调度和压缩函数之间的分离。对应的小工具接口可以提供更便捷且熟悉的 `update`/`finalize` API，还可以处理不需要芯片支持的部分，例如填充。这类似于 CPU 上加密原语的[加速指令](https://software.intel.com/content/www/us/en/develop/articles/intel-sha-extensions.html)通常通过软件库访问，而不是直接使用。

小工具还可以为电路编程提供模块化和可重用的高级抽象，类似于它们在[libsnark](https://github.com/christianlundkvist/libsnark-tutorial)和[bellman](https://electriccoin.co/blog/bellman-zksnarks-in-rust/)等库中的使用。除了抽象*功能*外，它们还可以抽象*类型*，例如椭圆曲线点或特定大小的整数。