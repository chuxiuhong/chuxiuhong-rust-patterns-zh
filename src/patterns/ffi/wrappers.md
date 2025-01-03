# 使用包装器整合类型

## 描述

这种模式旨在优雅地处理多个相关类型，同时最小化内存不安全的暴露面。

Rust 别名规则的基石之一是生命周期（lifetimes）。这确保了许多类型之间的访问模式可以实现内存安全，包括数据竞争安全。

然而，当 Rust 类型被导出到其他语言时，它们通常会被转换为指针。在 Rust 中，指针意味着"用户管理指针所指对象的生命周期"。避免内存不安全是用户的责任。

因此需要对用户代码有一定程度的信任，特别是在使用后释放（use-after-free）这种 Rust 无法处理的情况。不过，某些 API 设计比其他设计对其他语言编写的代码要求更高。

风险最低的 API 是"整合包装器"，它将所有可能的对象交互都整合到一个"包装器类型"中，同时保持 Rust API 的整洁。

## 代码示例

为了理解这一点，让我们看一个经典的需要导出的 API 示例：集合的迭代。

该 API 看起来是这样的：

1. 使用 `first_key` 初始化迭代器
2. 每次调用 `next_key` 都会推进迭代器
3. 如果迭代器已到达末尾，调用 `next_key` 将不会做任何事情
4. 如上所述，迭代器被"包装进"集合中（这与原生 Rust API 不同）

如果迭代器高效地实现了 `nth()`，那么可以将其设为每个函数调用的临时对象：

```rust,ignore
struct MySetWrapper {
    myset: MySet,
    iter_next: usize,
}

impl MySetWrapper {
    pub fn first_key(&mut self) -> Option<&Key> {
        self.iter_next = 0;
        self.next_key()
    }
    pub fn next_key(&mut self) -> Option<&Key> {
        if let Some(next) = self.myset.keys().nth(self.iter_next) {
            self.iter_next += 1;
            Some(next)
        } else {
            None
        }
    }
}
```

因此，包装器很简单，且不包含任何 `unsafe` 代码。

## 优点

这使得 API 使用起来更安全，避免了类型之间的生命周期问题。关于这种方法避免的优点和陷阱，请参见[基于对象的 API](./export.md)。

## 缺点

通常，包装类型是相当困难的，有时做出一些 Rust API 的妥协会使事情变得更容易。

例如，考虑一个不能高效实现 `nth()` 的迭代器。在这种情况下，值得添加特殊逻辑来使对象内部处理迭代，或者支持一个仅供外部函数 API 使用的、更高效的访问模式。

### 尝试包装迭代器（及其失败）

要正确地将任何类型的迭代器包装到 API 中，包装器需要做 C 版本代码会做的事情：擦除迭代器的生命周期，并手动管理它。

简单来说，这是*极其*困难的。

这里展示了*一个*陷阱的示例。

`MySetWrapper` 的第一个版本可能是这样的：

```rust,ignore
struct MySetWrapper {
    myset: MySet,
    iter_next: usize,
    // 从 transmuted Box<KeysIter + 'self> 创建
    iterator: Option<NonNull<KeysIter<'static>>>,
}
```

使用 `transmute` 来延长生命周期，并使用指针来隐藏它，这已经很丑陋了。但情况会变得更糟：*任何其他操作都可能导致 Rust 的未定义行为*。

考虑到包装器中的 `MySet` 可能在迭代过程中被其他函数操作，比如在正在迭代的键上存储新值。API 并不阻止这种行为，实际上一些类似的 C 库就是这样设计的。

一个简单的 `myset_store` 实现可能是：

```rust,ignore
pub mod unsafe_module {

    // 其他模块内容

    pub fn myset_store(myset: *mut MySetWrapper, key: datum, value: datum) -> libc::c_int {
        // 不要使用这段代码。它是不安全的，用于演示问题。

        let myset: &mut MySet = unsafe {
            // 安全性：糟糕，这里会发生未定义行为！
            &mut (*myset).myset
        };

        /* ...检查并转换 key 和 value 数据... */

        match myset.store(casted_key, casted_value) {
            Ok(_) => 0,
            Err(e) => e.into(),
        }
    }
}
```

如果在调用此函数时迭代器存在，我们就违反了 Rust 的别名规则。根据 Rust 的规则，这个代码块中的可变引用必须对对象有*独占*访问权。如果迭代器存在，就不是独占的，所以我们就有了`未定义行为`！[^1]

为了避免这种情况，我们必须有办法确保可变引用确实是独占的。这基本上意味着在迭代器存在时清除它的共享引用，然后重新构建它。在大多数情况下，这仍然比 C 版本效率低。

有人可能会问：C 是如何更高效地做到这一点的？答案是，它取巧了。Rust 的别名规则是问题所在，而 C 对其指针简单地忽略了这些规则。作为交换，在手册中经常会看到代码被声明为"在某些或所有情况下线程不安全"。事实上，[GNU C 库](https://manpages.debian.org/buster/manpages/attributes.7.en.html)有一整套专门用于并发行为的词汇表！

Rust 更倾向于始终保持所有内容的内存安全，这既是为了安全性，也是为了获得 C 代码无法达到的优化。被拒绝使用某些捷径是 Rust 程序员需要付出的代价。

[^1]: 对于那些对此感到困惑的 C 程序员来说，迭代器不需要在这段代码*执行期间*被读取就会导致未定义行为。独占性规则还启用了编译器优化，这可能导致迭代器的共享引用观察到不一致的结果（例如，为了效率而进行的栈溢出或重排指令）。这些观察可能在创建可变引用*之后的任何时候*发生。
