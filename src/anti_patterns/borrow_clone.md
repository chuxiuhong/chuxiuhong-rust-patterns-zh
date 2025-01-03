# 为了满足借用检查器而克隆

## 描述

借用检查器通过确保以下条件之一来防止 Rust 用户编写不安全的代码：要么只存在一个可变引用，要么可能存在多个但都是不可变引用。如果编写的代码不符合这些条件，当开发者通过克隆变量来解决编译器错误时，就会出现这种反模式。

## Example

```rust
// 定义任意变量
let mut x = 5;

// 借用 `x` -- 但先克隆它
let y = &mut (x.clone());

// 如果没有前面两行的 x.clone()，这一行会编译失败，因为
// x 已经被借用了
// 多亏了 x.clone()，x 从未被借用，所以这一行可以运行
println!("{x}");

// 对借用执行一些操作以防止 rust 将其优化掉
*y += 1;
```

## 动机

对于初学者来说，使用这种模式来解决令人困惑的借用检查器问题很有诱惑力。然而，这会带来严重的后果。使用 `.clone()` 会导致数据被复制。两个副本之间的任何更改都不会同步 -- 就像存在两个完全独立的变量一样。

有一些特殊情况 -- `Rc<T>` 被设计为智能地处理克隆。它在内部只管理数据的一个副本，克隆它只会克隆引用。

还有 `Arc<T>`，它提供了对堆上分配的 T 类型值的共享所有权。在 `Arc` 上调用 `.clone()` 会产生一个新的 `Arc` 实例，该实例指向与源 `Arc` 相同的堆分配，同时增加引用计数。

一般来说，克隆应该是经过深思熟虑的，并且完全理解其后果。如果使用克隆只是为了消除借用检查器错误，这很可能表明正在使用这种反模式。

尽管 `.clone()` 表明这是一种不好的模式，但在某些情况下**编写效率不高的代码是可以的**，比如：

- 开发者还是所有权概念的新手
- 代码对速度或内存没有很高的要求（比如黑客马拉松项目或原型）
- 满足借用检查器非常复杂，你更倾向于优化可读性而不是性能

如果怀疑存在不必要的克隆，应该在评估克隆是否必要之前，完全理解
[Rust Book 中关于所有权的章节](https://doc.rust-lang.org/book/ownership.html)。

另外，一定要在项目中运行 `cargo clippy`，它可以检测一些不需要 `.clone()` 的情况，比如
[1](https://rust-lang.github.io/rust-clippy/master/index.html#redundant_clone)、
[2](https://rust-lang.github.io/rust-clippy/master/index.html#clone_on_copy)、
[3](https://rust-lang.github.io/rust-clippy/master/index.html#map_clone) 或
[4](https://rust-lang.github.io/rust-clippy/master/index.html#clone_double_ref)。

## 另请参阅

- [使用 `mem::{take(_), replace(_)}` 在改变的枚举中保持所有权值](../idioms/mem-replace.md)
- [智能处理 .clone() 的 `Rc<T>` 文档](http://doc.rust-lang.org/std/rc/)
- [线程安全的引用计数指针 `Arc<T>` 文档](https://doc.rust-lang.org/std/sync/struct.Arc.html)
- [Rust 中的所有权技巧](https://web.archive.org/web/20210120233744/https://xion.io/post/code/rust-borrowchk-tricks.html)
