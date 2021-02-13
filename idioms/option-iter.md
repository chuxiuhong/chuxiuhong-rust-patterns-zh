# 关于 `Option`的迭代器

## 说明

`Option`可以被视为一个包含一个0个或者1个元素的容器。特别是它实现了`IntoIterator`特性，这样我们就可以用来写泛型代码。

## 示例

因为`Option`实现了`IntoIterator`特性，它就可以用来当[`.extend()`](https://doc.rust-lang.org/std/iter/trait.Extend.html#tymethod.extend)的参数:

```rust
let turing = Some("Turing");
let mut logicians = vec!["Curry", "Kleene", "Markov"];

logicians.extend(turing);

// equivalent to
if let Some(turing_inner) = turing {
    logicians.push(turing_inner);
}
```

如果你需要将一个`Option`添加到已有的迭代器后面，你可以用 [`.chain()`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.chain):

```rust
let turing = Some("Turing");
let logicians = vec!["Curry", "Kleene", "Markov"];

for logician in logicians.iter().chain(turing.iter()) {
    println!("{} is a logician", logician);
}
```

注意如果这个`Option`总是非空的，那么用[`std::iter::once`](https://doc.rust-lang.org/std/iter/fn.once.html)更加合适。

此外，因为`Option`实现了`IntoIterator`特性，它就可以用`for`循环来迭代。这等价于用`if let Some(..)`，大多数情况下倾向于用后者。

## 参阅

* [`std::iter::once`](https://doc.rust-lang.org/std/iter/fn.once.html) 是一个只产生一个元素的迭代器。这有一个更具可读性的替代品`Some(foo).into_iter()`。

* [`Iterator::filter_map`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.filter_map)
是 [`Iterator::flat_map`](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.flat_map)专注于处理返回值是`Option`的map函数版本。

* [`ref_slice`](https://crates.io/crates/ref_slice) 包提供将`Option`转换为0个或1个元素的切片的函数。

* [`Option<T>`的文档](https://doc.rust-lang.org/std/option/enum.Option.html)
