# `Deref` 多态

## 说明

滥用`Deref`特性来模拟结构体之间的继承，从而重用方法。

## 代码示例

有时我们想要从诸如Java之类的面向对象语言中模拟以下常见模式：

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

我们可以用deref多态反模式来实现：

```rust,ignore
use std::ops::Deref;

struct Foo {}

impl Foo {
    fn m(&self) {
        //..
    }

}

struct Bar {
    f: Foo
}

impl Deref for Bar {
    type Target = Foo;
    fn deref(&self) -> &Foo {
        &self.f
    }
}

fn main() {
    let b = Bar { Foo {} };
    b.m();
}
```

Rust中没有结构体的继承。取而代之的是我们使用组合方式在`Bar`内包含`Foo`（因为字段是一个值，它在内部存储），因此它们都是字段，拥有和Java版本相同的内存布局。（如果你想要确保这一点，可以用`#[repr(C)]`）。

为了使方法调用有效，我们为`Bar`实现了`Deref`特性，生成目标为`Foo`（返回的是内置的`Foo`字段）。这就相当于当我们对`Bar`解引用的时候我们就会获取到一个`Foo`对象。这是非常诡异的，解引用通常是通过一个类型的引用获取这个类型的值，然而这里却是两种不相关的类型。不过，因为点运算符是隐式的解引用，所以方法调用时也将搜索`Foo`类型的方法。

## 优点

节省了一些样板代码，例如：

```rust,ignore
impl Bar {
    fn m(&self) {
        self.f.m()
    }
}
```

## 缺点

最重要的是这是一个令人惊讶的习惯用法——未来的程序员在阅读这些代码时不会期望发生这种情况。这是因为我们滥用了`Deref`特性，而不是按预期的那样去使用。同时也是因为这里的机制是完全隐式的。

这种模式并没有实现像Java或者C++里的继承。此外，对`Foo`实现的特性也不会自动地适用于`Boo`，所以这种模式对于边界检查和泛型编程来说非常差。

使用这种模式，就`self`而言，给出了与大多数面向对象语言截然不同的语义。通常它仍是子类型的引用，在这种模式下它将是定义方法的“类”。

最后，这种模式仅支持单继承，并且没有接口的概念、基于类的隐私性或者其他的与继承相关的特性。因此，对于习惯于Java那种继承的程序员来说，它提供了一种“惊喜”。

## 讨论

这没有好的替代方案。根据具体情况，最好用特性重新实现，或者手动编写分发给`Foo`的方法。我们确实打算为Rust添加一种像这样的继承机制，但是可能需要一段时间才能进入稳定版本的Rust。看这些 [博客](http://aturon.github.io/blog/2015/09/18/reuse/)、[文章](http://smallcultfollowing.com/babysteps/blog/2015/10/08/virtual-structs-part-4-extended-enums-and-thin-traits/) 和这个[RFC issue](https://github.com/rust-lang/rfcs/issues/349) 来了解更多细节。

`Deref`特性是被设计用来实现自定义指针类型的。它的用处是将`T`的引用转变为`T`的值，而不是在类型间转换。遗憾的是，这不是（或者说无法）靠特性定义来强制执行。


Rust尝试在显式和隐式机制之间做出权衡，更偏向于类型间进行显式转换。点运算符自动解引用是出于符合人体工程学的角度做的隐式设计，其目的仅限于有限的间接程度，而不是任意类型之间做隐式转换。

## 参阅

- [Collections are smart pointers idiom](../idioms/deref.md).
- Delegation crates for less boilerplate like [delegate](https://crates.io/crates/delegate)
  or [ambassador](https://crates.io/crates/ambassador)
- [Documentation for `Deref` trait](https://doc.rust-lang.org/std/ops/trait.Deref.html).
