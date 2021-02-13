# 构造器

## 说明

Rust 没有语言层面的构造器。取而代之的是常用一个静态的`new`方法创建一个对象作为构造器。

## 例子

```rust,ignore
// A Rust vector, see liballoc/vec.rs
pub struct Vec<T> {
    buf: RawVec<T>,
    len: usize,
}

impl<T> Vec<T> {
    // 构造一个新的 `Vec<T>`.
    // 注意这是一个静态方法
    // 这个构造器不需要任何参数，但有些需要参数来初始化对象
    pub fn new() -> Vec<T> {
        // Create a new Vec with fields properly initialised.
        Vec {
            // 注意我们这里调用的是RawVec类型的构造器
            buf: RawVec::new(),
            len: 0,
        }
    }
}
```

## 参阅

[生成器模式](../patterns/builder.md)用于有多种构造对象方式的情况。
