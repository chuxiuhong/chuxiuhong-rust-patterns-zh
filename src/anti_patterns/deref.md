# `Deref` 多态

## 描述

错误地使用 `Deref` trait 来模拟结构体之间的继承关系，从而重用方法。

## 示例

有时我们想要模拟类似 Java 等面向对象语言中的常见模式：

```java
class Foo {
    void m() { ... }
}

class Bar extends Foo {}

public static void main(String[] args) {
    Bar b = new Bar();
    b.m();
}
```

我们可以使用 deref 多态反模式来实现这一点：

```rust
use std::ops::Deref;

struct Foo {}

impl Foo {
    fn m(&self) {
        //..
    }
}

struct Bar {
    f: Foo,
}

impl Deref for Bar {
    type Target = Foo;
    fn deref(&self) -> &Foo {
        &self.f
    }
}

fn main() {
    let b = Bar { f: Foo {} };
    b.m();
}
```

Rust 中没有结构体继承。相反，我们使用组合，在 `Bar` 中包含一个 `Foo` 的实例（由于该字段是一个值，它是内联存储的，所以如果有字段的话，它们在内存中的布局与 Java 版本相同（可能的话，如果你想确保这一点，应该使用 `#[repr(C)]`））。

为了使方法调用正常工作，我们为 `Bar` 实现了 `Deref` trait，并将 `Foo` 作为目标类型（返回嵌入的 `Foo` 字段）。这意味着当我们解引用一个 `Bar`（例如，使用 `*`）时，我们会得到一个 `Foo`。这很奇怪。解引用通常是从 `T` 的引用得到 `T`，而这里我们有两个不相关的类型。然而，由于点运算符会进行隐式解引用，这意味着方法调用会在 `Foo` 和 `Bar` 上都搜索方法。

## 优点

你可以节省一些样板代码，例如：

```rust,ignore
impl Bar {
    fn m(&self) {
        self.f.m()
    }
}
```

## 缺点

最重要的是，这是一个令人意外的用法 - 未来阅读这段代码的程序员不会期望这种情况发生。这是因为我们在滥用 `Deref` trait，而不是按照其设计意图（以及文档等）来使用它。同时也因为这里的机制是完全隐式的。

这种模式并不会像 Java 或 C++ 中的继承那样在 `Foo` 和 `Bar` 之间引入子类型关系。此外，`Foo` 实现的 trait 不会自动为 `Bar` 实现，所以这种模式与边界检查以及泛型编程的交互并不理想。

使用这种模式在 `self` 的语义上与大多数面向对象语言有着微妙的区别。通常 `self` 保持对子类的引用，而在这种模式中，它将是定义方法的"类"。

最后，这种模式只支持单继承，并且没有接口、基于类的私有性或其他与继承相关的特性的概念。因此，对于习惯于 Java 继承等的程序员来说，这种体验会带来微妙的意外。

## 讨论

没有一个完美的替代方案。根据具体情况，使用 trait 重新实现或手动编写外观方法来分派到 `Foo` 可能会更好。我们确实打算为 Rust 添加类似这样的继承机制，但可能需要一段时间才能进入稳定版 Rust。更多细节请参见这些[博客](http://aturon.github.io/blog/2015/09/18/reuse/)[文章](http://smallcultfollowing.com/babysteps/blog/2015/10/08/virtual-structs-part-4-extended-enums-and-thin-traits/)和这个 [RFC issue](https://github.com/rust-lang/rfcs/issues/349)。

`Deref` trait 是为实现自定义指针类型而设计的。其目的是将指向 `T` 的指针转换为 `T`，而不是在不同类型之间转换。遗憾的是，这一点无法（可能无法）通过 trait 定义来强制执行。

Rust 试图在显式和隐式机制之间取得平衡，倾向于类型之间的显式转换。点运算符中的自动解引用是一个例子，其中人体工程学强烈倾向于隐式机制，但目的是将其限制在间接程度上，而不是在任意类型之间进行转换。

## 另请参阅

- [集合是智能指针习语](../idioms/deref.md)
- 用于减少样板代码的委托 crate，如 [delegate](https://crates.io/crates/delegate) 或 [ambassador](https://crates.io/crates/ambassador)
- [`Deref` trait 的文档](https://doc.rust-lang.org/std/ops/trait.Deref.html)
