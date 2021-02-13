# 新类型

如果在某些情况下，我们希望一个类型的行为类似于另一个类型，或者在编译时强制执行某些行为，而仅使用类型别名是不够的呢？
举例来说，如果我们出于安全考虑想要创建一个`String`的自定义的`Display`实现（例如密码）。
这种情况我们可以用`新类型`模式提供类型安全和封装。

## 说明

用带有单独字段的结构来创建一个类型的不透明包装器。这将创建一个新类型，而不是类型的别名。

## 代码示例

```rust,ignore
// Some type, not necessarily in the same module or even crate.
struct Foo {
    //..
}

impl Foo {
    // These functions are not present on Bar.
    //..
}

// The newtype.
pub struct Bar(Foo);

impl Bar {
    // Constructor.
    pub fn new(
        //..
    ) -> Bar {

        //..

    }

    //..
}

fn main() {
    let b = Bar::new(...);

    // Foo and Bar are type incompatible, the following do not type check.
    // let f: Foo = b;
    // let b: Bar = Foo { ... };
}
```

## 出发点

新类型的最初动机是抽象。其允许你在不同类型间共享实现代码并且精准控制接口。通过使用新类型而不是将实现作为API的一部分公开出去，它支持你向后兼容地更改实现。

新类型可以用来区分单位。例如封装`f64`类型为可辨识的`Miles`和`Kms`。

## 优点

被包装的类型和包装后的类型是不兼容的，所以新类型的用户永远不会困惑于区分这二者的类型。

新类型是零开销抽象——没有运行时负担。

隐私系统确保用户不能访问包装的类型（如果字段是私有的，默认私有）。

## 缺点

新类型的缺点（尤其是与类型别名比较），是没有特殊的语言支持。这就意味着会有大量的啰嗦的样板代码。对于要在包装类型上公开的每个方法，都需要一个穿透的方法，还有对包装器类型的实现来支持每一个想要的特性。

## 讨论

在Rust代码中新类型模式是很常见的。抽象或表达单元是最常见的用法，但他们也可以用于其他原因：

- 限制功能（减少暴露的函数或者特性实现），
- 使具有复制语义的类型具有移动语义
- 通过提供更具体的类型来进行抽象，从而隐藏内部类型，例如

```rust,ignore
pub struct Foo(Bar<T1, T2>);
```

在这里`Bar`也许是一个公开的泛型，`T1`和`T2`是一些内部类型。我们模块的用户不应该知道我们通过`Bar`来实现`Foo`，但是我们真正想隐藏的是类型`T1`和`T2`，以及他们是如何被`Bar`使用的。

## 参阅

- [Advanced Types in the book](https://doc.rust-lang.org/book/ch19-04-advanced-types.html?highlight=newtype#using-the-newtype-pattern-for-type-safety-and-abstraction)
- [Newtypes in Haskell](https://wiki.haskell.org/Newtype)
- [Type aliases](https://doc.rust-lang.org/stable/book/ch19-04-advanced-types.html#creating-type-synonyms-with-type-aliases)
- [derive_more](https://crates.io/crates/derive_more), a crate for deriving many
  builtin traits on newtypes.
- [The Newtype Pattern In Rust](https://www.worthe-it.co.za/blog/2020-10-31-newtype-pattern-in-rust.html)
