# 构造器

## 描述

Rust 没有将构造器作为语言结构。相反，约定是使用一个 [关联函数][associated function] `new` 来创建对象：

````rust
/// Time in seconds.
///
/// # Example
///
/// ```
/// let s = Second::new(42);
/// assert_eq!(42, s.value());
/// ```
pub struct Second {
    value: u64,
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
````

## 默认构造器

Rust 通过 [`Default`][std-default] trait 支持默认构造器：

````rust
/// Time in seconds.
///
/// # Example
///
/// ```
/// let s = Second::default();
/// assert_eq!(0, s.value());
/// ```
pub struct Second {
    value: u64,
}

impl Second {
    /// Returns the value in seconds.
    pub fn value(&self) -> u64 {
        self.value
    }
}

impl Default for Second {
    fn default() -> Self {
        Self { value: 0 }
    }
}
````

如果所有字段的类型都实现了 `Default`，那么也可以直接派生 `Default`，就像 `Second` 这样：

````rust
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
    value: u64,
}

impl Second {
    /// Returns the value in seconds.
    pub fn value(&self) -> u64 {
        self.value
    }
}
````

**注意：** 类型同时实现 `Default` 和一个无参数的 `new` 构造器是很常见且符合预期的。`new` 是 Rust 中的构造器约定，用户期望它的存在，所以如果基本构造器合理地不需要参数，那么就应该实现它，即使它在功能上与 default 完全相同。

**提示：** 实现或派生 `Default` 的优势在于，你的类型现在可以在需要 `Default` 实现的地方使用，最显著的例子是标准库中的任何 [`*or_default` 函数][std-or-default]。

## 另见

- [default 惯用法](default.md) 中有关于 `Default` trait 更深入的描述。

- [构建器模式](../patterns/creational/builder.md) 用于构造具有多种配置的对象。

- [API Guidelines/C-COMMON-TRAITS][API Guidelines/C-COMMON-TRAITS] 用于同时实现 `Default` 和 `new`。

[associated function]: https://doc.rust-lang.org/stable/book/ch05-03-method-syntax.html#associated-functions
[std-default]: https://doc.rust-lang.org/stable/std/default/trait.Default.html
[std-or-default]: https://doc.rust-lang.org/stable/std/?search=or_default
[API Guidelines/C-COMMON-TRAITS]: https://rust-lang.github.io/api-guidelines/interoperability.html#types-eagerly-implement-common-traits-c-common-traits
