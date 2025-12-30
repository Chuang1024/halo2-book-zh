# Using halo2 in WASM

Since halo2 is written in Rust, you can compile it to WebAssembly (wasm), which will allow you to use the prover and verifier for your circuits in browser applications. This tutorial takes you through all you need to know to compile your circuits to wasm.

Throughout this tutorial, we will follow the repository for [Zordle](https://github.com/nalinbhardwaj/zordle) for reference, one of the first known webapps based on Halo 2 circuits. Zordle is ZK [Wordle](https://www.nytimes.com/games/wordle/index.html), where the circuit takes as advice values the player's input words and the player's share grid (the grey, yellow and green squares) and verifies that they match correctly. Therefore, the proof verifies that the player knows a "preimage" to the output share sheet, which can then be verified using just the ZK proof.

## Circuit code setup

The first step is to create functions in Rust that will interface with the browser application. In the case of a prover, this will typically input some version of the advice and instance data, use it to generate a complete witness, and then output a proof. In the case of a verifier, this will typically input a proof and some version of the instance, and then output a boolean indicating whether the proof verified correctly or not.

In the case of Zordle, this code is contained in [wasm.rs](https://github.com/nalinbhardwaj/zordle/blob/main/circuits/src/wasm.rs), and consists of two primary functions:

### Prover

```rust,ignore
#[wasm_bindgen]
pub async fn prove_play(final_word: String, words_js: JsValue, params_ser: JsValue) -> JsValue {
  // Steps:
  // - Deserialise function parameters
  // - Generate the instance and advice columns using the words
  // - Instantiate the circuit and generate the witness
  // - Generate the proving key from the params
  // - Create a proof
}
```

While the specific inputs and their serialisations will depend on your circuit and webapp set up, it's useful to note the format in the specific case of Zordle since your use case will likely be similar:

This function takes as input the `final_word` that the user aimed for, and the words they attempted to use (in the form of `words_js`). It also takes as input the parameters for the circuit, which are serialized in `params_ser`. We will expand on this in the [Params](#params) section below.

Note that the function parameters are passed in `wasm_bindgen`-compatible formats: `String` and `JsValue`. The `JsValue` type is a type from the [`Serde`](https://serde.rs) library. You can find much more details about this type and how to use it in the documentation [here](https://rustwasm.github.io/wasm-bindgen/reference/arbitrary-data-with-serde.html#serializing-and-deserializing-arbitrary-data-into-and-from-jsvalue-with-serde).

The output is a `Vec<u8>` converted to a `JSValue` using Serde. This is later passed in as input to the the verifier function.

### Verifier

```rust,ignore
#[wasm_bindgen]
pub fn verify_play(final_word: String, proof_js: JsValue, diffs_u64_js: JsValue, params_ser: JsValue) -> bool {
  // Steps:
  // - Deserialise function parameters
  // - Generate the instance columns using the diffs representation of the columns
  // - Generate the verifying key using the params
  // - Verify the proof
}
```

Similar to the prover, we take in input and output a boolean true/false indicating the correctness of the proof. The `diffs_u64_js` object is a 2D JS array consisting of values for each cell that indicate the color: grey, yellow or green. These are used to assemble the instance columns for the circuit.

### Params

Additionally, both the prover and verifier functions input `params_ser`, a serialised form of the public parameters of the polynomial commitment scheme. These are passed in as input (instead of being regenerated in prove/verify functions) as a performance optimisation since these are constant based only on the circuit's value of `K`. We can store these separately on a static web server and pass them in as input to the WASM. To generate the binary serialised form of these (separately outside the WASM functions), you can run something like:

```rust,ignore
fn write_params(K: u32) {
    let mut params_file = File::create("params.bin").unwrap();
    let params: Params<EqAffine> = Params::new(K);
    params.write(&mut params_file).unwrap();
}
```

Later, we can read the `params.bin` file from the web-server in Javascript in a byte-serialised format as a `Uint8Array` and pass it to the WASM as `params_ser`, which can be deserialised in Rust using the [`js_sys`](https://docs.rs/js-sys/latest/js_sys/) library.

Ideally, in future, instead of serialising the parameters we would be able to serialise and work directly with the proving key and the verifying key of the circuit, but that is currently not supported by the library, and tracked as issue [#449](https://github.com/zcash/halo2/issues/449) and [#443](https://github.com/zcash/halo2/issues/443).

## Rust and WASM environment setup

Typically, Rust code is compiled to WASM using the [`wasm-pack`](https://developer.mozilla.org/en-US/docs/WebAssembly/Rust_to_wasm) tool and is as simple as changing some build commands. In the case of halo2 prover/verifier functions however, we need to make some additional changes to the build process. In particular, there are two main changes:

- **Parallelism**: halo2 uses the `rayon` library for parallelism, which is not directly supported by WASM. However, the Chrome team has an adapter to enable rayon-like parallelism using Web Workers in browser: [`wasm-bindgen-rayon`](https://github.com/GoogleChromeLabs/wasm-bindgen-rayon). We'll use this to enable parallelism in our WASM prover/verifier.
- **WASM max memory**: The default memory limit for WASM with `wasm-bindgen` is set to 2GB, which is not enough to run the halo2 prover for large circuits (with `K` > 10 or so). We need to increase this limit to the maximum allowed by WASM (4GB!) to support larger circuits (up to `K = 15` or so).

Firstly, add all the dependencies particular to your WASM interfacing functions to your `Cargo.toml` file. You can restrict the dependencies to the WASM compilation by using the WASM target feature flag. In the case of Zordle, [this looks like](https://github.com/nalinbhardwaj/zordle/blob/main/circuits/Cargo.toml#L24):

```toml
[target.'cfg(target_family = "wasm")'.dependencies]
getrandom = { version = "0.2", features = ["js"]}
wasm-bindgen = { version = "0.2.81", features = ["serde-serialize"]}
console_error_panic_hook = "0.1.7"
rayon = "1.5"
wasm-bindgen-rayon = { version = "1.0"}
web-sys = { version = "0.3", features = ["Request", "Window", "Response"] }
wasm-bindgen-futures = "0.4"
js-sys = "0.3"
```

Next, let's integrate `wasm-bindgen-rayon` into our code. The [README for the library](https://github.com/GoogleChromeLabs/wasm-bindgen-rayon) has a great overview of how to do so. In particular, note the [changes to the Rust compilation pipeline](https://github.com/GoogleChromeLabs/wasm-bindgen-rayon#using-config-files). You need to switch to a nightly version of Rust and enable support for WASM atomics. Additionally, remember to export the [`init_thread_pool`](https://github.com/GoogleChromeLabs/wasm-bindgen-rayon#setting-up) in Rust code.

Next, we will bump up the default 2GB max memory limit for `wasm-pack`. To do so, add `"-C", "link-arg=--max-memory=4294967296"` Rust flag to the wasm target in the `.cargo/config` file. With the setup for `wasm-bindgen-rayon` and the memory bump, the `.cargo/config` file should now look like:

```toml
[target.wasm32-unknown-unknown]
rustflags = ["-C", "target-feature=+atomics,+bulk-memory,+mutable-globals", "-C", "link-arg=--max-memory=4294967296"]
...
```

Shoutout to [@mattgibb](https://github.com/mattgibb) who documented this esoteric change for increasing maximum memory in a random GitHub issue [here](https://github.com/rustwasm/wasm-bindgen/issues/2498#issuecomment-801498175).[^1]

[^1]: Off-topic but it was quite surprising for me to learn that WASM has a hard maximum limitation of 4GB memory. This is because WASM currently has a 32-bit architecture, which was quite surprising to me for such a new, forward-facing assembly language. There are, however, some open proposals to [move WASM to a larger address space](https://github.com/WebAssembly/memory64).

Now that we have the Rust set up, you should be able to build a WASM package simply using `wasm-pack build --target web --out-dir pkg` and use the output WASM package in your webapp.

## Webapp setup

Zordle ships with a minimal React test client as an example (that simply adds WASM support to the default `create-react-app` template). You can find the code for the test client [here](https://github.com/nalinbhardwaj/zordle/tree/main/test-client). I would recommend forking the test client for your own application and working from there.

The test client includes a clean WebWorker that interfaces with the Rust WASM package. Putting the interface in a WebWorker prevents blocking the main thread of the browser and allows for a clean interface from React/application logic. Checkout [`halo-worker.ts`](https://github.com/nalinbhardwaj/zordle/blob/main/test-client/src/halo-worker.ts) for the WebWorker code and see how you can interface with the web worker from React in [`App.tsx`](https://github.com/nalinbhardwaj/zordle/blob/main/test-client/src/App.tsx#L7-L26).

If you've done everything right so far, you should now be able to generate proofs and verify them in browser! In the case of Zordle, proof generation for a circuit with `K = 14` takes about a minute or so on my laptop. During proof generation, if you pop open the Chrome/Firefox task manager, you should additionally see something like this:

<img src="https://i.imgur.com/TpIIVJh.png" alt="Example halo2 proof generation in-browser" width="500">

Zordle and its test-client set the parallelism to the number of cores available on the machine by default. If you would like to reduce this, you can do so by changing the argument to [`initThreadPool`](https://github.com/nalinbhardwaj/zordle/blob/main/test-client/src/halo-worker.ts#L7).

If you'd prefer to use your own Worker/React setup, the code to [fetch and serialise parameters](https://github.com/nalinbhardwaj/zordle/blob/main/test-client/src/halo-worker.ts#L13), proofs and other instance and advice values may still be useful to look at!

## Safari

Note that `wasm-bindgen-rayon` library is not supported by Safari because it spawns Web Workers from inside another Web Worker. According to the relevant [Webkit issue](https://bugs.webkit.org/show_bug.cgi?id=25212), support for this feature had made it into Safari Technology Preview by November 2022, and indeed the [Release Notes for Safari Technology Preview Release 155](https://developer.apple.com/safari/technology-preview/release-notes/#r155) claim support, so it is worth checking whether this has made it into Safari if that is important to you.

## Debugging

Often, you'll run into issues with your Rust code and see that the WASM execution errors with `Uncaught (in promise) RuntimeError: unreachable`, a wholly unhelpful error for debugging. This is because the code is compiled in release mode which strips out error messages as a performance optimisation. To debug, you can build the WASM package in debug mode using the flag `--dev` with `wasm-pack build`. This will build in debug mode, slowing down execution significantly but allowing you to see any runtime error messages in the browser console. Additionally, you can install the [`console_error_panic_hook`](https://github.com/rustwasm/console_error_panic_hook) crate (as is done by Zordle) to also get helpful debug messages for runtime panics.

## Credits

This guide was written by [Nalin](https://twitter.com/nibnalin). Thanks additionally to [Uma](https://twitter.com/pumatheuma) and [Blaine](https://twitter.com/BlaineBublitz) for significant work on figuring out these steps. Feel free to reach out to me if you have trouble with any of these steps.


# 在 WASM 中使用 halo2

由于 halo2 是用 Rust 编写的，您可以将其编译为 WebAssembly (WASM)，从而允许在浏览器应用程序中使用电路的证明器和验证器。本教程将带您了解将电路编译为 WASM 所需的所有知识。

在本教程中，我们将参考 [Zordle](https://github.com/nalinbhardwaj/zordle) 的代码库，这是第一个基于 Halo 2 电路的 Web 应用程序之一。Zordle 是 ZK [Wordle](https://www.nytimes.com/games/wordle/index.html)，其中电路将玩家的输入单词和玩家的共享网格（灰色、黄色和绿色方块）作为建议值，并验证它们是否正确匹配。因此，证明验证玩家知道输出共享表的“原像”，然后可以仅使用 ZK 证明进行验证。

## 电路代码设置

第一步是在 Rust 中创建与浏览器应用程序交互的函数。对于证明器，这通常会输入一些版本的建议和实例数据，使用它生成完整的见证，然后输出证明。对于验证器，这通常会输入证明和一些版本的实例，然后输出一个布尔值，指示证明是否正确验证。

在 Zordle 的案例中，此代码包含在 [wasm.rs](https://github.com/nalinbhardwaj/zordle/blob/main/circuits/src/wasm.rs) 中，并由两个主要函数组成：

### 证明器

```rust,ignore
#[wasm_bindgen]
pub async fn prove_play(final_word: String, words_js: JsValue, params_ser: JsValue) -> JsValue {
  // 步骤：
  // - 反序列化函数参数
  // - 使用单词生成实例和建议列
  // - 实例化电路并生成见证
  // - 从参数生成证明密钥
  // - 创建证明
}
```

虽然具体输入及其序列化将取决于您的电路和 Web 应用程序设置，但值得注意的是 Zordle 案例中的格式，因为您的用例可能类似：

此函数将用户目标的 `final_word` 和他们尝试使用的单词（以 `words_js` 的形式）作为输入。它还输入电路的参数，这些参数在 `params_ser` 中序列化。我们将在下面的 [参数](#params) 部分中详细讨论这一点。

请注意，函数参数以 `wasm_bindgen` 兼容的格式传递：`String` 和 `JsValue`。`JsValue` 类型来自 [`Serde`](https://serde.rs) 库。您可以在[此处](https://rustwasm.github.io/wasm-bindgen/reference/arbitrary-data-with-serde.html#serializing-and-deserializing-arbitrary-data-into-and-from-jsvalue-with-serde)找到有关此类型及其使用方法的更多详细信息。

输出是使用 Serde 转换为 `JSValue` 的 `Vec<u8>`。稍后将其作为验证器函数的输入传递。

### 验证器

```rust,ignore
#[wasm_bindgen]
pub fn verify_play(final_word: String, proof_js: JsValue, diffs_u64_js: JsValue, params_ser: JsValue) -> bool {
  // 步骤：
  // - 反序列化函数参数
  // - 使用列的 diffs 表示生成实例列
  // - 使用参数生成验证密钥
  // - 验证证明
}
```

与证明器类似，我们输入并输出一个布尔值 true/false，指示证明的正确性。`diffs_u64_js` 对象是一个 2D JS 数组，包含每个单元格的值，指示颜色：灰色、黄色或绿色。这些用于组装电路的实例列。

### 参数

此外，证明器和验证器函数都输入 `params_ser`，这是多项式承诺方案的公共参数的序列化形式。这些作为输入传递（而不是在证明/验证函数中重新生成），作为性能优化，因为这些参数仅基于电路的 `K` 值。我们可以将它们单独存储在静态 Web 服务器上，并将其作为输入传递给 WASM。要生成这些参数的二进制序列化形式（在 WASM 函数之外单独生成），您可以运行如下代码：

```rust,ignore
fn write_params(K: u32) {
    let mut params_file = File::create("params.bin").unwrap();
    let params: Params<EqAffine> = Params::new(K);
    params.write(&mut params_file).unwrap();
}
```

稍后，我们可以从 Web 服务器读取 `params.bin` 文件，以字节序列化格式作为 `Uint8Array` 传递给 WASM 作为 `params_ser`，可以使用 [`js_sys`](https://docs.rs/js-sys/latest/js_sys/) 库在 Rust 中反序列化。

理想情况下，未来我们能够直接序列化并处理电路的证明密钥和验证密钥，而不是序列化参数，但目前库不支持这一点，并在问题 [#449](https://github.com/zcash/halo2/issues/449) 和 [#443](https://github.com/zcash/halo2/issues/443) 中跟踪。

## Rust 和 WASM 环境设置

通常，Rust 代码使用 [`wasm-pack`](https://developer.mozilla.org/en-US/docs/WebAssembly/Rust_to_wasm) 工具编译为 WASM，只需更改一些构建命令即可。然而，在 halo2 证明器/验证器函数的情况下，我们需要对构建过程进行一些额外的更改。特别是，有两个主要更改：

- **并行性**：halo2 使用 `rayon` 库进行并行化，而 WASM 不直接支持。然而，Chrome 团队有一个适配器，可以使用浏览器中的 Web Workers 启用类似 rayon 的并行化：[`wasm-bindgen-rayon`](https://github.com/GoogleChromeLabs/wasm-bindgen-rayon)。我们将使用它来启用 WASM 证明器/验证器中的并行化。
- **WASM 最大内存**：`wasm-bindgen` 的默认内存限制为 2GB，这对于运行大型电路（`K` > 10 左右）的 halo2 证明器来说是不够的。我们需要将此限制增加到 WASM 允许的最大值（4GB！）以支持更大的电路（最多 `K = 15` 左右）。

首先，将 WASM 接口函数特定的所有依赖项添加到您的 `Cargo.toml` 文件中。您可以通过使用 WASM 目标功能标志来限制依赖项仅用于 WASM 编译。在 Zordle 的案例中，[如下所示](https://github.com/nalinbhardwaj/zordle/blob/main/circuits/Cargo.toml#L24)：

```toml
[target.'cfg(target_family = "wasm")'.dependencies]
getrandom = { version = "0.2", features = ["js"]}
wasm-bindgen = { version = "0.2.81", features = ["serde-serialize"]}
console_error_panic_hook = "0.1.7"
rayon = "1.5"
wasm-bindgen-rayon = { version = "1.0"}
web-sys = { version = "0.3", features = ["Request", "Window", "Response"] }
wasm-bindgen-futures = "0.4"
js-sys = "0.3"
```

接下来，将 `wasm-bindgen-rayon` 集成到我们的代码中。[该库的 README](https://github.com/GoogleChromeLabs/wasm-bindgen-rayon) 详细介绍了如何做到这一点。特别是，请注意 [Rust 编译管道的更改](https://github.com/GoogleChromeLabs/wasm-bindgen-rayon#using-config-files)。您需要切换到 Rust 的 nightly 版本并启用对 WASM 原子操作的支持。此外，记得在 Rust 代码中导出 [`init_thread_pool`](https://github.com/GoogleChromeLabs/wasm-bindgen-rayon#setting-up)。

接下来，我们将提高 `wasm-pack` 的默认 2GB 最大内存限制。为此，请在 `.cargo/config` 文件中为 wasm 目标添加 `"-C", "link-arg=--max-memory=4294967296"` Rust 标志。在设置 `wasm-bindgen-rayon` 和内存增加后，`.cargo/config` 文件现在应如下所示：

```toml
[target.wasm32-unknown-unknown]
rustflags = ["-C", "target-feature=+atomics,+bulk-memory,+mutable-globals", "-C", "link-arg=--max-memory=4294967296"]
```

感谢 [@mattgibb](https://github.com/mattgibb) 在 [这个 GitHub 问题](https://github.com/rustwasm/wasm-bindgen/issues/2498#issuecomment-801498175) 中记录了这种增加最大内存的深奥更改。

: 题外话，但令我惊讶的是，WASM 有一个硬性的最大内存限制为 4GB。这是因为 WASM 目前是 32 位架构，这对于如此新的、面向未来的汇编语言来说相当令人惊讶。不过，有一些开放提案 [将 WASM 迁移到更大的地址空间](https://github.com/WebAssembly/memory64)。

现在我们已经设置好了 Rust，您应该能够简单地使用 `wasm-pack build --target web --out-dir pkg` 构建 WASM 包，并在您的 Web 应用程序中使用输出的 WASM 包。

## Web 应用程序设置

Zordle 附带了一个最小的 React 测试客户端作为示例（它只是为默认的 `create-react-app` 模板添加了 WASM 支持）。您可以在 [此处](https://github.com/nalinbhardwaj/zordle/tree/main/test-client) 找到测试客户端的代码。我建议为您的应用程序分叉测试客户端并从中开始工作。

测试客户端包括一个与 Rust WASM 包交互的干净 WebWorker。将接口放在 WebWorker 中可以防止阻塞浏览器的主线程，并允许从 React/应用程序逻辑中获得干净的接口。查看 [`halo-worker.ts`](https://github.com/nalinbhardwaj/zordle/blob/main/test-client/src/halo-worker.ts) 中的 WebWorker 代码，并了解如何在 [`App.tsx`](https://github.com/nalinbhardwaj/zordle/blob/main/test-client/src/App.tsx#L7-L26) 中与 WebWorker 交互。

如果您到目前为止一切都做对了，您现在应该能够在浏览器中生成证明并验证它们！在 Zordle 的案例中，在我的笔记本电脑上生成 `K = 14` 的电路的证明大约需要一分钟左右。在证明生成期间，如果您打开 Chrome/Firefox 任务管理器，您应该会看到类似这样的内容：

<img src="https://i.imgur.com/TpIIVJh.png" alt="Example halo2 proof generation in-browser" width="500">

Zordle 及其测试客户端默认将并行性设置为机器上可用的核心数。如果您想减少这一点，可以通过更改 [`initThreadPool`](https://github.com/nalinbhardwaj/zordle/blob/main/test-client/src/halo-worker.ts#L7) 的参数来实现。

如果您更喜欢使用自己的 Worker/React 设置，[获取和序列化参数](https://github.com/nalinbhardwaj/zordle/blob/main/test-client/src/halo-worker.ts#L13)、证明以及其他实例和建议值的代码可能仍然值得一看！

## Safari

请注意，`wasm-bindgen-rayon` 库不受 Safari 支持，因为它从另一个 WebWorker 中生成 Web Workers。根据相关的 [Webkit 问题](https://bugs.webkit.org/show_bug.cgi?id=25212)，对此功能的支持已于 2022 年 11 月进入 Safari Technology Preview，实际上 [Safari Technology Preview Release 155 的发布说明](https://developer.apple.com/safari/technology-preview/release-notes/#r155) 声称支持，因此如果这对您很重要，值得检查是否已进入 Safari。

## 调试

通常，您会遇到 Rust 代码的问题，并看到 WASM 执行错误 `Uncaught (in promise) RuntimeError: unreachable`，这是一个完全无助于调试的错误。这是因为代码是在发布模式下编译的，该模式会剥离错误消息作为性能优化。要调试，您可以使用 `wasm-pack build` 的 `--dev` 标志在调试模式下构建 WASM 包。这将在调试模式下构建，显著减慢执行速度，但允许您在浏览器控制台中查看任何运行时错误消息。此外，您可以安装 [`console_error_panic_hook`](https://github.com/rustwasm/console_error_panic_hook) crate（如 Zordle 所做的那样），以获取运行时 panic 的有用调试消息。

## 致谢
本指南由[Nalin]（https://twitter.com/nibnalin）撰写。另外还要感谢[Uma]（https://twitter.com/pumatheuma）和[Blaine]（https://twitter.com/BlaineBublitz）为解决这些步骤所做的重要工作。如果你在这些步骤中有任何问题，请随时联系我。
