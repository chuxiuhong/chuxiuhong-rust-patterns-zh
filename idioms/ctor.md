# 构造器

## 说明

Rust 没有语言层面的构造器。
取而代之的是常用一个[关联函数][] `new` 创建对象：

## 示例

```rust
/// Time in seconds.
///
/// # Example
///
/// ```
/// let s = Second::new(42);
/// assert_eq!(42, s.value());
/// ```
pub struct Second {
    value: u64
}
impl Second {
    // Constructs a new instance of [`Second`].
    // Note this is an associated function - no self.
    pub fn new(value: u64) -> Self {
        Self { value }
    }
    /// Returns the value in seconds.
    pub fn value(&self) -> u64 {
        self.value
    }
}
```

## Default Constructors

Rust supports default constructors with the [`Default`][std-default] trait:

```rust,ignore
// A Rust vector, see liballoc/vec.rs
pub struct Vec<T> {
    buf: RawVec<T>,
    len: usize,
```rust
/// Time in seconds.
///
/// # Example
///
/// ```
/// let s = Second::default();
/// assert_eq!(0, s.value());
/// ```
pub struct Second {
    value: u64
}
impl Second {
    /// Returns the value in seconds.
    pub fn value(&self) -> u64 {
        self.value
    }
}
impl<T> Vec<T> {
    // Constructs a new, empty `Vec<T>`.
    // Note this is a static method - no self.
    // This constructor doesn't take any arguments, but some might in order to
    // properly initialise an object
    pub fn new() -> Vec<T> {
        // Create a new Vec with fields properly initialised.
        Vec {
            // Note that here we are calling RawVec's constructor.
            buf: RawVec::new(),
            len: 0,
        }
impl Default for Second {
    fn default() -> Self {
        Self { value: 0 }
    }
}
```

`Default` can also be derived if all types of all fields implement `Default`,
like they do with `Second`:

```rust
/// Time in seconds.
///
/// # Example
///
/// ```
/// let s = Second::default();
/// assert_eq!(0, s.value());
/// ```
#[derive(Default)]
pub struct Second {
    value: u64
}
impl Second {
    /// Returns the value in seconds.
    pub fn value(&self) -> u64 {
        self.value
    }
}
```

**Note:** When implementing `Default` for a type, it is neither required nor
recommended to also provide an associated function `new` without arguments.

**Hint:** The advantage of implementing or deriving `Default` is that your type
can now be used where a `Default` implementation is required, most prominently,
any of the [`*or_default` functions in the standard library][std-or-default].

## 参阅

- [default idiom](default.md)有对`Default` trait更深入的介绍。
- [生成器模式](../patterns/creational/builder.md)用于有多种构造对象方式的情况。

[associated function]: https://doc.rust-lang.org/stable/book/ch05-03-method-syntax.html#associated-functions
[std-default]: https://doc.rust-lang.org/stable/std/default/trait.Default.html
[std-or-default]: https://doc.rust-lang.org/stable/std/?search=or_default

