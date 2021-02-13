# 类型合并封装

## 说明

这个模式是被设计用来在最小化内存不安全代码区域的情况下，支持优雅地处理多种相关类型。

Rust的别名规则的基石之一就是生命周期。其确保了多种在类型间的访问模式是内存安全的，也包括安全的数据竞争。

不过当Rust 的类型导出到其他语言时，通常转换为指针。在Rust中，指针相当于“用户管理指针指向对象的生命周期”。谁使用谁负责避免内存不安全的情况。

因此需要对用户代码有一定程度的信任，特别是在释放内存之后，Rust对此无能为力。不过，一些API设计相比于其他设计来说，对另一种语言编写的代码造成更大的负担。

风险最小的API设计是“合并包装器”，所有可能的互动都合并到一个“包装器类型”中，保持Rust的API干净。

## 代码示例

为了便于理解，让我们看看一个经典的API导出的例子：在集合中循环访问。

API看起来像这样:

1. 迭代器用`first_key`初始化。
2. 每次调用`next_key`将会递增迭代器。
3. Calls to `next_key` if the iterator is at the end will do nothing.
4. 当迭代器到尾时，调用`next_key`将什么都不做。
5. 像前面所说，迭代器将会被包装进集合中（不像Rust的原生API）

如果迭代器高效实现了`nth()`，就可以实现对每个函数调用都是很快的：

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

因此，包装器实现简单并且不包含任何`unsafe`代码。

## 优点

这使得API使用起来更安全，避免了在类型间交互时的生命周期问题。关于更多的优点和避免的陷阱请看 [基于对象的API](./ffi-export.md)。

## 缺点

包装类型常常是困难的，并且有时Rust的API做出妥协将会使事情更容易。

举例来说，想想一个没有高效实现`nth()`的迭代器。它肯定需要写特殊的逻辑来保证对象处理循环全在内部，或者单独支持一个不同的访问模式仅用来做外部语言访问。

### 尝试包装迭代器 (并且失败了)

为了正确地包装类型，包装器将会实现C语言版本的代码要做的事：擦除迭代器的生命周期，手动管理其生命周期。

简单地说，这是离谱的难。

下面仅仅是其中一个陷阱的说明。

`MySetWrapper`的第一个版本像下面这样：

```rust,ignore
struct MySetWrapper {
    myset: MySet,
    iter_next: usize,
    // created from a transmuted Box<KeysIter + 'self>
    iterator: Option<NonNull<KeysIter<'static>>>,
}
```

用`transmute`来延长生命周期，然后用一个指针来隐藏它，这就够丑陋的。不过它还有更坏的：任何其他的操作将会导致Rust的`未定义行为`(undefined behavior)。

在包装器内的`MySet`将会被其他函数在循环时操控，例如存储一个重复的新值。而API无法阻止这一点，并且事实上一些相似的C语言库也预期如此。

一个`myset_store` 的简单实现如下：

```rust,ignore
pub mod unsafe_module {

    // other module content

    pub fn myset_store(
        myset: *mut MySetWrapper,
        key: datum,
        value: datum) -> libc::c_int {

        // DO NOT USE THIS CODE. IT IS UNSAFE TO DEMONSTRATE A PROLBEM.

        let myset: &mut MySet = unsafe { // SAFETY: whoops, UB occurs in here!
            &mut (*myset).myset
        };

        /* ...check and cast key and value data... */

        match myset.store(casted_key, casted_value) {
            Ok(_) => 0,
            Err(e) => e.into()
        }
    }
}
```

当函数调用时迭代器已经存在，我们将违背Rust的一个别名规则。根据Rust的规则，在这段代码中的可变引用必须独占。如果迭代器已经存在，它就不是独占的，所以我们会有`未定义行为`！[^1]

为了避免这种情况的发生，我们必须有一种确保可变引用独占的方法。这基本相当于当迭代器存在时清除迭代器的共享引用，然后重新创建它。在绝大多数情况下，这还是比C语言版本的效率更低。

一些人可能会问：C语言是如何高效地处理这种情况的？答案是：它作弊。Rust的别名规则是一个问题，但C语言直接用指针完全忽略这个问题。作为交换，
常常能看见一些代码在手册中被声明在某些或所有情况下为非线程安全的。事实上，[GNU C library](https://manpages.debian.org/buster/manpages/attributes.7.en.html)
有专门研究并发行为的全部词典。

Rust总是使内存中的一切安全，能同时获得C语言中无法兼得的安全性和性能。被拒绝使用某些捷径是Rust的开发者必须付出的代价。

[^1]: 对于那些正在绞尽脑汁的C程序员来说，在这段代码中不需要读取迭代器，因为是未定义行为。排他性规则还支持编译器优化，这可能会导致由于迭代器的共享引用产生不一致的观察结果。（例如栈溢出或者重新排序指令以提高效率）。这些情况将可能在可变引用创建后的任何时间发生。
