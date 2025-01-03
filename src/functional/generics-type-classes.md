# 泛型作为类型类

## 描述

Rust 的类型系统设计更接近函数式语言（如 Haskell），而不是命令式语言（如 Java 和 C++）。因此，Rust 可以将许多编程问题转化为"静态类型"问题。这是选择函数式语言的最大优势之一，也是 Rust 在编译时提供许多保证的关键。

这个理念的一个关键部分是泛型类型的工作方式。例如，在 C++ 和 Java 中，泛型类型是编译器的元编程构造。C++ 中的 `vector<int>` 和 `vector<char>` 只是同一个 `vector` 类型（称为 `template`）的两个不同副本，填入了两个不同的类型。

在 Rust 中，泛型类型参数创建了在函数式语言中被称为"类型类约束"的东西，最终用户填入的每个不同参数*实际上会改变类型*。换句话说，`Vec<isize>` 和 `Vec<char>` *是两个不同的类型*，类型系统的所有部分都将它们识别为不同的类型。

这被称为**单态化**，即从**多态**代码创建不同的类型。这种特殊行为要求 `impl` 块指定泛型参数。泛型类型的不同值会导致不同的类型，而不同的类型可以有不同的 `impl` 块。

在面向对象语言中，类可以从其父类继承行为。然而，这不仅允许为类型类的特定成员附加额外的行为，还允许添加额外的行为。

最接近的等价物是 Javascript 和 Python 中的运行时多态，其中任何构造函数都可以随意向对象添加新成员。然而，与这些语言不同，Rust 的所有额外方法在使用时都可以进行类型检查，因为它们的泛型是静态定义的。这使得它们在保持安全的同时更加可用。

## 示例

假设你正在为一系列实验室机器设计存储服务器。由于涉及的软件，你需要支持两种不同的协议：BOOTP（用于 PXE 网络启动）和 NFS（用于远程挂载存储）。

你的目标是用 Rust 编写一个程序来处理这两种协议。它将有协议处理程序并监听两种类型的请求。主应用程序逻辑将允许实验室管理员为实际文件配置存储和安全控制。

实验室中机器请求文件时包含的基本信息是相同的，无论它们来自哪个协议：一个认证方法和要检索的文件名。一个直接的实现可能如下所示：

```rust,ignore
enum AuthInfo {
    Nfs(crate::nfs::AuthInfo),
    Bootp(crate::bootp::AuthInfo),
}

struct FileDownloadRequest {
    file_name: PathBuf,
    authentication: AuthInfo,
}
```

这个设计可能足够好用。但现在假设你需要支持添加*特定于协议*的元数据。例如，对于 NFS，你想要确定它们的挂载点以便执行额外的安全规则。

当前结构体的设计将协议决策留到运行时。这意味着任何仅适用于一个协议而不适用于另一个协议的方法都需要程序员进行运行时检查。

获取 NFS 挂载点的方式如下：

```rust,ignore
struct FileDownloadRequest {
    file_name: PathBuf,
    authentication: AuthInfo,
    mount_point: Option<PathBuf>,
}

impl FileDownloadRequest {
    // ... 其他方法 ...

    /// 如果这是 NFS 请求，则获取 NFS 挂载点。否则，
    /// 返回 None。
    pub fn mount_point(&self) -> Option<&Path> {
        self.mount_point.as_ref()
    }
}
```

`mount_point()` 的每个调用者都必须检查 `None` 并编写代码来处理它。即使他们知道在给定的代码路径中只使用 NFS 请求，这也是必要的！

如果在混淆不同请求类型时导致编译时错误，那将是更优的。毕竟，用户代码的整个路径，包括他们使用的库函数，都会知道请求是 NFS 请求还是 BOOTP 请求。

在 Rust 中，这实际上是可能的！解决方案是*添加泛型类型*来分割 API。

这是它的样子：

```rust
use std::path::{Path, PathBuf};

mod nfs {
    #[derive(Clone)]
    pub(crate) struct AuthInfo(String); // 省略 NFS 会话管理
}

mod bootp {
    pub(crate) struct AuthInfo(); // bootp 中没有认证
}

// 私有模块，以防外部用户发明自己的协议类型！
mod proto_trait {
    use super::{bootp, nfs};
    use std::path::{Path, PathBuf};

    pub(crate) trait ProtoKind {
        type AuthInfo;
        fn auth_info(&self) -> Self::AuthInfo;
    }

    pub struct Nfs {
        auth: nfs::AuthInfo,
        mount_point: PathBuf,
    }

    impl Nfs {
        pub(crate) fn mount_point(&self) -> &Path {
            &self.mount_point
        }
    }

    impl ProtoKind for Nfs {
        type AuthInfo = nfs::AuthInfo;
        fn auth_info(&self) -> Self::AuthInfo {
            self.auth.clone()
        }
    }

    pub struct Bootp(); // 没有额外的元数据

    impl ProtoKind for Bootp {
        type AuthInfo = bootp::AuthInfo;
        fn auth_info(&self) -> Self::AuthInfo {
            bootp::AuthInfo()
        }
    }
}

use proto_trait::ProtoKind; // 保持内部以防止实现
pub use proto_trait::{Bootp, Nfs}; // 重新导出以便调用者可以看到它们

struct FileDownloadRequest<P: ProtoKind> {
    file_name: PathBuf,
    protocol: P,
}

// 所有通用 API 部分都放在泛型 impl 块中
impl<P: ProtoKind> FileDownloadRequest<P> {
    fn file_path(&self) -> &Path {
        &self.file_name
    }

    fn auth_info(&self) -> P::AuthInfo {
        self.protocol.auth_info()
    }
}

// 所有特定于协议的实现都放在它们自己的块中
impl FileDownloadRequest<Nfs> {
    fn mount_point(&self) -> &Path {
        self.protocol.mount_point()
    }
}

fn main() {
    // 你的代码在这里
}
```

