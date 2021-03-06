# 留隐私，为扩展

## 说明

使用私有字段确保结构体日后可以在不破坏API稳定性的情况下进一步扩展。

## 示例

```rust,ignore
mod a {
    // Public struct.
    pub struct S {
        pub foo: i32,
        // Private field.
        bar: i32,
    }
}

fn main(s: a::S) {
    // Because S::bar is private, it cannot be named here and we must use `..`
    // in the pattern.
    let a::S { foo: _, ..} = s;
}

```

## 讨论

向结构体中增加字段通常是保持向后兼容的。然而，如果调用者用模式匹配来解构这个结构体的实例，他可能对现有结构体内的所有公开字段进行修改，
这样就破坏了我们的设计。或者调用者可以对一部分字段赋值，然后在模式里用`..`来自动增加其他可匹配的字段。
使用私有字段至少强迫调用者必须用后一种方法，确保结构体是未来是可以扩展的。

这种实现的缺点是你可能需要增加用不到的变量。你可以用`()`基类型来确保不增加运行时开销，`_`作为字段名可以避免没使用变量的编译器警告。

如果Rust支持枚举的私有成员，我们就可以用相同的技巧来对一个枚举增加新的成员。
但是不用通配符而是穷尽的模式匹配会导致未实现的问题。私有成员会强迫调用者在模式匹配是加上`_`通配符。这个问题的常规解决方法是加上`#[non_exhaustive]`属性。
