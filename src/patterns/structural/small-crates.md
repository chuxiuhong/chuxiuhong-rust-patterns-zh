# 倾向使用小型 crates

## 描述

倾向于使用能把一件事做好的小型 crates。

相比 C 或 C++，Cargo 和 crates.io 使添加第三方库变得更加容易。此外，由于 crates.io 上的包在发布后不能被编辑或删除，所以任何现在能工作的构建在将来也应该能继续工作。我们应该充分利用这些工具，使用更小、更精细的依赖。

## 优势

- 小型 crates 更容易理解，并且鼓励更模块化的代码。
- Crates 允许在项目之间重用代码。例如，`url` crate 最初是作为 Servo 浏览器引擎的一部分开发的，但后来在项目之外得到了广泛使用。
- 由于 Rust 的编译单元是 crate，将项目拆分为多个 crates 可以让更多的代码并行构建。

## 劣势

- 这可能导致"依赖地狱"，即当一个项目同时依赖于某个 crate 的多个冲突版本时。例如，`url` crate 同时存在 1.0 和 0.5 版本。由于 `url:1.0` 中的 `Url` 和 `url:0.5` 中的 `Url` 是不同的类型，使用 `url:0.5` 的 HTTP 客户端将不会接受使用 `url:1.0` 的网页抓取器产生的 `Url` 值。
- crates.io 上的包没有经过筛选。一个 crate 可能写得很差，文档不够有帮助，或者甚至可能是恶意的。
- 两个小型 crates 可能比一个大型 crate 的优化程度更低，因为编译器默认不执行链接时优化（LTO）。

## 示例

[`url`](https://crates.io/crates/url) crate 提供了处理 URL 的工具。

[`num_cpus`](https://crates.io/crates/num_cpus) crate 提供了一个查询机器 CPU 数量的函数。

[`ref_slice`](https://crates.io/crates/ref_slice) crate 提供了将 `&T` 转换为 `&[T]` 的函数。（历史示例）

## 另见

- [crates.io: Rust 社区的 crate 托管平台](https://crates.io/)
