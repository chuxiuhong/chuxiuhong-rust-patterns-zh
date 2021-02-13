# 关于初始化的文档

## 说明

如果一个结构体初始化操作很复杂，当写文档的时候，可以在文档中写一个使用样例的函数。

## 出发点

有时候结构体有多个或者很复杂的参数和一堆方法。每个方法都应该有相应的例子说明。

举例来说:

```rust,ignore
struct Connection {
    name: String,
    stream: TcpStream,
}

impl Connection {
    /// Sends a request over the connection.
    ///
    /// # Example
    /// ```no_run
    /// # // Boilerplate are required to get an example working.
    /// # let stream = TcpStream::connect("127.0.0.1:34254");
    /// # let connection = Connection { name: "foo".to_owned(), stream };
    /// # let request = Request::new("RequestId", RequestType::Get, "payload");
    /// let response = connection.send_request(request);
    /// assert!(response.is_ok());
    /// ```
    fn send_request(&self, request: Request) -> Result<Status, SendErr> {
        // ...
    }

    /// Oh no, all that boilerplate needs to be repeated here!
    fn check_status(&self) -> Status {
        // ...
    }
}
```

## 示例

不用每次都写初始化的部分，主要写一个以这个结构体为参数的函数的用法即可。

```rust,ignore
struct Connection {
    name: String,
    stream: TcpStream,
}

impl Connection {
    /// Sends a request over the connection.
    ///
    /// # Example
    /// ```
    /// # fn call_send(connection: Connection, request: Request) {
    /// let response = connection.send_request(request);
    /// assert!(response.is_ok());
    /// # }
    /// ```
    fn send_request(&self, request: Request) {
        // ...
    }
}
```

**注意**：上面的例子里的 `assert!(response.is_ok());` 不会真的执行，因为其所在的函数并没有被调用。

## 优点

这样更简洁。

## 缺点

作为例子的函数不会被真的测试。但是在`cargo test`的时候还是会检查能不能编译通过。所以这个模式是在需要`no_run`的时候更能彰显作用，这样写就不必用`no_run`。

## 讨论

如果不需要断言，那么这种模式就可以很好地工作。

如果需要，另一个方法是创建一个公开的方法来创建用`#[doc(hidden)]`注释的帮助示例（这样用户就看不见）。因为这是包里的公开API，所以在rustdoc里会显示这个方法。