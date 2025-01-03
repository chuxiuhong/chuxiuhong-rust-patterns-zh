# 基于对象的 API

## 描述

在设计需要暴露给其他语言的 Rust API 时，有一些重要的设计原则与普通的 Rust API 设计相反：

1. 所有封装类型应当由 Rust *拥有*，由用户*管理*，并且保持*不透明*。
2. 所有事务性数据类型应当由用户*拥有*，并且保持*透明*。
3. 所有库的行为应当是作用于封装类型的函数。
4. 所有库的行为应当被封装到类型中，这种封装不是基于结构，而是基于*来源/生命周期*。

## 动机

Rust 内置了对其他语言的 FFI 支持。它通过允许 crate 作者提供基于不同 ABI 的 C 兼容 API 来实现这一点（尽管这对本实践来说并不重要）。

设计良好的 Rust FFI 遵循 C API 的设计原则，同时尽可能少地在 Rust 端做出妥协。任何外部 API 都有三个目标：

1. 使其在目标语言中易于使用。
2. 尽可能避免 API 在 Rust 端强制要求内部不安全代码。
3. 将潜在的内存不安全和 Rust 的 `undefined behaviour` 的可能性降到最低。

Rust 代码必须在某种程度上信任外部语言的内存安全性。然而，Rust 端的每一段 `unsafe` 代码都是潜在的 bug 源，或者可能加剧 `undefined behaviour`。

例如，如果指针来源错误，可能会因为无效的内存访问而导致段错误。但如果它被不安全代码操作，可能会导致完全的堆损坏。

基于对象的 API 设计允许编写具有良好内存安全特性的垫片代码，并在安全和 `unsafe` 之间建立清晰的边界。

## 代码示例

