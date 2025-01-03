# 接受字符串

## 描述

通过 FFI 接受字符串指针时，应遵循以下两个原则：

1. 保持外部字符串为"借用"状态，而不是直接复制它们。
2. 在将 C 风格字符串转换为原生 Rust 字符串时，尽量减少复杂性和 `unsafe` 代码的使用。

## 动机

C 语言中使用的字符串与 Rust 中使用的字符串有不同的行为特征：

- C 字符串以空字符结尾，而 Rust 字符串存储其长度
- C 字符串可以包含任意非零字节，而 Rust 字符串必须是 UTF-8 编码
- C 字符串通过 `unsafe` 指针操作来访问和操作，而 Rust 字符串则通过安全方法进行交互

Rust 标准库提供了与 Rust 的 `String` 和 `&str` 相对应的 C 语言版本，即 `CString` 和 `&CStr`，这使我们可以避免在 C 字符串和 Rust 字符串之间转换时涉及大量的复杂性和 `unsafe` 代码。

`&CStr` 类型还允许我们使用借用数据，这意味着在 Rust 和 C 之间传递字符串是一个零成本操作。

## 代码示例

```rust,ignore
pub mod unsafe_module {

    // 其他模块内容

    /// 在指定级别记录消息。
    ///
    /// # 安全性
    ///
    /// 调用者需要确保 `msg`：
    ///
    /// - 不是空指针
    /// - 指向有效的、已初始化的数据
    /// - 指向以空字节结尾的内存
    /// - 在函数调用期间不会被修改
    #[no_mangle]
    pub unsafe extern "C" fn mylib_log(msg: *const libc::c_char, level: libc::c_int) {
        let level: crate::LogLevel = match level { /* ... */ };

        // 安全性：调用者已经保证这是可以的（参见文档注释中的 `# 安全性` 部分）
        let msg_str: &str = match std::ffi::CStr::from_ptr(msg).to_str() {
            Ok(s) => s,
            Err(e) => {
                crate::log_error("FFI string conversion failed");
                return;
            }
        };

        crate::log(msg_str, level);
    }
}
```

## 优点

这个示例的编写确保了：

1. `unsafe` 代码块尽可能小。
2. 具有"未追踪"生命周期的指针变成了"已追踪"的共享引用。

考虑另一种实际复制字符串的替代方案：

```rust,ignore
pub mod unsafe_module {

    // 其他模块内容

    pub extern "C" fn mylib_log(msg: *const libc::c_char, level: libc::c_int) {
        // 请勿使用此代码。
        // 它很丑陋、冗长，而且包含一个微妙的错误。

        let level: crate::LogLevel = match level { /* ... */ };

        let msg_len = unsafe { /* 安全性：strlen 就是这样的 */ 
            libc::strlen(msg)
        };

        let mut msg_data = Vec::with_capacity(msg_len + 1);

        let msg_cstr: std::ffi::CString = unsafe {
            // 安全性：从预期在整个栈帧中存活的外部指针复制到拥有所有权的内存中
            std::ptr::copy_nonoverlapping(msg, msg_data.as_mut(), msg_len);

            msg_data.set_len(msg_len + 1);

            std::ffi::CString::from_vec_with_nul(msg_data).unwrap()
        }

        let msg_str: String = unsafe {
            match msg_cstr.into_string() {
                Ok(s) => s,
                Err(e) => {
                    crate::log_error("FFI string conversion failed");
                    return;
                }
            }
        };

        crate::log(&msg_str, level);
    }
}
```

这段代码相比原始版本有两个方面的劣势：

1. 包含了更多的 `unsafe` 代码，更重要的是，需要维护更多的不变量。
2. 由于需要进行大量的算术运算，这个版本中存在一个会导致 Rust `undefined behaviour`（未定义行为）的错误。

这里的错误是一个简单的指针算术错误：字符串被复制了所有的 `msg_len` 字节，但是末尾的 `NUL` 终止符没有被复制。

然后向量的大小被*设置*为带零填充字符串的长度，而不是*调整*到这个长度（这本应该在末尾添加一个零）。结果，向量中的最后一个字节是未初始化的内存。当在代码块底部创建 `CString` 时，它对向量的读取将导致 `undefined behaviour`！

像许多类似的问题一样，这将是一个难以追踪的问题。有时它会因为字符串不是 `UTF-8` 而发生 panic，有时它会在字符串末尾放置一个奇怪的字符，有时它会完全崩溃。

## 缺点

似乎没有？
