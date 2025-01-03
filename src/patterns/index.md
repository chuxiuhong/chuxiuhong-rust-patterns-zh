# 设计模式

[设计模式](https://en.wikipedia.org/wiki/Software_design_pattern)是"在软件设计中，针对特定上下文中经常出现的问题提供的一种可重用的通用解决方案"。设计模式是描述编程语言文化的绝佳方式。设计模式与编程语言密切相关 - 在一种语言中被视为模式的解决方案，在另一种语言中可能因为某些语言特性而变得不必要，或者由于缺少某些特性而无法实现。

如果过度使用，设计模式可能会给程序增加不必要的复杂性。然而，它们是分享编程语言中级和高级知识的绝佳方式。

## Rust 中的设计模式

Rust 拥有许多独特的特性。这些特性通过消除整类问题为我们带来了巨大的好处。其中一些特性也是 *独特* 于 Rust 的模式。

## YAGNI

YAGNI 是 `You Aren't Going to Need It`（你不会需要它）的缩写。这是编写代码时需要遵循的一个重要软件设计原则。

> 我写过最好的代码是那些我从未写过的代码。

如果我们将 YAGNI 原则应用到设计模式中，我们会发现 Rust 的特性使得许多模式变得不再必要。例如，由于我们可以直接使用 [traits](https://doc.rust-lang.org/book/traits.html)，所以在 Rust 中就不需要 [strategy pattern](https://en.wikipedia.org/wiki/Strategy_pattern)（策略模式）。
