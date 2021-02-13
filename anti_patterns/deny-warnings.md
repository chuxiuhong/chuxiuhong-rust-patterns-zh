# `#![deny(warnings)]`

## 说明

一个善意的库作者想要确保他们的代码在编译时不会产生警告。因此他们在库里标注以下内容：

## 示例

```rust
#![deny(warnings)]

// 一切安好
```

## 优点

它很短，如果有什么错误就停止编译。

## 缺点

通过禁用编译器生成警告，库的作者放弃了Rust的稳定性。有时新的特性或者旧的不合格的特性需要被更改，因此，将会在一段宽限期内给出警告，之后变成禁用。

举例来说，一个类型可以有两个具有相同方法的实现。这被认为是一个坏主意，但是为了顺利过渡，引入 `overlapping-inherent-impls`提示来警告那些在将来版本中出现严重错误的人。

而且有时API会被弃用，所以使用它们会发出警告。

所有的这些在改变时都可能破坏编译过程。

此外，除非这个删除注释，否则不能再使用提供额外警告的库。（例如rust-clippy）这可以通过[--cap-lints]缓解。`--cap-lints=warn`命令行参数将所有的`deny`提示的错误转换为警告。但是请注意`forbid`警告比`deny`要更强，因此不能把`forbid`级别的提示改写为低于错误的任何级别。因此，`forbid`提示仍将停止编译。

## 替代方案

解决这个问题有两种方法：第一种，我们可以将编译设置与代码解耦；第二种，我们可以显式地命名要拒绝的警告。

下面这个命令行参数将会带着所有关闭的警告进行编译：

```RUSTFLAGS="-D warnings" cargo build```

任何独立开发者都可以这样做（或者设置到持续集成工具，如Travis，但是记住当某些内容发生变化时，可能会破坏编译）。

或者，我们可以指定我们想要在代码中关闭的警告。下面是警告提示列表（Rustc 1.48.0）：

```rust,ignore
#[deny(bad-style,
       const-err,
       dead-code,
       improper-ctypes,
       non-shorthand-field-patterns,
       no-mangle-generic-items,
       overflowing-literals,
       path-statements ,
       patterns-in-fns-without-body,
       private-in-public,
       unconditional-recursion,
       unused,
       unused-allocation,
       unused-comparisons,
       unused-parens,
       while-true)]
```

此外，下面的提示是推荐关闭的：

```rust,ignore
#[deny(missing-debug-implementations,
       missing-docs,
       trivial-casts,
       trivial-numeric-casts,
       unused-extern-crates,
       unused-import-braces,
       unused-qualifications,
       unused-results)]
```

有时可能需要增加`missing-copy-implementations`到清单中。

请注意，我们没有关闭`deprecated`提示，因为可以肯定的是，将来会有更多不推荐的API。

## 参阅

- [deprecate attribute] documentation
- Type `rustc -W help` for a list of lints on your system. Also type
`rustc --help` for a general list of options
- [rust-clippy] is a collection of lints for better Rust code

[rust-clippy]: https://github.com/Manishearth/rust-clippy
[deprecate attribute]: https://doc.rust-lang.org/reference/attributes.html#deprecation
[--cap-lints]: https://doc.rust-lang.org/rustc/lints/levels.html#capping-lints
