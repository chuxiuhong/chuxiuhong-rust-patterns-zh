# `Default` 特征

## 描述

Rust 中的许多类型都有[构造器（constructor）][constructor]。然而，这是*特定于*类型的；Rust 无法对"所有具有 `new()` 方法的类型"进行抽象。为了实现这一点，[`Default`] 特征被引入，它可以与容器和其他泛型类型一起使用（例如 [`Option::unwrap_or_default()`]）。值得注意的是，一些容器在适用的情况下已经实现了它。

不仅 `Cow`、`Box` 或 `Arc` 这样的单元素容器为其包含的 `Default` 类型实现了 `Default`，对于所有字段都实现了 `Default` 的结构体，我们还可以自动使用 `#[derive(Default)]` 来派生实现，因此越多类型实现 `Default`，它就变得越有用。

另一方面，构造器可以接受多个参数，而 `default()` 方法不能。甚至可以有多个具有不同名称的构造器，但每个类型只能有一个 `Default` 实现。

## 示例

```rust
use std::{path::PathBuf, time::Duration};

// 注意这里我们可以简单地自动派生 Default
#[derive(Default, Debug, PartialEq)]
struct MyConfiguration {
    // Option 默认值为 None
    output: Option<PathBuf>,
    // Vec 默认值为空向量
    search_path: Vec<PathBuf>,
    // Duration 默认值为零时长
    timeout: Duration,
    // bool 默认值为 false
    check: bool,
}

impl MyConfiguration {
    // 在此处添加 setter 方法
}

fn main() {
    // 使用默认值构造新实例
    let mut conf = MyConfiguration::default();
    // 在这里对 conf 进行一些操作
    conf.check = true;
    println!("conf = {conf:#?}");

    // 使用默认值进行部分初始化，创建相同的实例
    let conf1 = MyConfiguration {
        check: true,
        ..Default::default()
    };
    assert_eq!(conf, conf1);
}
```

## 参见

- [构造器（constructor）][constructor] 惯用法是另一种生成实例的方式，这些实例可能是也可能不是"默认的"
- [`Default`] 文档（向下滚动查看实现者列表）
- [`Option::unwrap_or_default()`]
- [`derive(new)`]

[constructor]: ctor.md
[`Default`]: https://doc.rust-lang.org/stable/std/default/trait.Default.html
[`Option::unwrap_or_default()`]: https://doc.rust-lang.org/stable/std/option/enum.Option.html#method.unwrap_or_default
[`derive(new)`]: https://crates.io/crates/derive-new/
