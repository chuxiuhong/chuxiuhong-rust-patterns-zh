# 策略模式

## 说明

[策略模式](https://en.wikipedia.org/wiki/Strategy_pattern)是支持关注点分离的一门技术。 它还支持通过 [依赖倒置](https://en.wikipedia.org/wiki/Dependency_inversion_principle)来分离软件模块。

策略模式背后的基本思想是，给定一个解决特定问题的算法，我们仅在抽象层次上定义算法的框架，并将指定的算法实现分成不同的部分。

这样，使用该算法的客户端可以选择特定的实现，而通用的算法工作流可以保持不变。换句话说，类的抽象规范不依赖于派生类的具体实现，而是具体实现必须遵循抽象规范。这就是我们为什么叫它“依赖倒置”。

## 出发点

想象一下我们正在开发一个需要每个月生成报告的项目。我们需要用不同格式生成报告（不同策略）例如用`JSON`或者`富文本`。但是事物是在发展的，我们也不知道未来有什么需求。例如，我们也许需要用一种全新的格式生成报告，或者是修改我们已有的一种格式。

## 代码示例

在这个例子中我们的不变量（或者说抽象）是`Context`,`Formatter`和`Report`，同时`Text`和`Json`是我们的策略结构体。这些策略都要实现`Formatter`特性。

```rust
use std::collections::HashMap;
type Data = HashMap<String, u32>;

trait Formatter {
    fn format(&self, data: &Data, s: &mut String);
}

struct Report;

impl Report {
    fn generate<T: Formatter>(g: T, s: &mut String) {
        // backend operations...
        let mut data = HashMap::new();
        data.insert("one".to_string(), 1);
        data.insert("two".to_string(), 2);
        // generate report
        g.format(&data, s);
    }
}

struct Text;
impl Formatter for Text {
    fn format(&self, data: &Data, s: &mut String) {
        *s = data
            .iter()
            .map(|(key, val)| format!("{} {}\n", key, val))
            .collect();
    }
}

struct Json;
impl Formatter for Json {
    fn format(&self, data: &Data, s: &mut String) {
        *s = String::from("[");
        let mut iter = data.into_iter();
        if let Some((key, val)) = iter.next() {
            let entry = format!(r#"{{"{}":"{}"}}"#, key, val);
            s.push_str(&entry);
            while let Some((key, val)) = iter.next() {
                s.push(',');
                let entry = format!(r#"{{"{}":"{}"}}"#, key, val);
                s.push_str(&entry);
            }
        }
        s.push(']');
    }
}

fn main() {
    let mut s = String::from("");
    Report::generate(Text, &mut s);
    assert!(s.contains("one 1"));
    assert!(s.contains("two 2"));

    Report::generate(Json, &mut s);
    assert!(s.contains(r#"{"one":"1"}"#));
    assert!(s.contains(r#"{"two":"2"}"#));
}
```

## 优点

主要的优点是分离关注点。举例来说，在这个例子里`Report`并不知道`Json`和`Text`的特定实现，尽管输出的实现并不关心数据是如何被预处理、存储和抓取的。它仅仅需要知道上下文和需要实现的特定的特性和方法，就像`Formatter`和`run`。

## 缺点

对于每个策略，必须至少实现一个模块，因此模块的数量会随着策略数量增加。如果有很多策略可供选择，那么用户就必须知道策略之间的区别。

## 讨论

在前面的例子中所有的策略实现都在一个文件中。提供不同策略的方式包括：

- 所有都在一个文件中（如本例所示，类似于被分离为模块）
- 分离成模块，例如`formatter::json`模块、`formatter::text`模块
- 使用编译器特性标志，例如`json`特性、`text`特性
- 分离成不同的库，例如`json`库、`text`库

Serde库是策略模式的一个实践的好例子。Serde通过手动实现`Serialize`和`Deserialize`特性支持[完全定制](https://serde.rs/custom-serialization.html)化序列化的行为。例如，我们可以轻松替换`serde_json`为`serde_cbor`因为它们暴露相似的方法。有了它，库`serde_transcode`更有用和符合人体工程学。

不过，我们在Rust中不需要特性来实现这个模式。

下面这个玩具例子演示了用Rust的`闭包`来实现策略模式的思路：

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

事实上，Rust已经将这个思路用于`Option`的`map`方法：

```rust
fn main() {
    let val = Some("Rust");

    let len_strategy = |s: &str| s.len();
    assert_eq!(4, val.map(len_strategy).unwrap());

    let first_byte_strategy = |s: &str| s.bytes().next().unwrap();
    assert_eq!(82, val.map(first_byte_strategy).unwrap());
}
```

## See also

- [策略模式](https://en.wikipedia.org/wiki/Strategy_pattern)
- [依赖注入](https://en.wikipedia.org/wiki/Dependency_injection)
- [基于策略的设计](https://en.wikipedia.org/wiki/Modern_C++_Design#Policy-based_design)
