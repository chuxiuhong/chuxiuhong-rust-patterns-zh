# 泛型作为类型类

## 说明

Rust的类型系统设计的更像函数式语言（比如Haskell），而非指令式语言如Java和C++。因此，Rust可以将许多编程问题转换成“静态类型”问题。这是选择函数式语言时最大的亮点之一，对于Rust的许多编译时保证来说是至关重要的。

这个概念的一个关键部分正是泛型的工作方式。在C++与Java中，举个例子，泛型是编译器的一种元编程结构。C++的`vector<int>`和`vector<char>`只是`vector`类型（叫`模板`）的同一模板代码的两个不同副本，其中填充了两种不同的类型。

在Rust中，泛型参数如同函数式语言中的“类型类约束”，而最终用户填写的每个不同的参数*实际上都会改变类型*。换句话说，`Vec<isize>`和`Vec<char>`*是两个不同的类型*，它们被类型系统识别为不同的类型。

这被称作**单态化**，不同类型以**多态**代码创建。这种特殊行为需要用`impl`块指定泛型参数：泛型的不同值会导致不同的类型，而不同的类型可以有不同的`impl`块。

在面向对象语言中，类可以从父类那里继承行为。实际上，这不仅允许将额外的行为附加到类型类的特定成员上，还允许附加额外的行为。

最接近的是Javascript和Python中的运行时多态性，新的成员可以被任何构造函数随意添加到对象中。然而，与这些语言不同，Rust的所有额外方法在使用时都可以进行类型检查，因为它们的泛型是静态定义的。这使得它们在保持安全的同时更具有实用性。

## 示例

想象你正在为实验室机器集群设计存储服务器。因为涉及的软件，有两个不同的协议需要你支持。BOOTP（用于PXE网络启动），和NFS（用于远程安装存储）。

你的目标是一个用Rust编写的程序，它可以处理这两种请求。它将有协议handler，监听两种请求。此外，主应用逻辑要允许实验室管理员配置实际文件的存储和安全控制。

不管来自什么协议，实验室机器对文件的请求都包含相同的基本信息：一个认证方法，和一个要检索的文件名。一个直接的实现会是这样的：

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

这种设计可能工作得很好。但现在，假设你需要支持添加*协议特定*的元数据。例如，对于NFS，你想确定他们的挂载点是什么，以便执行额外的安全规则。

当前结构的设计方式将协议的决定权留给了运行时。这也就是说，任何适用于一种协议而非另一种协议的方法都需要程序员进行运行时检查。

下面是获取NFS挂载点的情况：

```rust,ignore
struct FileDownloadRequest {
    file_name: PathBuf,
    authentication: AuthInfo,
    mount_point: Option<PathBuf>,
}

impl FileDownloadRequest {
    // ... 其他方法 ...

    /// 如果有NFS请求，获取一个NFS挂载点。
    /// 否则返回None。
    pub fn mount_point(&self) -> Option<&Path> {
        self.mount_point.as_ref()
    }
}
```

每个`mount_point()`的调用者都必须检查`None`并编写代码来处理它。就算他们知道，在一个给定的代码路径中只有NFS请求被使用。

如果不同的请求类型被弄混，引起编译时错误会理想。毕竟，用户的整个代码路径，包括他们使用的库中那些函数，都会知道一个请求是NFS请求还是BOOTP请求。

在Rust中，这是可能的！解决方案是*加个泛型*，分割API。

这样子：

```rust
use std::path::{Path, PathBuf};

mod nfs {
    #[derive(Clone)]
    pub(crate) struct AuthInfo(String); // NFS会话管理给省了
}

mod bootp {
    pub(crate) struct AuthInfo(); // bootp没验证机制
}

// private module, lest outside users invent their own protocol kinds!
mod proto_trait {
    use std::path::{Path, PathBuf};
    use super::{bootp, nfs};

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

    pub struct Bootp(); // 没有附加元数据

    impl ProtoKind for Bootp {
        type AuthInfo = bootp::AuthInfo;
        fn auth_info(&self) -> Self::AuthInfo {
            bootp::AuthInfo()
        }
    }
}

use proto_trait::ProtoKind; // 保持内部，以防止 impl
pub use proto_trait::{Nfs, Bootp}; // 重导出，这样调用者能看到它们

struct FileDownloadRequest<P: ProtoKind> {
    file_name: PathBuf,
    protocol: P,
}

// 把所有共同的API部分放进一个泛型实现块
impl<P: ProtoKind> FileDownloadRequest<P> {
    fn file_path(&self) -> &Path {
        &self.file_name
    }

    fn auth_info(&self) -> P::AuthInfo {
        self.protocol.auth_info()
    }
}

// all protocol-specific impls go into their own block
impl FileDownloadRequest<Nfs> {
    fn mount_point(&self) -> &Path {
        self.protocol.mount_point()
    }
}

fn main() {
    // 你代码扔这儿
}
```

