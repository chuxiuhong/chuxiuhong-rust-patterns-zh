# 传递字符串

## 描述

在向 FFI 函数传递字符串时，应遵循以下四个原则：

1. 使所拥有的字符串的生命周期尽可能长。
2. 在转换过程中最小化 `unsafe` 代码的使用。
3. 如果 C 代码可能修改字符串数据，使用 `Vec` 而不是 `CString`。
4. 除非外部函数 API 要求，字符串的所有权不应转移给被调用方。

## 动机

Rust 通过其 `CString` 和 `CStr` 类型内置了对 C 风格字符串的支持。然而，在从 Rust 函数向外部函数调用发送字符串时，可以采取不同的方法。

最佳实践很简单：以最小化 `unsafe` 代码的方式使用 `CString`。但是，还有一个次要的注意事项是*对象必须活得足够长*，这意味着生命周期应该被最大化。此外，文档说明在修改后"来回转换" `CString` 是未定义行为（UB），所以在这种情况下需要额外的工作。

## 代码示例

```rust,ignore
pub mod unsafe_module {

    // other module content

    extern "C" {
        fn seterr(message: *const libc::c_char);
        fn geterr(buffer: *mut libc::c_char, size: libc::c_int) -> libc::c_int;
    }

    fn report_error_to_ffi<S: Into<String>>(err: S) -> Result<(), std::ffi::NulError> {
        let c_err = std::ffi::CString::new(err.into())?;

        unsafe {
            // SAFETY: calling an FFI whose documentation says the pointer is
            // const, so no modification should occur
            seterr(c_err.as_ptr());
        }

        Ok(())
        // The lifetime of c_err continues until here
    }

    fn get_error_from_ffi() -> Result<String, std::ffi::IntoStringError> {
        let mut buffer = vec![0u8; 1024];
        unsafe {
            // SAFETY: calling an FFI whose documentation implies
            // that the input need only live as long as the call
            let written: usize = geterr(buffer.as_mut_ptr(), 1023).into();

            buffer.truncate(written + 1);
        }

        std::ffi::CString::new(buffer).unwrap().into_string()
    }
}
```

## 优势

这个示例的编写方式确保了：

1. `unsafe` 代码块尽可能小。
2. `CString` 的生命周期足够长。
3. 类型转换的错误总是在可能的情况下被传播。

一个常见的错误（如此常见以至于在文档中都有提到）是在第一个代码块中没有使用变量：

```rust,ignore
pub mod unsafe_module {

    // other module content

    fn report_error<S: Into<String>>(err: S) -> Result<(), std::ffi::NulError> {
        unsafe {
            // SAFETY: whoops, this contains a dangling pointer!
            seterr(std::ffi::CString::new(err.into())?.as_ptr());
        }
        Ok(())
    }
}
```

这段代码会导致悬垂指针，因为指针创建并不会延长 `CString` 的生命周期，这与创建引用的情况不同。

另一个经常被提出的问题是初始化一个包含 1k 个零的向量"很慢"。然而，最新版本的 Rust 实际上将该特定宏优化为对 `zmalloc` 的调用，这意味着它的速度取决于操作系统返回已清零内存的能力（这是相当快的）。

## 缺点

没有？
