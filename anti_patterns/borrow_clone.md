# Clone过借用检查

## 说明

借用检查阻止了Rust用户开发不安全的代码，以此保证：只存在一个可变引用，或者（许多）不可变引用。如果编写的代码不符合这些条件，而开发者通过克隆变量来解决编译器错误，就会产生这种反模式。

## 示例

```rust
// 定义任意变量
let mut x = 5;

// 借用 `x`（先clone）
let y = &mut (x.clone());

// 由于 x.clone(), x 并未被借用, 这行代码可以运行。
println!("{}", x);

// 用这个借用做点什么，防止因Rust优化直接砍掉这个借用
*y += 1;
```

## 出发点

用这种模式来解决借用检查令人困惑的问题是很诱人的，特别是对于初学者来说。然而，这有严重的后果。使用`.clone()`会导致数据被复制。两者之间的任何变化都不会同步——因为会有两个完全独立的变量存在。

有种特殊情况—— `Rc<T>` 被设计为智能处理 `clone` 。它在内部确切管理着一份数据的副本，clone它只会clone引用。

还有`Arc<T>`，它提供堆分配类型T的共享所有权。对`Arc`调用`.clone()`会得到新的`Arc`实例，它指向和源`Arc`相同的栈分配，增加引用计数。

一般来说，应该经过深思熟虑，充分了解其后果再clone。如果用clone消除借用检查器报错，很可能你使用了这种反模式。

即使`.clone()`是坏模式的预兆，有时**编写低效率的代码是可以的**，比如这些情况时：

- 开发者不大懂所有权
- 代码没有什么速度或内存限制（如黑客马拉松项目或原型）。
- 借用检查器太复杂了，而你更愿意优化可读性，而非性能

如果你怀疑做了不必要的clone，在评估是否需要clone之前，先去弄懂[《Rust Book》的所有权章节](https://doc.rust-lang.org/book/ownership.html)。

此外要保证一直给你的项目跑`cargo clippy`，它可以判断一些`.clone()`调用不必要的情况，比如[甲](https://rust-lang.github.io/rust-clippy/master/index.html#redundant_clone)，[乙](https://rust-lang.github.io/rust-clippy/master/index.html#clone_on_copy)，[丙](https://rust-lang.github.io/rust-clippy/master/index.html#map_clone)或者[丁](https://rust-lang.github.io/rust-clippy/master/index.html#clone_double_ref).

## 参见

- [`mem::{take(_), replace(_)}`在被更改的枚举中保持拥有的值](../idioms/mem-replace.md)。
- [`Rc<T>`文档，它智能地处理.clone()](http://doc.rust-lang.org/std/rc/)
- [`Arc<T>`文档 线程安全的引用计数指针](https://doc.rust-lang.org/std/sync/struct.Arc.html)
- [Rust所有权小窍门](https://web.archive.org/web/20210120233744/https://xion.io/post/code/rust-borrowchk-tricks.html)