对于这个方法，如果用户搞错了，使用了错误的类型：

```rust,ignore
fn main() {
    let mut socket = crate::bootp::listen()?;
    while let Some(request) = socket.next_request()? {
        match request.mount_point().as_ref()
            "/secure" => socket.send("Access denied"),
            _ => {} // 继续下去...
        }
        // 剩余代码部分放这里
    }
}
```

会得到一个类型错误。类型`FileDownloadRequest<Bootp>`没实现`mount_point()`，只有类型`FileDownloadRequest<Nfs>`实现了。而且说到底，那是NFS模块创建的，不是BOOTP！

## 优点

首先，它可以去重多个状态下共有的字段。通过使非共享字段成为泛型字段，它们只需要实现一次。

其次，它使`impl`块更容易阅读，因为它们是按状态分解的。所有状态下通用的方法都在一个块中输入一次，而某个状态下特有的方法则在一个单独的块中。

这两种情况都意味着代码行数更少，而且更有条理。

## 缺点

目前这将增加二进制文件大小，这是编译器实现单态化的方式造成的。希望这种实现方式在未来能够得到改善。

## 替代

- 如果一个类型由于构造或部分初始化，似乎需要一个 “切分的API”，可以考虑用[Builder模式](../patterns/creational/builder.md)代替。

- 如果类型之间的API不发生变化，只有行为发生变化，那么最好使用[策略](../patterns/behavioural/strategy.md)来代替。

## 参见

这种模式在整个标准库中都有应用。

- `Vec<u8>` can be cast from a String, unlike every other type of `Vec<T>`.[^1]
- They can also be cast into a binary heap, but only if they contain a type that implements the `Ord` trait.[^2]
- The `to_string` method was specialized for `Cow` only of type `str`.[^3]

它也被一些流行的crate使用，用以改进API灵活性：

- The `embedded-hal` ecosystem used for embedded devices makes extensive use of this pattern. For example, it allows statically verifying the configuration of device registers used to control embedded pins. When a pin is put into a mode, it returns a `Pin<MODE>` struct, whose generic determines the functions usable in that mode, which are not on the `Pin` itself. [^4](Example:)

- `hyper` HTTP客户端库用它为不同可插拔请求导出富API。Clients with different connectors have different methods on them as well as different trait implementations, while a core set of methods apply to any connector. [^5](See:)

- The "type state" pattern -- where an object gains and loses API based on an internal state or invariant -- is implemented in Rust using the same basic concept, and a slightly different technique. [^6](See:)

[^1]: 见[impl From&lt;CString&gt; for Vec&lt;u8&gt;](https://doc.rust-lang.org/stable/src/std/ffi/c_str.rs.html#799-801)

[^2]: 见[impl&lt;T&gt; From&lt;Vec&lt;T, Global&gt;&gt; for BinaryHeap&lt;T&gt;](https://doc.rust-lang.org/stable/src/alloc/collections/binary_heap.rs.html#1345-1354)

[^3]: 见[impl&lt;'_&gt; ToString for Cow&lt;'_, str&gt;](https://doc.rust-lang.org/stable/src/alloc/string.rs.html#2235-2240)

[https://docs.rs/stm32f30x-hal/0.1.0/stm32f30x_hal/gpio/gpioa/struct.PA0.html](https://docs.rs/stm32f30x-hal/0.1.0/stm32f30x_hal/gpio/gpioa/struct.PA0.html)

[https://docs.rs/hyper/0.14.5/hyper/client/struct.Client.html](https://docs.rs/hyper/0.14.5/hyper/client/struct.Client.html)

[The Case for the Type State Pattern](https://web.archive.org/web/20210325065112/https://www.novatec-gmbh.de/en/blog/the-case-for-the-typestate-pattern-the-typestate-pattern-itself/) and [Rusty Typestate Series (an extensive thesis)](https://web.archive.org/web/20210328164854/https://rustype.github.io/notes/notes/rust-typestate-series/rust-typestate-index)
