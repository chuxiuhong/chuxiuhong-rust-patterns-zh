# `#[non_exhaustive]` 和私有字段的可扩展性设计

## 描述

在某些特定场景下，库的作者可能希望在不破坏向后兼容性的前提下，为公共结构体添加公共字段或为枚举添加新的变体。

Rust 提供了两种解决方案：

- 在 `struct`、`enum` 和 `enum` 变体上使用 `#[non_exhaustive]`。关于 `#[non_exhaustive]` 的所有使用场景的详细文档，
  请参见[官方文档](https://doc.rust-lang.org/reference/attributes/type_system.html#the-non_exhaustive-attribute)。

- 你可以在结构体中添加一个私有字段，以防止它被直接实例化或被模式匹配（参见替代方案）

## 示例

```rust
mod a {
    // Public struct.
    #[non_exhaustive]
    pub struct S {
        pub foo: i32,
    }

    #[non_exhaustive]
    pub enum AdmitMoreVariants {
        VariantA,
        VariantB,
        #[non_exhaustive]
        VariantC {
            a: String,
        },
    }
}

fn print_matched_variants(s: a::S) {
    // 因为 S 是 `#[non_exhaustive]`，这里不能直接命名它
    // 我们必须在模式中使用 `..`
    let a::S { foo: _, .. } = s;

    let some_enum = a::AdmitMoreVariants::VariantA;
    match some_enum {
        a::AdmitMoreVariants::VariantA => println!("it's an A"),
        a::AdmitMoreVariants::VariantB => println!("it's a b"),

        // 这里需要 .. 是因为这个变体也是 non-exhaustive
        a::AdmitMoreVariants::VariantC { a, .. } => println!("it's a c"),

        // 通配符匹配是必需的，因为将来可能会添加新的变体
        _ => println!("it's a new variant"),
    }
}
```

## 替代方案：结构体的`私有字段`

`#[non_exhaustive]` 只在 crate 边界之间生效。在同一个 crate 内，可以使用私有字段方法。

向结构体添加字段通常是向后兼容的更改。但是，如果客户端使用模式来解构结构体实例，他们可能会列举结构体中的所有字段，这样添加新字段就会破坏该模式。客户端可以只命名部分字段并在模式中使用 `..`，这种情况下添加新字段是向后兼容的。通过使结构体的至少一个字段为私有，可以强制客户端使用后一种模式形式，从而确保结构体具有未来可扩展性。

这种方法的缺点是你可能需要向结构体添加一个原本不需要的字段。你可以使用 `()` 类型，这样就不会有运行时开销，并在字段名前加上 `_` 以避免未使用字段的警告。

```rust
pub struct S {
    pub a: i32,
    // 因为 `b` 是私有的，你不能在不使用 `..` 的情况下匹配 `S`，
    // 并且 `S` 不能被直接实例化或匹配
    _b: (),
}
```

## 讨论

对于 `struct`，`#[non_exhaustive]` 允许以向后兼容的方式添加额外的字段。它还会阻止客户端使用结构体构造函数，即使所有字段都是公共的。这可能会有帮助，但值得考虑的是，你是否*希望*客户端在添加额外字段时通过编译器错误发现，而不是可能被静默忽略。

`#[non_exhaustive]` 也可以应用于枚举变体。一个 `#[non_exhaustive]` 变体的行为与 `#[non_exhaustive]` 结构体相同。

请谨慎和有意识地使用这个特性：在添加字段或变体时增加主版本号通常是更好的选择。当你在对外部资源建模，且该资源可能与你的库不同步变化时，`#[non_exhaustive]` 可能是合适的，但它不是一个通用工具。

### 缺点

`#[non_exhaustive]` 可能会使你的代码使用起来不那么符合人体工程学，特别是在被迫处理未知枚举变体时。它应该只在需要在**不**增加主版本号的情况下进行这类演进时使用。

当 `#[non_exhaustive]` 应用于 `enum` 时，它会强制客户端处理通配符变体。如果在这种情况下没有合理的处理方式，这可能会导致代码尴尬，并且这些代码路径只在极少数情况下执行。如果客户端决定在这种情况下 `panic!()`，那么在编译时暴露这个错误可能会更好。实际上，`#[non_exhaustive]` 强制客户端处理"其他情况"；在这种情况下很少有合理的处理方式。

## 参见

- [引入 #[non_exhaustive] 属性用于枚举和结构体的 RFC](https://github.com/rust-lang/rfcs/blob/master/text/2008-non-exhaustive.md)
