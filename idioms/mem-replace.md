# 用`mem::{take(_), replace(_)}`在修改枚举变体时保持值的所有权

## 说明

假设我们有一个至少有两种变体的枚举`&mut MyEnum`，一种是`A { name: String, x: u8 }`，另一种是`B { name: String }`。现在我们想要当x=0时，将A变为B，同时变量除枚举类型变化外其他不变。

我们可以不用克隆`name`变体即可实现上述操作。

## 例子

```rust
use std::mem;

enum MyEnum {
    A { name: String, x: u8 },
    B { name: String }
}

fn a_to_b(e: &mut MyEnum) {

    // we mutably borrow `e` here. This precludes us from changing it directly
    // as in `*e = ...`, because the borrow checker won't allow it. Therefore
    // the assignment to `e` must be outside the `if let` clause.
    *e = if let MyEnum::A { ref mut name, x: 0 } = *e {

        // this takes out our `name` and put in an empty String instead
        // (note that empty strings don't allocate).
        // Then, construct the new enum variant (which will
        // be assigned to `*e`, because it is the result of the `if let` expression).
        MyEnum::B { name: mem::take(name) }

    // In all other cases, we return immediately, thus skipping the assignment
    } else { return }
}
```

这种方法对多种枚举变体也适用:

```rust
use std::mem;

enum MultiVariateEnum {
    A { name: String },
    B { name: String },
    C,
    D
}

fn swizzle(e: &mut MultiVariateEnum) {
    use MultiVariateEnum::*;
    *e = match *e {
        // Ownership rules do not allow taking `name` by value, but we cannot
        // take the value out of a mutable reference, unless we replace it:
        A { ref mut name } => B { name: mem::take(name) },
        B { ref mut name } => A { name: mem::take(name) },
        C => D,
        D => C
    }
}
```

## 出发点

当使用枚举的时候，我们可能想要改变枚举变体类型为其他类型。为了通过借用检查器检查，我们将分为两个阶段。在第一阶段，我们查看现有的值然后决定下一步怎么做。第二阶段我们可以修改值。


借用检查器不允许我们拿走`name`字段的值（因为那总得有有个东西放在那啊）。我们当然可以用`.clone()`克隆一个`name`的值，然后把这个克隆的值赋给`MyEnum::B`，不过这样就是一个反模式的实例（为了满足借用检查器就用克隆，增大了开销）。综上，我们可以通过仅仅一个可变借用来改变值，避免多余的空间申请。


`mem::take`支持我们交换值，用默认值替换，并且返回原值。对于`String`类型，默认值是一个空字符串，无需申请空间。因此，我们获取原来的`name`(作为一个拥有值的变量)，我们可以把它包装成另一个枚举。

注：`mem:replace`非常相似，不过其允许我们指定要替换的值。可以用它实现`mem::take`的功能：`mem::replace(name,String::new())`。

然而，如果我们要使用`Option`的默认值替换掉枚举变体的值，那么用`take()`方法还是更习惯和简便的。

## 优点

看好啦，没有内存申请！同时你在这么做的时候会感觉自己像Indiana Jones。（译者注：没看过夺宝奇兵，没get到梗）

## 缺点

这会变得优点啰嗦。如果错误地重复这个操作将会让你厌恶借用检查器。编译器将无法对替换操作优化，结果是让你觉得相比其他不安全的语言来说性能更低。

此外，`take`操作需要类型实现[`Default`](./default.md)特性。然而，如果这个类型没有实现`Default`特性，你还是可以用 `mem::replace`。

## 讨论

这个模式是只属于Rust的特点。在带GC的语言中，你可以直接用引用来替换。（GC会记录有哪些引用），在像C语言这些低级语言中你可以简单地给指针取个别名然后解决问题。

然而，在Rust中，我们不得不再多做一点工作。一个值只能有一个所有者，所以把值取走后，我们必须再往里面放点东西填充就像印第安纳琼斯一样，用一包沙子替换了宝物。

## 参阅

这在特定情况下可以消除利用克隆通过借用检查器的反模式。

[Clone to satisfy the borrow checker](TODO: Hinges on PR #23)
