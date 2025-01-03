# 简介

## 参与贡献

如果你有兴趣为本书做出贡献，请查看
[contribution guidelines](https://github.com/rust-unofficial/patterns/blob/master/CONTRIBUTING.md)。

## 新闻

- **2024-03-17**: 现在你可以从[这个链接](https://rust-unofficial.github.io/patterns/rust-design-patterns.pdf)下载本书的 PDF 格式。

## 设计模式

在软件开发中，我们经常遇到一些问题，这些问题无论在什么环境下出现都具有相似性。
虽然实现细节对解决当前任务至关重要，但我们可以从这些特定情况中抽象出普遍适用的通用实践。

设计模式是工程中针对重复出现问题的一系列可重用且经过验证的解决方案。它们使我们的软件更加模块化、
可维护且易于扩展。此外，这些模式为开发者提供了一种通用语言，使其成为团队解决问题时进行有效沟通的
绝佳工具。

请记住：每种模式都有其自身的权衡。关注你选择特定模式的原因，而不是仅仅关注如何实现它，这一点
至关重要。[^1]

## Rust 中的设计模式

Rust 不是面向对象的语言，它的所有特性的组合，如函数式编程元素、强大的类型系统和借用检查器，
使其独具特色。正因如此，Rust 的设计模式与其他传统的面向对象编程语言有所不同。这就是我们
决定编写这本书的原因。我们希望你能享受阅读的过程！本书分为三个主要章节：

- [惯用法](./idioms/index.md)：编码时需要遵循的准则。这些是社区的社交规范。只有在有充分理由的
  情况下才应该打破这些规范。
- [设计模式](./patterns/index.md)：解决编码中常见问题的方法。
- [反模式](./anti_patterns/index.md)：虽然也是解决编码中常见问题的方法，但与设计模式不同的是，
  反模式会带来更多问题。

[^1]: https://web.archive.org/web/20240124025806/https://www.infoq.com/podcasts/software-architecture-hard-parts/
