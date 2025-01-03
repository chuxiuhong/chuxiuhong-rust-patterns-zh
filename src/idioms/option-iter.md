# 遍历 `Option`

## 描述

`Option` 可以被视为一个包含零个或一个元素的容器。
特别地，它实现了 `IntoIterator` trait，因此可以用于需要这种类型的泛型代码中。

## 示例

由于 `Option` 实现了 `IntoIterator`，它可以作为 [`.extend()`](https://doc.rust-lang.org/std/iter/trait.Extend.html#tymethod.extend) 的参数：

```rust
let turing = Some("Turing");
let mut logicians = vec!["Curry", "Kleene", "Markov"];

logicians.extend(turing);

// 等价于
if let Some(turing_inner) = turing {
    logicians.push(turing_inner);
}
```

如果你需要将一个 `Option` 添加到现有迭代器的末尾，你可以使用 [`.chain()`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.chain)：

```rust
let turing = Some("Turing");
let logicians = vec!["Curry", "Kleene", "Markov"];

for logician in logicians.iter().chain(turing.iter()) {
    println!("{logician} is a logician");
}
```

注意，如果 `Option` 总是 `Some`，那么使用 [`std::iter::once`](https://doc.rust-lang.org/std/iter/fn.once.html) 来处理元素会更符合惯用法。

另外，由于 `Option` 实现了 `IntoIterator`，可以使用 `for` 循环来遍历它。这等价于使用 `if let Some(..)` 进行匹配，在大多数情况下，后者是更推荐的写法。

## 参见

- [`std::iter::once`](https://doc.rust-lang.org/std/iter/fn.once.html) 是一个只生成一个元素的迭代器。相比 `Some(foo).into_iter()`，它的可读性更好。

- [`Iterator::filter_map`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.filter_map) 是 [`Iterator::map`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.map) 的特化版本，专门用于返回 `Option` 的映射函数。

- [`ref_slice`](https://crates.io/crates/ref_slice) crate 提供了将 `Option` 转换为零个或一个元素的切片的函数。

- [`Option<T>` 的文档](https://doc.rust-lang.org/std/option/enum.Option.html)