POSIX 标准定义了访问文件数据库的 API，称为 [DBM](https://web.archive.org/web/20210105035602/https://www.mankier.com/0p/ndbm.h)。这是一个"基于对象"API 的优秀示例。

以下是 C 语言中的定义，对于那些涉及 FFI 的人来说应该很容易理解。下面的注释将帮助解释其中的细节。

```C
struct DBM;
typedef struct { void *dptr, size_t dsize } datum;

int     dbm_clearerr(DBM *);
void    dbm_close(DBM *);
int     dbm_delete(DBM *, datum);
int     dbm_error(DBM *);
datum   dbm_fetch(DBM *, datum);
datum   dbm_firstkey(DBM *);
datum   dbm_nextkey(DBM *);
DBM    *dbm_open(const char *, int, mode_t);
int     dbm_store(DBM *, datum, datum, int);
```

这个 API 定义了两种类型：`DBM` 和 `datum`。

`DBM` 类型在上文中被称为"封装"类型。它被设计用来包含内部状态，并作为库行为的入口点。

它对用户来说是完全不透明的，用户无法自己创建 `DBM`，因为他们不知道其大小和布局。相反，他们必须调用 `dbm_open`，而且只能获得*一个指向它的指针*。

这意味着所有的 `DBM` 在 Rust 的意义上都是由库"拥有"的。未知大小的内部状态保存在库控制的内存中，而不是用户控制的内存中。用户只能通过 `open` 和 `close` 管理其生命周期，并通过其他函数对其执行操作。

`datum` 类型在上文中被称为"事务性"类型。它被设计用来促进库和用户之间的信息交换。

数据库被设计为存储"非结构化数据"，没有预定义的长度或含义。因此，`datum` 相当于 Rust 中的切片：一堆字节和一个计数它们数量的值。主要区别在于没有类型信息，这就是 `void` 表示的含义。

请记住，这个头文件是从库的角度编写的。用户可能有某种他们正在使用的类型，这个类型有已知的大小。但库并不关心这一点，根据 C 语言的类型转换规则，任何类型背后的指针都可以转换为 `void`。

如前所述，这个类型对用户来说是*透明的*。但同时，这个类型也是由用户*拥有*的。这有一些微妙的影响，因为其中包含指针。问题是，谁拥有这个指针指向的内存？

从内存安全的角度来看，最好的答案是"用户"。但在某些情况下，比如检索值时，用户并不知道如何正确分配内存（因为他们不知道值的长度）。在这种情况下，库代码应该使用用户可以访问的堆（比如 C 库的 `malloc` 和 `free`），然后在 Rust 的意义上*转移所有权*。

这可能看起来很理论化，但这就是指针在 C 中的含义。它与 Rust 的含义相同："用户定义的生命周期"。库的用户需要阅读文档才能正确使用它。也就是说，有些决定如果用户做错了会带来或大或小的后果。最小化这些后果就是这个最佳实践的目的，关键是*转移所有透明内容的所有权*。

## 优点

这将用户必须遵守的内存安全保证最小化到相对较少的几点：

1. 不要使用非 `dbm_open` 返回的指针调用任何函数（无效访问或损坏）。
2. 不要在关闭后对指针调用任何函数（释放后使用）。
3. 任何 `datum` 的 `dptr` 必须是 `NULL`，或者指向具有声明长度的有效内存切片。

此外，它避免了许多指针来源问题。为了理解原因，让我们深入考虑一个替代方案：键的迭代。

Rust 以其迭代器而闻名。在实现迭代器时，程序员创建一个与其所有者具有有界生命周期的单独类型，并实现 `Iterator` trait。

以下是如何在 Rust 中为 `DBM` 实现迭代：

```rust,ignore
struct Dbm { ... }

impl Dbm {
    /* ... */
    pub fn keys<'it>(&'it self) -> DbmKeysIter<'it> { ... }
    /* ... */
}

struct DbmKeysIter<'it> {
    owner: &'it Dbm,
}

impl<'it> Iterator for DbmKeysIter<'it> { ... }
```

这是干净、惯用且安全的，这要归功于 Rust 的保证。然而，考虑一下直接的 API 转换会是什么样子：

```rust,ignore
#[no_mangle]
pub extern "C" fn dbm_iter_new(owner: *const Dbm) -> *mut DbmKeysIter {
    // 这是一个糟糕的 API 设计！在实际应用中，应该使用基于对象的设计
}
#[no_mangle]
pub extern "C" fn dbm_iter_next(
    iter: *mut DbmKeysIter,
    key_out: *const datum
) -> libc::c_int {
    // 这是一个糟糕的 API 设计！在实际应用中，应该使用基于对象的设计
}
#[no_mangle]
pub extern "C" fn dbm_iter_del(*mut DbmKeysIter) {
    // 这是一个糟糕的 API 设计！在实际应用中，应该使用基于对象的设计
}
```

这个 API 丢失了一个关键信息：迭代器的生命周期不能超过拥有它的 `Dbm` 对象的生命周期。库的用户可能会以导致迭代器超过其正在迭代的数据生命周期的方式使用它，从而导致读取未初始化的内存。

下面这个用 C 编写的示例包含一个 bug，稍后会解释：

```C
int count_key_sizes(DBM *db) {
    // 不要使用这个函数。它有一个微妙但严重的 bug！
    datum key;
    int len = 0;

    if (!dbm_iter_new(db)) {
        dbm_close(db);
        return -1;
    }

    int l;
    while ((l = dbm_iter_next(owner, &key)) >= 0) { // 错误用 -1 表示
        free(key.dptr);
        len += key.dsize;
        if (l == 0) { // 迭代器结束
            dbm_close(owner);
        }
    }
    if l >= 0 {
        return -1;
    } else {
        return len;
    }
}
```

这是一个经典的 bug。当迭代器返回结束标记时，会发生以下情况：

1. 循环条件将 `l` 设置为零，并因为 `0 >= 0` 而进入循环。
2. 长度增加，在这种情况下增加零。
3. if 语句为真，所以数据库被关闭。这里应该有一个 break 语句。
4. 循环条件再次执行，导致在已关闭的对象上调用 `next`。

这个 bug 最糟糕的地方是什么？如果 Rust 实现很小心，这段代码大多数时候都能工作！如果 `Dbm` 对象的内存没有立即被重用，内部检查几乎肯定会失败，导致迭代器返回表示错误的 `-1`。但偶尔会导致段错误，或者更糟糕的是，产生无意义的内存损坏！

这些都无法通过 Rust 来避免。从 Rust 的角度来看，它把这些对象放在堆上，返回指向它们的指针，并放弃对它们生命周期的控制。C 代码必须"友好相处"。

程序员必须阅读并理解 API 文档。虽然有些人认为这在 C 中是理所当然的，但一个好的 API 设计可以降低这种风险。POSIX 的 `DBM` API 通过*合并迭代器与其父对象的所有权*来实现这一点：

```C
datum   dbm_firstkey(DBM *);
datum   dbm_nextkey(DBM *);
```

因此，所有的生命周期都被绑定在一起，这样的不安全性就被防止了。

## 缺点

然而，这种设计选择也有一些缺点，这些也应该被考虑。

首先，API 本身变得不那么富有表现力。在 POSIX DBM 中，每个对象只有一个迭代器，每次调用都会改变其状态。这比几乎任何语言中的迭代器都要限制得多，尽管它是安全的。也许对于其他相关对象，如果它们的生命周期不那么层次化，这种限制的代价可能比安全性更高。

其次，根据 API 各部分的关系，可能需要大量的设计工作。许多较简单的设计点都有其他相关的模式：

- [Wrapper Type Consolidation](./wrappers.md) 将多个 Rust 类型组合成一个不透明的"对象"

- [FFI Error Passing](../../idioms/ffi/errors.md) 解释了使用整数代码和哨兵返回值（如 `NULL` 指针）的错误处理

- [Accepting Foreign Strings](../../idioms/ffi/accepting-strings.md) 允许以最少的不安全代码接受字符串，这比 [Passing Strings to FFI](../../idioms/ffi/passing-strings.md) 更容易做对

然而，并非每个 API 都可以这样做。这取决于程序员对其受众的最佳判断。
