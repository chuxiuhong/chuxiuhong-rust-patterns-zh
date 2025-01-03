# 函数式语言反射

反射系统(Optics)是函数式语言中常见的一种 API 设计。这是一个纯函数式的概念,在 Rust 中并不常用。

尽管如此,探索这个概念可能有助于理解 Rust API 中的其他模式,比如[访问者模式](../patterns/behavioural/visitor.md)。它们也有一些特定的使用场景。

这是一个相当大的主题,要完全理解它的功能需要阅读实际的语言设计书籍。不过它在 Rust 中的应用要简单得多。

为了解释这个概念的相关部分,我们将使用 `Serde` API 作为示例,因为仅从 API 文档来理解它对很多人来说都很困难。

在这个过程中,我们将介绍不同的特定模式,称为反射系统。这些包括 *The Iso*、*The Poly Iso* 和 *The Prism*。

## API 示例: Serde

仅通过阅读 API 来理解 *Serde* 的工作方式是一个挑战,特别是第一次接触时。考虑 `Deserializer` trait,它由任何解析新数据格式的库实现:


```rust,ignore
pub trait Deserializer<'de>: Sized {
    type Error: Error;

    fn deserialize_any<V>(self, visitor: V) -> Result<V::Value, Self::Error>
    where
        V: Visitor<'de>;

    fn deserialize_bool<V>(self, visitor: V) -> Result<V::Value, Self::Error>
    where
        V: Visitor<'de>;

    // remainder omitted
}
```

And here's the definition of the `Visitor` trait passed in generically:

```rust,ignore
pub trait Visitor<'de>: Sized {
    type Value;

    fn visit_bool<E>(self, v: bool) -> Result<Self::Value, E>
    where
        E: Error;

    fn visit_u64<E>(self, v: u64) -> Result<Self::Value, E>
    where
        E: Error;

    fn visit_str<E>(self, v: &str) -> Result<Self::Value, E>
    where
        E: Error;

}
```
这里涉及了大量的类型擦除，多层关联类型在其中来回传递。

但是从宏观来看是什么样的呢？为什么不直接让 `Visitor` 通过流式 API 返回调用者需要的部分就完事了呢？为什么需要这么多额外的部分？

理解这一点的一种方式是看看函数式语言中称为*反射系统*的概念。

这是一种行为和属性组合的方式，旨在促进 Rust 中常见的模式:失败处理、类型转换等。[^1]

Rust 语言本身并不直接支持这些。然而，它们出现在语言本身的设计中，而且这些概念可以帮助理解一些 Rust 的 API。因此，本文尝试用 Rust 的方式来解释这些概念。

这或许能揭示这些 API 实现的目标:特定的可组合性质。

## 基础反射系统

### The Iso

Iso 是两种类型之间的值转换器。它非常简单，但在概念上是一个重要的构建块。

举个例子，假设我们有一个用作文档索引的自定义哈希表结构。[^2] 它使用字符串作为键(单词)，使用索引列表作为值(例如文件偏移量)。

一个关键特性是能够将这种格式序列化到磁盘。一个"快速且简单"的方法是实现与 JSON 格式字符串之间的转换。(暂时忽略错误，稍后会处理。)

用函数式语言用户期望的标准形式来写:

```text
case class ConcordanceSerDe {
  serialize: Concordance -> String
  deserialize: String -> Concordance
}
```

因此 Iso 是一对用于转换不同类型值的函数:
`serialize` 和 `deserialize`。

一个直接的实现:

```rust
use std::collections::HashMap;

struct Concordance {
    keys: HashMap<String, usize>,
    value_table: Vec<(usize, usize)>,
}

struct ConcordanceSerde {}

impl ConcordanceSerde {
    fn serialize(value: Concordance) -> String {
        todo!()
    }
    // invalid concordances are empty
    fn deserialize(value: String) -> Concordance {
        todo!()
    }
}
```
这可能看起来有点傻。在 Rust 中，这种行为通常是通过 trait 来实现的。毕竟，标准库中已经有 `FromStr` 和 `ToString` 了。

