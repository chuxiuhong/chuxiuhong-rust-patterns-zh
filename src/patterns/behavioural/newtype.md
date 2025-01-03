# Newtype 模式

如果在某些情况下，我们希望一个类型的行为类似于另一个类型，或者在编译时强制执行某些行为，而仅使用类型别名（type aliases）是不够的，该怎么办？

例如，出于安全考虑（如处理密码），我们可能想要为 `String` 创建一个自定义的 `Display` 实现。

对于这种情况，我们可以使用 `Newtype` 模式来提供**类型安全**和**封装**。

## 描述

使用只有一个字段的元组结构体来为类型创建一个不透明的包装器。这会创建一个新类型，而不是类型的别名（`type` 项）。

## 示例

```rust
use std::fmt::Display;

// 创建 Newtype Password 来重写 String 的 Display trait
struct Password(String);

impl Display for Password {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "****************")
    }
}

fn main() {
    let unsecured_password: String = "ThisIsMyPassword".to_string();
    let secured_password: Password = Password(unsecured_password.clone());
    println!("unsecured_password: {unsecured_password}");
    println!("secured_password: {secured_password}");
}
```

```shell
unsecured_password: ThisIsMyPassword
secured_password: ****************
```

## 动机

Newtype 的主要动机是抽象。它允许你在类型之间共享实现细节，同时精确控制接口。通过使用 newtype 而不是在 API 中直接暴露实现类型，你可以向后兼容地更改实现。

Newtype 可以用于区分单位，例如，包装 `f64` 以区分 `Miles` 和 `Kilometres`。

## 优点

- 被包装类型和包装器类型在类型系统中不兼容（与使用 `type` 相反），因此 newtype 的用户永远不会"混淆"这两种类型
- Newtype 是零成本抽象 - 没有运行时开销
- 私有性系统确保用户无法访问被包装的类型（如果字段是私有的，这是默认设置）

## 缺点

Newtype 的缺点（特别是与类型别名相比）是没有特殊的语言支持。这意味着可能会有*大量*的样板代码。你需要为被包装类型上每个想要暴露的方法编写一个"传递"方法，并为每个想要在包装器类型上实现的 trait 编写实现。

## 讨论

Newtype 在 Rust 代码中非常常见。抽象或表示单位是最常见的用途，但它们还可以用于其他目的：

- 限制功能（减少暴露的函数或实现的 trait）
- 使具有复制语义的类型具有移动语义
- 通过提供更具体的类型来实现抽象，从而隐藏内部类型，例如：

```rust,ignore
pub struct Foo(Bar<T1, T2>);
```

在这里，`Bar` 可能是某个公共的泛型类型，而 `T1` 和 `T2` 是一些内部类型。我们模块的用户不需要知道我们使用 `Bar` 来实现 `Foo`，但我们真正隐藏的是类型 `T1` 和 `T2`，以及它们如何与 `Bar` 一起使用。

## 参见

- [Advanced Types in the book](https://doc.rust-lang.org/book/ch19-04-advanced-types.html?highlight=newtype#using-the-newtype-pattern-for-type-safety-and-abstraction)
- [Newtypes in Haskell](https://wiki.haskell.org/Newtype)
- [Type aliases](https://doc.rust-lang.org/stable/book/ch19-04-advanced-types.html#creating-type-synonyms-with-type-aliases)
- [derive_more](https://crates.io/crates/derive_more)，一个用于在 newtype 上派生许多内置 trait 的 crate
- [The Newtype Pattern In Rust](https://web.archive.org/web/20230519162111/https://www.worthe-it.co.za/blog/2020-10-31-newtype-pattern-in-rust.html)
