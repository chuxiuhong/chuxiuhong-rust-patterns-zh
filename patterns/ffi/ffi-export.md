# 基于对象的API

## 说明

当在Rust中设计暴露给其他语言的接口时，有一些与普通的API设计原则相反的重要原则。

1. 所有封装类型的所有权应该在Rust一端，由用户管理，并且不对外透明。
2. 所有用来交换的数据类型应该由用户所有，并且对外透明。
3. 库的操作应该是针对封装类型的函数。
4. 所有操作不应该封装成基于结构体的类型，而是*出处/生命周期*。

## 出发点

Rust有内置的FFI与其他语言交互。这种方式为库作者通过不同的ABI提供了兼容C的API方法。（尽管这和我们的做法无关）

设计良好的Rust的FFI遵循C语言API的设计原则，同时尽量减少Rust的设计。下面有三个和任何外部语言API设计的目标：

1. 让使用目标语言更简单。
2. 尽量避免API破坏Rust端的内部安全性。
3. 尽量使内存不安全的部分和Rust的未定义行为的部分越少越好。

Rust代码必须在与外部语言交互的某个层面之上保持安全。然而，`unsafe`代码中的每个比特都可能造成bug，或者导致未定义行为。

例如，如果一个指针是错误的，将会导致非法内存访问的错误。但是它如果是任由非安全代码执行的，它将会使堆内存彻底崩溃。

基于对象的API设计设计允许写一些接口代码，来清晰明了地划分`safe`和`unsafe`代码间的边界，同时保持良好的内存安全特性。

## 代码示例