但这就引出了我们的下一个主题：多态 Iso。

### 多态 Iso

前面的例子只是在两个固定类型之间进行转换。
下一个示例使用泛型构建在此基础之上，更加有趣。

多态 Iso 允许一个操作对任意类型进行泛型化，同时返回单一类型。

这让我们更接近解析。考虑一个基本的解析器在忽略错误情况下会做什么。再次以其标准形式表示：

```text
case class Serde[T] {
    deserialize(String) -> T
    serialize(T) -> String
}
```
这里我们有了第一个泛型，即被转换的类型 `T`。

在 Rust 中，这可以通过标准库中的一对 trait 来实现：`FromStr` 和 `ToString`。Rust 版本甚至还处理了错误：

```rust,ignore
pub trait FromStr: Sized {
    type Err;

    fn from_str(s: &str) -> Result<Self, Self::Err>;
}

pub trait ToString {
    fn to_string(&self) -> String;
}
```
与 Iso 不同，多态 Iso 允许应用多个类型，并以泛型方式返回它们。这正是基本字符串解析器所需要的。

乍看之下，这似乎是编写解析器的一个不错的选择。让我们看看它的实际应用:

```rust,ignore
use anyhow;

use std::str::FromStr;

struct TestStruct {
    a: usize,
    b: String,
}

impl FromStr for TestStruct {
    type Err = anyhow::Error;
    fn from_str(s: &str) -> Result<TestStruct, Self::Err> {
        todo!()
    }
}

impl ToString for TestStruct {
    fn to_string(&self) -> String {
        todo!()
    }
}

fn main() {
    let a = TestStruct {
        a: 5,
        b: "hello".to_string(),
    };
    println!("Our Test Struct as JSON: {}", a.to_string());
}
```
这看起来很合理。然而,这里有两个问题。

首先,`to_string` 并没有向 API 用户表明"这是 JSON"。每个类型都需要就 JSON 表示达成一致,而 Rust 标准库中的许多类型已经不这样做了。使用这种方式并不合适。这个问题可以通过我们自己的 trait 轻松解决。

但还有第二个更微妙的问题:可扩展性。

当每个类型都手写 `to_string` 时,这种方式是可行的。但如果每个想要其类型可序列化的人都必须编写大量代码 -- 并且可能使用不同的 JSON 库 -- 来自己实现这一点,这很快就会变成一团糟!

答案就是 Serde 的两个关键创新之一:一个独立的数据模型,用于将 Rust 数据表示为数据序列化语言中常见的结构。这样它就可以使用 Rust 的代码生成能力来创建一个称为 `Visitor` 的中间转换类型。

用正常形式表示(再次省略错误处理以简化):

```text
case class Serde[T] {
    deserialize: Visitor[T] -> T
    serialize: T -> Visitor[T]
}

case class Visitor[T] {
    toJson: Visitor[T] -> String
    fromJson: String -> Visitor[T]
}
```
结果是一个多态同构(Poly Iso)和一个同构(Iso)。这两者都可以通过 trait 来实现:

```rust
trait Serde {
    type V;
    fn deserialize(visitor: Self::V) -> Self;
    fn serialize(self) -> Self::V;
}

trait Visitor {
    fn to_json(self) -> String;
    fn from_json(json: String) -> Self;
}
```
由于有一套统一的规则来将 Rust 结构转换为独立形式,所以甚至可以通过代码生成来创建与类型 `T` 关联的 `Visitor`:

```rust,ignore
#[derive(Default, Serde)] // the "Serde" derive creates the trait impl block
struct TestStruct {
    a: usize,
    b: String,
}

// user writes this macro to generate an associated visitor type
generate_visitor!(TestStruct);
```

或者他们真的需要这样做吗?

