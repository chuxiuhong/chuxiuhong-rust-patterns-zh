# 使用 `format!` 连接字符串

## 描述

虽然可以通过在可变的 `String` 上使用 `push` 和 `push_str` 方法，或使用其 `+` 运算符来构建字符串。但是，使用 `format!` 通常更加方便，特别是在需要混合使用字面量和非字面量字符串的情况下。

## 示例

```rust
fn say_hello(name: &str) -> String {
    // We could construct the result string manually.
    // let mut result = "Hello ".to_owned();
    // result.push_str(name);
    // result.push('!');
    // result

    // But using format! is better.
    format!("Hello {name}!")
}
```

## 优点

使用 `format!` 通常是组合字符串最简洁和可读性最好的方式。

## 缺点

这通常不是组合字符串最高效的方式 - 在可变字符串上执行一系列 `push` 操作通常是最高效的（特别是当字符串已经预分配了预期的大小时）。
