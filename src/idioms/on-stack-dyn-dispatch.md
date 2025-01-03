# 栈上动态分发

## 描述

我们可以对多个值进行动态分发，但是为了实现这一点，我们需要声明多个变量来绑定不同类型的对象。为了根据需要延长生命周期，我们可以使用延迟条件初始化，如下所示：

## 示例

```rust
use std::io;
use std::fs;

# fn main() -> Result<(), Box<dyn std::error::Error>> {
# let arg = "-";

// 我们需要描述类型以实现动态分发
let readable: &mut dyn io::Read = if arg == "-" {
    &mut io::stdin()
} else {
    &mut fs::File::open(arg)?
};

// 在这里从 `readable` 读取

# Ok(())
# }
```

## 动机

Rust 默认会进行单态化（monomorphisation）。这意味着代码会为每个使用它的类型生成一个副本，并独立进行优化。虽然这在热路径上能产生非常快的代码，但在性能不是关键的地方会导致代码膨胀，从而增加编译时间和缓存使用。

幸运的是，Rust 允许我们使用动态分发，但我们必须显式地请求它。

## 优势

我们不需要在堆上分配任何内容。我们既不需要初始化后面不会使用的内容，也不需要将后续的整个代码单态化以同时适用于 `File` 或 `Stdin`。

## 劣势

在 Rust 1.79.0 之前，代码需要两个带有延迟初始化的 `let` 绑定，这比基于 `Box` 的版本有更多的移动部分：

```rust,ignore
// 我们仍然需要指定类型以进行动态分发
let readable: Box<dyn io::Read> = if arg == "-" {
    Box::new(io::stdin())
} else {
    Box::new(fs::File::open(arg)?)
};
// 在这里从 `readable` 读取
```

幸运的是，这个劣势现在已经消除了。太好了！

## 讨论

从 Rust 1.79.0 开始，编译器会自动在函数作用域内尽可能地延长 `&` 或 `&mut` 中临时值的生命周期。

这意味着我们可以在这里简单地使用 `&mut` 值，而不用担心将内容放入某个 `let` 绑定中（这在之前的解决方案中是必需的，用于延迟初始化）。

我们仍然为每个值都有一个位置（即使该位置是临时的），编译器知道每个值的大小，并且每个被借用的值的生命周期都长于从它借用的所有引用。

## 另请参阅

- [析构函数中的终结化](dtor-finally.md)和[RAII guards](../patterns/behavioural/RAII.md)可以从对生命周期的严格控制中受益。
- 对于条件填充的 `Option<&T>` （可变）引用，可以直接初始化一个 `Option<T>` 并使用其 [`.as_ref()`] 方法来获取一个可选引用。

[`.as_ref()`]: https://doc.rust-lang.org/std/option/enum.Option.html#method.as_ref
