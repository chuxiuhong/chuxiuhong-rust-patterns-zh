# Strategy (策略模式，又称 Policy)

## 描述

[Strategy 设计模式](https://en.wikipedia.org/wiki/Strategy_pattern)是一种实现关注点分离的技术。它还允许通过[依赖倒置](https://en.wikipedia.org/wiki/Dependency_inversion_principle)来解耦软件模块。

Strategy 模式的基本思想是，对于解决特定问题的算法，我们只在抽象层面定义算法的框架，并将具体算法的实现分离到不同的部分。

通过这种方式，使用该算法的客户端可以选择特定的实现，而通用的算法工作流程保持不变。换句话说，类的抽象规范不依赖于派生类的具体实现，但具体实现必须遵守抽象规范。这就是为什么我们称之为"依赖倒置"。

## 动机

假设我们正在开发一个每月生成报告的项目。我们需要以不同的格式（策略）生成报告，例如 `JSON` 或 `Plain Text` 格式。但是需求会随时间变化，我们无法预知未来可能出现的需求。例如，我们可能需要以全新的格式生成报告，或者仅修改现有格式之一。

## 示例

在这个示例中，我们的不变量（或抽象）是 `Formatter` 和 `Report`，而 `Text` 和 `Json` 是我们的策略结构体。这些策略必须实现 `Formatter` trait。

```rust
use std::collections::HashMap;

type Data = HashMap<String, u32>;

trait Formatter {
    fn format(&self, data: &Data, buf: &mut String);
}

struct Report;

impl Report {
    // 应该使用 Write，但我们保持使用 String 以忽略错误处理
    fn generate<T: Formatter>(g: T, s: &mut String) {
        // 后端操作...
        let mut data = HashMap::new();
        data.insert("one".to_string(), 1);
        data.insert("two".to_string(), 2);
        // 生成报告
        g.format(&data, s);
    }
}

struct Text;
impl Formatter for Text {
    fn format(&self, data: &Data, buf: &mut String) {
        for (k, v) in data {
            let entry = format!("{k} {v}\n");
            buf.push_str(&entry);
        }
    }
}

struct Json;
impl Formatter for Json {
    fn format(&self, data: &Data, buf: &mut String) {
        buf.push('[');
        for (k, v) in data.into_iter() {
            let entry = format!(r#"{{"{}":"{}"}}"#, k, v);
            buf.push_str(&entry);
            buf.push(',');
        }
        if !data.is_empty() {
            buf.pop(); // 移除末尾多余的逗号
        }
        buf.push(']');
    }
}

fn main() {
    let mut s = String::from("");
    Report::generate(Text, &mut s);
    assert!(s.contains("one 1"));
    assert!(s.contains("two 2"));

    s.clear(); // 重用相同的缓冲区
    Report::generate(Json, &mut s);
    assert!(s.contains(r#"{"one":"1"}"#));
    assert!(s.contains(r#"{"two":"2"}"#));
}
```

## 优点

主要优点是关注点分离。例如，在这个案例中，`Report` 不需要了解 `Json` 和 `Text` 的具体实现，而输出实现也不关心数据是如何预处理、存储和获取的。它们只需要知道要实现的特定 trait 和定义具体算法实现来处理结果的方法，即 `Formatter` 和 `format(...)`。

## 缺点

每个策略都必须实现至少一个模块，因此模块数量会随着策略数量的增加而增加。如果有许多策略可供选择，用户必须了解这些策略之间的区别。

## 讨论

在前面的示例中，所有策略都在单个文件中实现。提供不同策略的方式包括：

- 全部在一个文件中（如本示例所示，类似于作为模块分离）
- 作为模块分离，例如 `formatter::json` 模块，`formatter::text` 模块
- 使用编译器特性标志，例如 `json` 特性，`text` 特性
- 作为 crate 分离，例如 `json` crate，`text` crate

Serde crate 是 `Strategy` 模式实践的一个很好的例子。Serde 通过手动为我们的类型实现 `Serialize` 和 `Deserialize` trait，允许[完全自定义](https://serde.rs/custom-serialization.html)序列化行为。例如，我们可以轻松地将 `serde_json` 替换为 `serde_cbor`，因为它们暴露了类似的方法。这使得辅助 crate `serde_transcode` 变得更加有用和人性化。

然而，在 Rust 中设计这种模式并不一定需要使用 trait。

下面这个简单的示例演示了使用 Rust `闭包`实现 Strategy 模式的思想：

```rust
struct Adder;
impl Adder {
    pub fn add<F>(x: u8, y: u8, f: F) -> u8
    where
        F: Fn(u8, u8) -> u8,
    {
        f(x, y)
    }
}

fn main() {
    let arith_adder = |x, y| x + y;
    let bool_adder = |x, y| {
        if x == 1 || y == 1 {
            1
        } else {
            0
        }
    };
    let custom_adder = |x, y| 2 * x + y;

    assert_eq!(9, Adder::add(4, 5, arith_adder));
    assert_eq!(0, Adder::add(0, 0, bool_adder));
    assert_eq!(5, Adder::add(1, 3, custom_adder));
}
```

实际上，Rust 已经在 `Option` 的 `map` 方法中使用了这个思想：

```rust
fn main() {
    let val = Some("Rust");

    let len_strategy = |s: &str| s.len();
    assert_eq!(4, val.map(len_strategy).unwrap());

    let first_byte_strategy = |s: &str| s.bytes().next().unwrap();
    assert_eq!(82, val.map(first_byte_strategy).unwrap());
}
```

## 参见

- [Strategy Pattern](https://en.wikipedia.org/wiki/Strategy_pattern)
- [Dependency Injection](https://en.wikipedia.org/wiki/Dependency_injection)
- [Policy Based Design](https://en.wikipedia.org/wiki/Modern_C++_Design#Policy-based_design)
- [使用 Strategy 模式在 Rust 中实现航天应用的 TCP 服务器](https://web.archive.org/web/20231003171500/https://robamu.github.io/posts/rust-strategy-pattern/)