```rust,ignore
fn main() {
    let a = TestStruct { a: 5, b: "hello".to_string() };
    let a_data = a.serialize().to_json();
    println!("Our Test Struct as JSON: {a_data}");
    let b = TestStruct::deserialize(
        generated_visitor_for!(TestStruct)::from_json(a_data));
}
```
事实证明,这种转换并不是对称的!在理论上是对称的,但在自动生成的代码中,从 `String` 完全转换所需的实际类型名称是隐藏的。我们需要某种 `generated_visitor_for!` 宏来获取类型名称。

这种方式虽然不太优雅,但确实可以工作...直到我们遇到一个大问题。

目前只支持 JSON 格式。我们如何支持更多格式呢?

当前的设计需要完全重写所有代码生成并创建一个新的 Serde trait。这非常糟糕且完全不可扩展!

为了解决这个问题,我们需要一些更强大的东西。

## 棱镜(Prism)

为了考虑格式,我们需要一个像这样的标准形式:

```text
case class Serde[T, F] {
    serialize: T, F -> String
    deserialize: String, F -> Result[T, Error]
}
```
这种结构被称为棱镜(Prism)。它在泛型层次上比多态同构(Poly Isos)高一级(在这种情况下,"交叉"类型 F 是关键)。

不幸的是,由于 `Visitor` 是一个 trait(因为每个实例都需要自己的自定义代码),这需要一种 Rust 不支持的泛型类型边界。

幸运的是,我们仍然有之前的 `Visitor` 类型。`Visitor` 在做什么?它试图让每个数据结构定义自己被解析的方式。

那如果我们能为泛型格式添加一个接口呢?这样 `Visitor` 就只是一个实现细节,它将"桥接"这两个 API。

用标准形式表示:

```text
case class Serde[T] {
    serialize: F -> String
    deserialize F, String -> Result[T, Error]
}

case class VisitorForT {
    build: F, String -> Result[T, Error]
    decompose: F, T -> String
}

case class SerdeFormat[T, V] {
    toString: T, V -> String
    fromString: V, String -> Result[T, Error]
}
```
看看,底部有一对可以实现为 trait 的多态同构(Poly Isos)!

因此我们得到了 Serde API:

1. 每个要序列化的类型都实现 `Deserialize` 或 `Serialize`,
   相当于 `Serde` 类
1. 它们获得一个(实际上是两个,每个方向一个)实现 `Visitor`
   trait 的类型,这通常(但不总是)通过派生宏生成的代码完成。
   这包含了在数据类型和 Serde 数据模型格式之间构造或解构的逻辑。
1. 实现 `Deserializer` trait 的类型处理特定于格式的所有细节,
   由 `Visitor` "驱动"。

这种分割和 Rust 类型擦除实际上是通过间接方式实现棱镜(Prism)。

你可以在 `Deserializer` trait 上看到这一点

```rust,ignore
pub trait Deserializer<'de>: Sized {
    type Error: Error;

    fn deserialize_any<V>(self, visitor: V) -> Result<V::Value, Self::Error>
    where
        V: Visitor<'de>;

    fn deserialize_bool<V>(self, visitor: V) -> Result<V::Value, Self::Error>
    where
        V: Visitor<'de>;

    // remainder omitted
}
```

以及访问者:

```rust,ignore
pub trait Visitor<'de>: Sized {
    type Value;

    fn visit_bool<E>(self, v: bool) -> Result<Self::Value, E>
    where
        E: Error;

    fn visit_u64<E>(self, v: u64) -> Result<Self::Value, E>
    where
        E: Error;

    fn visit_str<E>(self, v: &str) -> Result<Self::Value, E>
    where
        E: Error;

    // remainder omitted
}
```

以及由宏实现的 `Deserialize` trait:

```rust,ignore
pub trait Deserialize<'de>: Sized {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: Deserializer<'de>;
}
```
以上内容比较抽象,让我们来看一个具体的例子。

实际的 Serde 是如何将一段 JSON 反序列化为前面的 `struct Concordance` 的呢?

