# `Default` 特性

## 说明

许多Rust中的类型有一个构造器。然而，构造器是针对特定类型的。Rust不能抽象出一个代表所有带有`new()`方法的东西。为了实现这个想法，
一个可被容器和其他泛型使用的`Default`特性应运而生（如 [`Option::unwrap_or_default()`）。尤其是一些容器已经在适当的情况下实现了它。

单例容器如 `Cow`, `Box` 和 `Arc`为`Default`类型实现了`Default`，
并且可以自动地对每个成员都实现`Default`的结构体支持`#[derive(Default)]`。所以越多的类型支持 `Default`，它就会越有用。

另一方面，构造器能够接受多个参数，而`default()`方法不能。你甚至可以定义多个不同的函数做多个构造器，但是你最多只能为一个类型实现一种`Default`的实现。

## 例子

```rust
use std::{path::PathBuf, time::Duration};

// 注意我们可以用自动导出 Default.
#[derive(Default, Debug)]
struct MyConfiguration {
    // Option defaults to None
    output: Option<PathBuf>,
    // Vecs default to empty vector
    search_path: Vec<PathBuf>,
    // Duration defaults to zero time
    timeout: Duration,
    // bool defaults to false
    check: bool,
}

impl MyConfiguration {
    // add setters here
}

fn main() {
    // construct a new instance with default values
    let mut conf = MyConfiguration::default();
    // do something with conf here
    conf.check = true;
    println!("conf = {:#?}", conf);
}
```

## 参阅

- The [constructor] idiom is another way to generate instances that may or may
not be "default"
- The [`Default`] documentation (scroll down for the list of implementors)
- [`Option::unwrap_or_default()`]
- [`derive(new)`]

[constructor]: ctor.md
[`Default`]: https://doc.rust-lang.org/stable/std/default/trait.Default.html
[`Option::unwrap_or_default()`]: https://doc.rust-lang.org/stable/std/option/enum.Option.html#method.unwrap_or_default
[`derive(new)`]: https://crates.io/crates/derive-new/