POSIX标准定义了访问基于文件的数据库的API，如[DBM](https://web.archive.org/web/20210105035602/https://www.mankier.com/0p/ndbm.h)

以下是一个基于对象的API的绝好示例。

这是一段很容易阅读的涉及FFI的C语言代码。下面的说明将助你把握微妙之处。

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

这个API定义了两种类型：`DBM`和`datum`。

`DBM`类型被一个封装类型调用。它包含内部状态并且作为库操作的接入点。

由于不知道`DBM`类型的大小和内存结构，所以它对用户完全不透明，无法创建这种对象。取而代之的是必须通过调用`dbm_open`方法，仅会给其中一方一个指针。

这意味着所有的`DBM`对象被库所有。库掌握其内部内存，而不是用户。用户仅通过`open`和`close`来掌控对象的生命周期，以及用其他函数来执行操作。

`datum`类型在前文中被称为用来交换的数据类型。它是用来在用户和库之间传递信息的数据类型。

数据库是用来存储非结构数据的，没有预先定义的长度或意义。作为结果，`datum`是C中等价于Rust中的切片的类型：一大块字节空间和长度。最大的区别是这里没有类型信息，只有`void`指针表示。

记住这个头文件是从库的视角来写的。用户有一些自己知道尺寸的类型。但是库并不关心这一点，而且由于C的类型强制转换，任何类型的指针都可以被转换为`void`。

如前所述，这种类型对用户是*透明的*。而且这个类型归用户所有。因为里面有指针，所以有些微妙的影响。问题是，谁拥有这个指针指向的数据？

对于最佳的内存安全性来说，答案是用户。但是实际取回一个值时，用户并不知道如何申请内存（因为并不知道值有多长）。库代码将会使用用户访问的堆空间，例如C语言中的`malloc`和`free`函数，然后将所有权传给Rust一端。

这看起来都是推测，但实际上C语言中的指针就是这样。在Rust中相当于“用户定义生命周期”。库的用户需要阅读文档来正确使用它。用户需要阅读文档才能正确使用它。也就是说用户做错某些决定，后果无法确定。使出现这种情况最少的关键点是把透明的对象的所有权交出去。

## 优点

这样可以让用户为内存安全保证所付出的努力最小化：

1. 不要在调用函数的时候使用不是由`dbm_open`返回的指针（将造成非法访问）
2. 不要调用函数的时候使用已经关闭的指针（释放后再使用）
3. 任何`datum`的`dptr`必须是空指针或者指向一片合法的内存区域。

此外，这也避免了一系列指针错误问题。为了理解原因，让我们深入考虑另一种情况：键值循环（key iteration）。

Rust的迭代器很有名。当实现一个迭代器时，开发者创造了一个生命周期受所有者限制的独立类型，并且实现`Iterator`特性。

下面是在Rust中如何为`DBM`实现迭代器的方法：

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

托Rust的福，这样实现干净、符合习惯并且安全。

不过，考虑将API直译过来的情况如下：

```rust,ignore
#[no_mangle]
pub extern "C" fn dbm_iter_new(owner: *const Dbm) -> *mut DbmKeysIter {
    // THIS API IS A BAD IDEA! For real applications, use object-based design instead.
}
#[no_mangle]
pub extern "C" fn dbm_iter_next(
    iter: *mut DbmKeysIter,
    key_out: *const datum
) -> libc::c_int {
    // THIS API IS A BAD IDEA! For real applications, use object-based design instead.
}
#[no_mangle]
pub extern "C" fn dbm_iter_del(*mut DbmKeysIter) {
    // THIS API IS A BAD IDEA! For real applications, use object-based design instead.
}
```

这样的API丢失了一个重要信息：迭代器的生命周期不能长于`Dbm`对象的生命周期。库的用户将会在某些情况下通过迭代器访问到已经释放的数据，导致读取未初始化内存的错误。

下面用C语言写的例子包含了一个bug，以下将详细说明

```C
int count_key_sizes(DBM *db) {
    // DO NOT USE THIS FUNCTION. IT HAS A SUBTLE BUT SERIOUS BUG!
    datum key;
    int len = 0;

    if (!dbm_iter_new(db)) {
        dbm_close(db);
        return -1;
    }

    int l;
    while ((l = dbm_iter_next(owner, &key)) >= 0) { // an error is indicated by -1
        free(key.dptr);
        len += key.dsize;
        if (l == 0) { // end of the iterator
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

这个bug是经典bug。当迭代器返回结束循环的标志时将发生：

1. 循环条件设置`l`为0，然后因为`0 >= 0`进入循环。
2. 长度是递增的，初始化是0。
3. if条件是true，所以数据库被关闭。这应该有一个break。
4. 循环条件再次执行，导致`next`访问已经被关闭的对象。

这个bug里最坏的部分是什么？如果Rust实现部分比较小心，这段代码在大多数情况下可以使用！如果`Dbm`对象的内存没有立刻被重用，内部检查将总是失败，导致迭代器返回-1表示错误。但是其将会偶尔地导致段错误，或者更坏，更离谱的内存错误！

这种问题不是单靠Rust所能避免的。从库的角度来看，它将对象放在堆上，返回指向这些对象的指针，然后放弃对生命周期的控制。C语言的部分必须“做的漂亮点”。

开发者必须阅读和理解API文档。虽然有些人认为C语言出现这些问题是意料之中，但是通过一个好的API设计是可以减轻这种风险的。`DBM`的POSIX标准API是将所有权合并到其根节点来实现的：

```C
datum   dbm_firstkey(DBM *);
datum   dbm_nextkey(DBM *);
```

像这样，所有的生命周期都被绑在一块了，因此避免了风险。

## 缺点

不过，这样的设计也有一些也需要考虑到的缺点。

首先，API本身的表达力变得更差了。用POSIX标准的DBM，每个对象只有一个迭代器，并且每次调用改变自身状态。尽管它是安全的，但这比几乎任何语言中的迭代器都要严格得多。或许对于其他相关对象，它们的生命周期没有那么多层次，这时这种限制的成本比安全性收益要更大。

其次，根据API各部分之间的关系，可能会涉及大量的设计工作。许多更简单的设计点都有与之相关的设计模式：

- [类型合并封装](ffi-wrappers.md) 打包多个Rust类型为一个不透明的对象
  
- [常见错误处理](../../idioms/ffi/ffi-errors.md) 讲述使用整型作为错误代码和返回值的哨兵（就像`NULL`指针一样）
  
- [接受字符串](../../idioms/ffi/ffi-accepting-strings.md) 代码的情况下接受字符串，并且更容易成功[传递字符串](../../idioms/ffi/ffi-passing-strings.md)

不过，也不是所有API都可以这样设计。具体情况具体分析。
