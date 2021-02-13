# 设计模式

[设计模式](https://en.wikipedia.org/wiki/Software_design_pattern)设计模式（design pattern）是对软件设计中普遍存在（反复出现）的各种问题，所提出的解决方案。设计模式是用来描述一门编程语言文化的好标准。设计模式与编程语言息息相关，一门语言中的模式可能在另一种语言中没什么必要，因为语言可能自身特性就能解决问题。或者可能在另一门语言中由于缺少某些特性，压根就实现不了。

设计模式如果滥用，那将会增加程序不必要的复杂性。不过设计模式倒可以用来分享关于一门语言深层次和进阶水平的知识。

## Rust中的设计模式

Rust有很多独特的特性。这些特性消除了大量的问题，给我们极大的帮助。有些还是Rust的独特设计模式。

## YAGNI

如果你还不了解这个词，YAGNI是不过早添加功能的缩写（`You Aren't Going to Need It`）。这是写代码时的重要原则。

> 我曾写过的最好的代码是我没写过的代码

如果我们将YAGNI原则应用到设计模式中，我们可以发现Rust的特性能让我们省掉很多不必要的模式。例如，不再需要[策略模式](https://en.wikipedia.org/wiki/Strategy_pattern)。在Rust里可以直接用[traits](https://doc.rust-lang.org/book/traits.html)。

TODO: Maybe include some code to illustrate the traits.
