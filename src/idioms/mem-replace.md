# 使用 `mem::{take(_), replace(_)}` 在修改枚举时保持所有权值

## 描述

假设我们有一个 `&mut MyEnum`，它（至少）有两个变体：`A { name: String, x: u8 }` 和 `B { name: String }`。现在我们想要在 `x` 为零时将 `MyEnum::A` 转换为 `B`，同时保持 `MyEnum::B` 不变。

我们可以在不克隆 `name` 的情况下完成这个操作。

## 示例

```rust
use std::mem;

enum MyEnum {
    A { name: String, x: u8 },
    B { name: String },
}

fn a_to_b(e: &mut MyEnum) {
    if let MyEnum::A { name, x: 0 } = e {
        // 这里取出我们的 `name` 并替换为一个空字符串
        // （注意空字符串不会分配内存）
        // 然后，构造新的枚举变体（它将被赋值给 `*e`）
        *e = MyEnum::B {
            name: mem::take(name),
        }
    }
}
```

这种方法也适用于更多变体的情况：

```rust
use std::mem;

enum MultiVariateEnum {
    A { name: String },
    B { name: String },
    C,
    D,
}

fn swizzle(e: &mut MultiVariateEnum) {
    use MultiVariateEnum::*;
    *e = match e {
        // 所有权规则不允许按值获取 `name`，但我们不能从可变引用中
        // 取出值，除非我们替换它：
        A { name } => B {
            name: mem::take(name),
        },
        B { name } => A {
            name: mem::take(name),
        },
        C => D,
        D => C,
    }
}
```

## 动机

在处理枚举时，我们可能想要就地改变一个枚举值，可能是改变为另一个变体。为了让借用检查器满意，这通常分两个阶段完成。在第一阶段，我们观察现有值并查看其组成部分，以决定下一步该做什么。在第二阶段，我们可以有条件地更改值（如上例所示）。

借用检查器不允许我们从枚举中取出 `name`（因为那里*必须*有些东西）。我们当然可以 `.clone()` name 并将克隆放入我们的 `MyEnum::B` 中，但这将是一个[为满足借用检查器而克隆](../anti_patterns/borrow_clone.md)的反模式实例。无论如何，我们可以通过仅使用可变借用来避免额外的内存分配。

`mem::take` 让我们可以交换出值，用其默认值替换它，并返回先前的值。对于 `String` 来说，默认值是一个空的 `String`，它不需要分配内存。因此，我们获得了原始的 `name` *作为一个拥有所有权的值*。然后我们可以将其包装在另一个枚举中。

**注意：** `mem::replace` 非常相似，但允许我们指定用什么值来替换。与我们的 `mem::take` 行等效的写法是 `mem::replace(name, String::new())`。

但是，如果我们使用的是 `Option` 并想要将其值替换为 `None`，那么 `Option` 的 `take()` 方法提供了一个更简短和更惯用的替代方案。

## 优点

看，没有内存分配！而且在使用时你可能会感觉自己像印第安纳·琼斯。

## 缺点

这种写法有点啰嗦。反复写错会让你讨厌借用检查器。编译器可能无法优化掉双重存储，与在不安全语言中的做法相比，这可能会导致性能下降。

此外，你要获取的类型需要实现 [`Default` trait](./default.md)。不过，如果你使用的类型没有实现这个 trait，你可以使用 `mem::replace` 作为替代。

## 讨论

这种模式只在 Rust 中才有意义。在使用 GC 的语言中，你默认会获取值的引用（GC 会跟踪引用），而在 C 这样的其他低级语言中，你只需简单地别名指针，稍后再修复即可。

然而，在 Rust 中，我们需要做更多工作来实现这一点。一个拥有所有权的值只能有一个所有者，所以要取出它，我们需要放回一些东西 —— 就像印第安纳·琼斯用一袋沙子替换神器一样。

## 另请参阅

这种方法在特定情况下消除了[为满足借用检查器而克隆](../anti_patterns/borrow_clone.md)的反模式。
