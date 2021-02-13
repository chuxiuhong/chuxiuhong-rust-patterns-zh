# FFI 习惯用法

编写FFI的代码本身就是一门学问。
不过，这有一些习惯用法可以使其像指针一样操作，并且避免缺少经验的开发者陷入`unsafe`Rust的陷阱。

这一章中包括下列能在做FFI时有用的习惯用法：

1. [常见错误处理](./ffi-errors.md) - 使用整型代表错误类型以及哨兵返回值（sentinel）。
2. [接受字符串](./ffi-accepting-strings.md) 同时使用最少的unsafe代码。
3. [传递字符串](./ffi-passing-strings.md) 给FFI函数。
