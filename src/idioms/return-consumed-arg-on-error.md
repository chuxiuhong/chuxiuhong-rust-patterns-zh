# 错误时返回被消耗的参数

## 描述

如果一个可能失败的函数消耗（移动）了一个参数，应当在错误发生时将该参数作为错误的一部分返回。

## 示例

```rust
pub fn send(value: String) -> Result<(), SendError> {
    println!("using {value} in a meaningful way");
    // Simulate non-deterministic fallible action.
    use std::time::SystemTime;
    let period = SystemTime::now()
        .duration_since(SystemTime::UNIX_EPOCH)
        .unwrap();
    if period.subsec_nanos() % 2 == 1 {
        Ok(())
    } else {
        Err(SendError(value))
    }
}

pub struct SendError(String);

fn main() {
    let mut value = "imagine this is very long string".to_string();

    let success = 's: {
        // Try to send value two times.
        for _ in 0..2 {
            value = match send(value) {
                Ok(()) => break 's true,
                Err(SendError(value)) => value,
            }
        }
        false
    };

    println!("success: {success}");
}
```

## 动机

在发生错误的情况下，你可能想尝试一些替代方案或者在非确定性函数中重试该操作。但如果参数总是被消耗掉，你就不得不在每次调用时克隆它，这样的效率并不高。

标准库在一些地方使用了这种方法，例如 `String::from_utf8` 方法。当给定的向量不包含有效的 UTF-8 时，会返回一个 `FromUtf8Error`。你可以使用 `FromUtf8Error::into_bytes` 方法获取回原始向量。

## 优点

由于尽可能地移动参数而不是复制，因此性能更好。

## 缺点

错误类型略微复杂一些。
