# 生成器

## 说明

通过调用生成器来构造对象。

## 示例

```rust
#[derive(Debug, PartialEq)]
pub struct Foo {
    // Lots of complicated fields.
    bar: String,
}

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

## 出发点

当你需要很多不同的构造器或者构造器有副作用的时候这个模式会有帮助。

## 优点

将构造方法与其他方法分开。

防止构造器数量过多。

即使构造器本身很复杂，也可以做到封装后一行初始化。

## 缺点

与直接构造一个结构体或者一个简单的构造函数相比，这种方法太复杂。

## 讨论

因为Rust缺少重载功能，所以这种模式在Rust里比其他语言更常见。由于一个方法一个名称不能重载，所以Rust相比于C++、Java来说更不适合写很多构造器。

这种模式经常不是为了作为构造器而设计。例如[`std::process::Command`](https://doc.rust-lang.org/std/process/struct.Command.html)
是 [`Child`](https://doc.rust-lang.org/std/process/struct.Child.html)的构造器（一个进程）。这种情况下没有使用`T`和`TBuilder`命名模式。

下面的例子按值获取和返回。然而更符合人体工程学（以及更效率）的方法是按可变引用获取和返回。借用检查器将会帮助我们。传入传出可变引用将会让我们从下面这种代码：

```rust,ignore
let mut fb = FooBuilder::new();
fb.a();
fb.b();
let f = fb.build();
```

转变为`FooBuilder::new().a().b().build()` 风格代码。

## 参阅

- [Description in the style guide](https://web.archive.org/web/20210104103100/https://doc.rust-lang.org/1.12.0/style/ownership/builders.html)
- [derive_builder](https://crates.io/crates/derive_builder), a crate for automatically
  implementing this pattern while avoiding the boilerplate.
- [Constructor pattern](../../idioms/ctor.md) for when construction is simpler.
- [Builder pattern (wikipedia)](https://en.wikipedia.org/wiki/Builder_pattern)
- [Construction of complex values](https://web.archive.org/web/20210104103000/https://rust-lang.github.io/api-guidelines/type-safety.html#c-builder)