1. 用户会调用一个库函数来反序列化数据。这会基于 JSON 格式创建一个 `Deserializer`。
1. 基于结构体中的字段,会创建一个 `Visitor`(稍后会详细介绍),它知道如何创建表示所需数据的通用数据模型中的每种类型:`Vec`(列表)、`u64` 和 `String`。
1. 反序列化器会在解析项目时调用 `Visitor`。
1. `Visitor` 会指示找到的项目是否符合预期,如果不符合,则会引发错误表明反序列化失败。

对于我们上面的简单结构,预期的模式是:

1. 开始访问一个映射(*Serde* 中等同于 `HashMap` 或 JSON 的字典)。
1. 访问一个名为 "keys" 的字符串键。
1. 开始访问一个映射值。
1. 对于每个项目,访问一个字符串键然后是一个整数值。
1. 访问映射的结尾。
1. 将映射存储到数据结构的 `keys` 字段中。
1. 访问一个名为 "value_table" 的字符串键。
1. 开始访问一个列表值。
1. 对于每个项目,访问一个整数。
1. 访问列表的结尾。
1. 将列表存储到 `value_table` 字段中。
1. 访问映射的结尾。

但是什么决定了预期的"观察"模式呢?

函数式编程语言能够使用柯里化来基于类型本身创建每个类型的反射。Rust 不支持这一点,所以每个类型都需要根据其字段及其属性编写自己的代码。

*Serde* 通过派生宏解决了这个可用性挑战:

```rust,ignore
use serde::Deserialize;

#[derive(Deserialize)]
struct IdRecord {
    name: String,
    customer_id: String,
}
```
该宏简单地生成一个 impl 块,使结构体实现名为 `Deserialize` 的 trait。

这个函数决定了如何创建结构体本身。代码是基于结构体的字段生成的。当解析库被调用时 - 在我们的例子中是 JSON 解析库 - 它会创建一个 `Deserializer` 并以其作为参数调用 `Type::deserialize`。

`deserialize` 代码随后会创建一个 `Visitor`,它的调用会被 `Deserializer` "折射"。如果一切顺利,最终该 `Visitor` 会构造一个与正在解析的类型相对应的值并返回它。

完整示例请参见 [*Serde* 文档](https://serde.rs/deserialize-struct.html)。

最终结果是,需要反序列化的类型只需要实现 API 的"顶层",而文件格式只需要实现"底层"。由于泛型类型会桥接它们,每个部分都可以与生态系统的其余部分"无缝工作"。

总之,Rust 的泛型启发的类型系统可以让它接近这些概念并利用它们的力量,正如这个 API 设计所示。但它可能还需要过程宏来为其泛型创建桥梁。

如果您有兴趣了解更多关于这个主题的内容,请查看以下部分。

## 另请参阅

- [lens-rs crate](https://crates.io/crates/lens-rs) - 一个预构建的镜头实现,接口比这些示例更清晰
- [Serde](https://serde.rs) 本身,它使这些概念对最终用户(即定义结构体)来说很直观,无需理解细节
- [luminance](https://github.com/phaazon/luminance-rs) 是一个用于绘制计算机图形的 crate,使用类似的 API 设计,包括过程宏来为不同像素类型的缓冲区创建完整的棱镜,同时保持泛型
- [关于 Scala 中镜头的文章](https://web.archive.org/web/20221128185849/https://medium.com/zyseme-technology/functional-references-lens-and-other-optics-in-scala-e5f7e2fdafe) - 即使不懂 Scala 也很容易理解
- [论文: Profunctor Optics: 模块化数据访问器](https://web.archive.org/web/20220701102832/https://arxiv.org/ftp/arxiv/papers/1703/1703.10857.pdf)
- [Musli](https://github.com/udoprog/musli) 是一个尝试使用不同方法实现类似结构的库,例如摒弃访问者模式

[^1]: [Haskell 学校: 镜头入门教程](https://web.archive.org/web/20221128190041/https://www.schoolofhaskell.com/school/to-infinity-and-beyond/pick-of-the-week/a-little-lens-starter-tutorial)

[^2]: [Wikipedia 上的 Concordance](https://en.wikipedia.org/wiki/Concordance_(publishing))
