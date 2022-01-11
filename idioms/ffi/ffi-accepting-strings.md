# 接受字符串

## 说明

当通过FFI的指针接受字符串时，有两条需要遵守的原则：

1. 保持对外部字符串的借用，而不是直接复制一份。
2. 在转换数据类型时最小化`unsafe`的代码区域。

## 出发点

Rust有对C语言风格字符串的内置支持，如`CString`和`CStr`类型。然而，有多种不同途径接受外部传入的字符串。

最佳实现是很简单的：用`CStr`最小化unsafe的代码区域，然后创建一个借用的切片。如果需要拥有其所有权的`String`，对字符串切片调用`to_string()`方法。

## 代码示例

```rust,ignore
pub mod unsafe_module {

    // other module content

    #[no_mangle]
    pub extern "C" fn mylib_log(msg: *const libc::c_char, level: libc::c_int) {
        let level: crate::LogLevel = match level { /* ... */ };

        let msg_str: &str = unsafe {
            // SAFETY: accessing raw pointers expected to live for the call,
            // and creating a shared reference that does not outlive the current
            // stack frame.
            match std::ffi::CStr::from_ptr(msg).to_str() {
                Ok(s) => s,
                Err(e) => {
                    crate::log_error("FFI string conversion failed");
                    return;
                }
            }
        };

        crate::log(msg_str, level);
    }
}
```

## 优点

样例能保证下面两点：

1. `unsafe`代码块尽可能的小。
2. 无法记录生命周期的指针转变为可以记录追踪的共享引用。

考虑另一种实现，也就是字符串被实际拷贝一份的情况：

```rust,ignore
pub mod unsafe_module {

    // other module content

    pub extern "C" fn mylib_log(msg: *const libc::c_char, level: libc::c_int) {
        // DO NOT USE THIS CODE.
        // IT IS UGLY, VERBOSE, AND CONTAINS A SUBTLE BUG.

        let level: crate::LogLevel = match level { /* ... */ };

        let msg_len = unsafe { /* SAFETY: strlen is what it is, I guess? */
            libc::strlen(msg)
        };

        let mut msg_data = Vec::with_capacity(msg_len + 1);

        let msg_cstr: std::ffi::CString = unsafe {
            // SAFETY: copying from a foreign pointer expected to live
            // for the entire stack frame into owned memory
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

这份代码与第一版相比有两个方面缺点:

1. 有更多的`unsafe`代码，更加不灵活。
2. 由于调用大量的算法，这个版本有一个会导致Rust的未定义行为（`undefined behaviour`）的bug。

这里的bug是一个简单的指针计算的错误：字符串被拷贝走`msg_len`个字节。然而没有包括在末尾的`NUL`终止符。

向量长度将会被设置为未做填充字符串的长度而不是末尾填一个0的调整后大小。因此，向量内的最后一个字节是没有初始化的内存。当最终创建`CString`时，其读取向量将会导致未定义行为！

像很多问题一样，这是很难查到的。有些时候它因为字符串不是`UTF-8`编码而产生恐慌，有时它又会在末尾放一个奇怪的字符，有时它会完全崩溃掉。

## 缺点

或许没有？
