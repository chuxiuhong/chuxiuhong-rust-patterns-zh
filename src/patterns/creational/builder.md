# Builder 模式

## 描述

通过调用 builder 辅助对象来构造一个对象。

## 示例

```rust
#[derive(Debug, PartialEq)]
pub struct Foo {
    // Lots of complicated fields.
    bar: String,
}

impl Foo {
    // This method will help users to discover the builder
    pub fn builder() -> FooBuilder {
        FooBuilder::default()
    }
}

#[derive(Default)]
pub struct FooBuilder {
    // Probably lots of optional fields.
    bar: String,
}

impl FooBuilder {
    pub fn new(/* ... */) -> FooBuilder {
        // Set the minimally required fields of Foo.
        FooBuilder {
            bar: String::from("X"),
        }
    }

    pub fn name(mut self, bar: String) -> FooBuilder {
        // Set the name on the builder itself, and return the builder by value.
        self.bar = bar;
        self
    }

    // If we can get away with not consuming the Builder here, that is an
    // advantage. It means we can use the FooBuilder as a template for constructing
    // many Foos.
    pub fn build(self) -> Foo {
        // Create a Foo from the FooBuilder, applying all settings in FooBuilder
        // to Foo.
        Foo { bar: self.bar }
    }
}

#[test]
fn builder_test() {
    let foo = Foo {
        bar: String::from("Y"),
    };
    let foo_from_builder: Foo = FooBuilder::new().name(String::from("Y")).build();
    assert_eq!(foo, foo_from_builder);
}
```

## 动机

当你需要多个构造函数或构造过程中有副作用时，这个模式特别有用。

## 优点

- 将构建方法与其他方法分离
- 避免构造函数的泛滥
- 可用于简单的一行初始化，也适用于更复杂的构造场景

## 缺点

相比直接创建结构体对象或使用简单的构造函数，这种模式更为复杂。

## 讨论

由于 Rust 不支持函数重载，这种模式在 Rust 中（即使是对于较简单的对象）比其他语言中更常见。因为在 Rust 中同名方法只能有一个，所以与 C++、Java 等语言相比，在 Rust 中使用多个构造函数并不那么优雅。

这种模式经常用于 builder 对象本身就有用处的场景，而不仅仅是作为一个构建器。例如，[`std::process::Command`](https://doc.rust-lang.org/std/process/struct.Command.html) 是 [`Child`](https://doc.rust-lang.org/std/process/struct.Child.html)（一个进程）的 builder。在这些情况下，不会使用 `T` 和 `TBuilder` 的命名模式。

示例中采用了值传递和返回 builder。通常来说，使用可变引用来传递和返回 builder 会更符合人体工程学（也更高效）。借用检查器使这种方式自然可行。这种方法的优势在于既可以写成：

```rust,ignore
let mut fb = FooBuilder::new();
fb.a();
fb.b();
let f = fb.build();
```

也可以写成 `FooBuilder::new().a().b().build()` 的链式调用风格。

## 参见

- [Rust 风格指南中的描述](https://web.archive.org/web/20210104103100/https://doc.rust-lang.org/1.12.0/style/ownership/builders.html)
- [derive_builder](https://crates.io/crates/derive_builder)：一个自动实现此模式的 crate，可以避免编写样板代码
- [构造器模式](../../idioms/ctor.md)：适用于更简单的构造场景
- [Builder 模式（维基百科）](https://en.wikipedia.org/wiki/Builder_pattern)
- [复杂值的构造](https://web.archive.org/web/20210104103000/https://rust-lang.github.io/api-guidelines/type-safety.html#c-builder)
