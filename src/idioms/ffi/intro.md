# FFI 惯用法

编写 FFI 代码本身就是一门完整的课程。然而，这里提供了几个惯用法作为指引，可以帮助 `unsafe` Rust 的新手避免一些常见陷阱。

本节包含在进行 FFI 开发时可能有用的惯用法。

1. [惯用的错误处理](./errors.md) - 使用整数错误码和哨兵返回值（例如 `NULL` 指针）进行错误处理

2. [接收字符串](./accepting-strings.md) - 使用最少的 unsafe 代码

3. [传递字符串](./passing-strings.md) 到 FFI 函数
