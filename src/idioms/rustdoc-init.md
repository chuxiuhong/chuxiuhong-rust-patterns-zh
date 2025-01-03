# 简化文档初始化

## 描述

当在编写文档时，如果一个结构体的初始化过程比较复杂，可以通过创建一个以该结构体为参数的辅助函数来简化示例代码。

## 动机

有时候一个结构体可能有多个参数或复杂的参数，以及多个方法。这些方法都需要示例。

例如：

````rust,ignore
struct Connection {
    name: String,
    stream: TcpStream,
}

impl Connection {
    /// 通过连接发送请求。
    ///
    /// # 示例
    /// ```no_run
    /// # // 需要样板代码来使示例正常工作。
    /// # let stream = TcpStream::connect("127.0.0.1:34254");
    /// # let connection = Connection { name: "foo".to_owned(), stream };
    /// # let request = Request::new("RequestId", RequestType::Get, "payload");
    /// let response = connection.send_request(request);
    /// assert!(response.is_ok());
    /// ```
    fn send_request(&self, request: Request) -> Result<Status, SendErr> {
        // ...
    }

    /// 糟糕，所有的样板代码都需要在这里重复一遍！
    fn check_status(&self) -> Status {
        // ...
    }
}
````

## 示例

与其每次都输入所有这些样板代码来创建 `Connection` 和 `Request`，不如创建一个包装辅助函数，将它们作为参数传入：

````rust,ignore
struct Connection {
    name: String,
    stream: TcpStream,
}

impl Connection {
    /// 通过连接发送请求。
    ///
    /// # 示例
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
````

**注意**：在上面的示例中，`assert!(response.is_ok());` 这行代码实际上不会在测试时运行，因为它位于一个从未被调用的函数中。

## 优点

这种方式更加简洁，避免了示例中的重复代码。

## 缺点

由于示例代码位于函数中，这些代码不会被测试。不过在运行 `cargo test` 时仍会检查代码是否能够编译。因此这种模式在需要使用 `no_run` 时最为有用。使用这种方式，你就不需要添加 `no_run`。

## 讨论

如果不需要断言，这种模式就能很好地工作。

如果需要断言，一种替代方案是创建一个公共方法来创建辅助实例，并使用 `#[doc(hidden)]` 标注（这样用户就看不到它）。然后这个方法就可以在 rustdoc 中被调用，因为它是 crate 公共 API 的一部分。
