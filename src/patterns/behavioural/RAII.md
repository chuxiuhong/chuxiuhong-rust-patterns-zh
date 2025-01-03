# 基于守卫的 RAII 模式

## 描述

[RAII][wikipedia] 代表 "Resource Acquisition is Initialisation"（资源获取即初始化），这是一个不太恰当的命名。这个模式的核心思想是在对象的构造函数中进行资源初始化，在析构函数中进行资源释放。在 Rust 中，这个模式得到了扩展：通过使用 RAII 对象作为资源的守卫，并依靠类型系统来确保对资源的访问始终通过守卫对象进行中介。

## 示例

互斥锁守卫（Mutex guards）是标准库中这个模式的经典示例（以下是实际实现的简化版本）：

```rust,ignore
use std::ops::Deref;

struct Foo {}

struct Mutex<T> {
    // We keep a reference to our data: T here.
    //..
}

struct MutexGuard<'a, T: 'a> {
    data: &'a T,
    //..
}

// Locking the mutex is explicit.
impl<T> Mutex<T> {
    fn lock(&self) -> MutexGuard<T> {
        // Lock the underlying OS mutex.
        //..

        // MutexGuard keeps a reference to self
        MutexGuard {
            data: self,
            //..
        }
    }
}

// Destructor for unlocking the mutex.
impl<'a, T> Drop for MutexGuard<'a, T> {
    fn drop(&mut self) {
        // Unlock the underlying OS mutex.
        //..
    }
}

// Implementing Deref means we can treat MutexGuard like a pointer to T.
impl<'a, T> Deref for MutexGuard<'a, T> {
    type Target = T;

    fn deref(&self) -> &T {
        self.data
    }
}

fn baz(x: Mutex<Foo>) {
    let xx = x.lock();
    xx.foo(); // foo is a method on Foo.
              // The borrow checker ensures we can't store a reference to the underlying
              // Foo which will outlive the guard xx.

    // x is unlocked when we exit this function and xx's destructor is executed.
}
```

## 动机

当资源在使用后必须进行释放时，可以使用 RAII 来完成这个释放过程。如果在资源释放后访问该资源会导致错误，那么这个模式可以用来防止此类错误的发生。

## 优势

防止资源未被正确释放的错误，以及防止资源在释放后被使用的错误。

## 讨论

RAII 是一个用于确保资源正确释放或终结的有用模式。在 Rust 中，我们可以利用借用检查器来静态地防止资源释放后使用的错误。

借用检查器的核心目标是确保对数据的引用不会超过数据本身的生命周期。RAII 守卫模式之所以有效，是因为守卫对象包含对底层资源的引用，并且只暴露这些引用。Rust 确保守卫不会超过底层资源的生命周期，并且通过守卫获得的资源引用不会超过守卫的生命周期。要理解这一点，查看去除生命周期省略后的 `deref` 签名会很有帮助：

```rust,ignore
fn deref<'a>(&'a self) -> &'a T {
    //..
}
```

返回的资源引用与 `self` 具有相同的生命周期（`'a`）。因此，借用检查器确保对 `T` 的引用的生命周期短于 `self` 的生命周期。

注意，实现 `Deref` 并不是这个模式的核心部分，它只是使守卫对象的使用更加符合人体工程学。在守卫上实现 `get` 方法同样可以达到目的。

## 参见

[析构函数中的终结化习语](../../idioms/dtor-finally.md)

RAII 是 C++ 中的一个常见模式：
[cppreference.com](http://en.cppreference.com/w/cpp/language/raii),
[wikipedia][wikipedia]。

[wikipedia]: https://en.wikipedia.org/wiki/Resource_Acquisition_Is_Initialization

[样式指南条目](https://doc.rust-lang.org/1.0.0/style/ownership/raii.html)
（目前只是一个占位符）。