使用这种方法，如果用户犯了错误并使用了错误的类型：

```rust,ignore
fn main() {
    let mut socket = crate::bootp::listen()?;
    while let Some(request) = socket.next_request()? {
        match request.mount_point().as_ref() {
            "/secure" => socket.send("Access denied"),
            _ => {} // 继续...
        }
        // 其余代码在这里
    }
}
```

他们会得到一个语法错误。类型 `FileDownloadRequest<Bootp>` 没有实现 `mount_point()`，只有类型 `FileDownloadRequest<Nfs>` 实现了它。当然，这是由 NFS 模块创建的，而不是 BOOTP 模块！

## 优点

首先，它允许多个状态共有的字段被去重。通过使非共享字段泛型化，它们只需实现一次。

其次，它使 `impl` 块更容易阅读，因为它们按状态分解。所有状态共有的方法在一个块中只输入一次，而特定于一个状态的方法在一个单独的块中。

这两点都意味着代码行数更少，而且组织得更好。

## 缺点

由于编译器中单态化的实现方式，这目前会增加二进制文件的大小。希望将来实现能够改进。

## 替代方案

- 如果一个类型似乎需要由于构造或部分初始化而"分割 API"，请考虑使用[构建器模式](../patterns/creational/builder.md)。

- 如果类型之间的 API 没有变化 -- 只有行为发生变化 -- 那么最好使用[策略模式](../patterns/behavioural/strategy.md)。

## 另请参阅

这种模式在标准库中被广泛使用：

- `Vec<u8>` 可以从 String 转换，而其他类型的 `Vec<T>` 则不行。[^1]
- 它们也可以转换为二进制堆，但只有在它们包含实现了 `Ord` trait 的类型时才可以。[^2]
- `to_string` 方法仅针对 `str` 类型的 `Cow` 进行了特化。[^3]

一些流行的 crate 也使用它来提供 API 灵活性：

- 用于嵌入式设备的 `embedded-hal` 生态系统广泛使用这种模式。例如，它允许静态验证用于控制嵌入式引脚的设备寄存器的配置。当一个引脚被置于某个模式时，它返回一个 `Pin<MODE>` 结构体，其泛型决定了在该模式下可用的函数，这些函数不在 `Pin` 本身上。[^4]

- `hyper` HTTP 客户端库使用这种模式来为不同的可插拔请求公开丰富的 API。具有不同连接器的客户端有不同的方法和不同的 trait 实现，而核心方法集适用于任何连接器。[^5]

- "类型状态"模式 -- 对象基于内部状态或不变量获得和失去 API -- 在 Rust 中使用相同的基本概念和略微不同的技术来实现。[^6]

[^1]: 参见：
[impl From\<CString\> for Vec\<u8\>](https://doc.rust-lang.org/1.59.0/src/std/ffi/c_str.rs.html#803-811)

[^2]: 参见：
[impl\<T: Ord\> FromIterator\<T\> for BinaryHeap\<T\>](https://web.archive.org/web/20201030132806/https://doc.rust-lang.org/stable/src/alloc/collections/binary_heap.rs.html#1330-1335)

[^3]: 参见：
[impl\<'\_\> ToString for Cow\<'\_, str>](https://doc.rust-lang.org/stable/src/alloc/string.rs.html#2235-2240)

[^4]: 示例：
[https://docs.rs/stm32f30x-hal/0.1.0/stm32f30x_hal/gpio/gpioa/struct.PA0.html](https://docs.rs/stm32f30x-hal/0.1.0/stm32f30x_hal/gpio/gpioa/struct.PA0.html)

[^5]: 参见：
[https://docs.rs/hyper/0.14.5/hyper/client/struct.Client.html](https://docs.rs/hyper/0.14.5/hyper/client/struct.Client.html)

[^6]: 参见：
[类型状态模式的案例](https://web.archive.org/web/20210325065112/https://www.novatec-gmbh.de/en/blog/the-case-for-the-typestate-pattern-the-typestate-pattern-itself/)
和
[Rusty Typestate 系列（一个广泛的论文）](https://web.archive.org/web/20210328164854/https://rustype.github.io/notes/notes/rust-typestate-series/rust-typestate-index)
