# 传递字符串

## 说明

当传递字符串给FFI函数时，有以下4点需要遵守的原则：

1. 让拥有的字符串生命周期尽可能长。
2. 在转换时保持最小化`unsafe`区域代码。
3. 如果C语言代码会修改字符串数据，那么使用`Vec`类型而不是`CString`。
4. 除非外部函数的API需要字符串的所有权，否则不要传给被调用的函数。

## 出发点


Rust有对C语言风格字符串的内置支持，如`CString`和`CStr`类型。不过，有多种不同途径从Rust函数传给FFI函数字符串的方法。


最佳实现是很简单的：用`CSring`最小化unsafe的代码区域。然而，第二个警告是*对象必须生存足够长时间*，意味着生命周期应该最大化。此外，在修改后双向传递`CStirng`类型的对象是未定义行为，这种情况需要额外的操作来完善。

## 代码示例

```rust,ignore
pub mod unsafe_module {

    // other module content

    extern "C" {
        fn seterr(message: *const libc::c_char);
        fn geterr(buffer: *mut libc::c_char, size: libc::c_int) -> libc::c_int;
    }

    fn report_error_to_ffi<S: Into<String>>(
        err: S
    ) -> Result<(), std::ffi::NulError>{
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

## 优点

样例能保证下面三点：
1. unsafe`代码块尽可能的小。
2. `CString`生命周期足够长
3. 类型转换时发生的错误能够尽早地传播出来。

一个常见（在文档中很常见）的错误是在代码块的开头部分不定义变量。

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

这样的代码会导致悬垂指针，因为`CString`的生命周期并没有因为创建指针而延长，不像创建一个引用那样。



另一个经常提到的问题是初始化一个全0的1K长度的向量很慢。然而，最新的Rust版本针对这种情况提供了一个宏调用`zmalloc`，和操作系统能返回全0内存的速度一样快。（真的很快）

## 缺点

或许没有？
