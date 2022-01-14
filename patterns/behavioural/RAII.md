# RAII 守卫

## 说明

[RAII](https://zh.wikipedia.org/wiki/RAII)是个糟糕的名字，代表“资源获取即初始化”。该模式的本质是，资源的初始化在对象的构造函数中完成，以及确定性析构器。通过使用一个RAII对象作为一些资源的守卫，并且依赖类型系统确保访问始终要通过守卫对象，以此在Rust中扩展这种模式。

## 代码示例

互斥保护是std库中这种模式的经典示例（这是实际实现中的简化版本）：

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

## 出发点

当资源被使用后必须被销毁，RAII可以被用来实现确定性析构。如果在销毁后访问该资源是错误的，那么此模式可用于防止此类错误。

## 优点

防止使用未初始化资源和销毁后资源的错误。

## 讨论

RAII是确保资源被合适地析构或确定的实用模式。我们可以在Rust中使用借用检查器静态地防止析构后发生使用资源的错误。

借用检查器的核心目标是确保对数据的引用不能超过数据的生命周期。RAII守卫模式之所以有效，是因为守卫对象包含对底层资源的引用并且只暴露这样的引用。Rust确保了守卫不能比底层资源活的更长，并且由守卫控制的对资源的引用不能比守卫获得更长。要了解这是如何工作的，最好检查`deref`的签名不进行生命周期省略。

```rust,ignore
fn deref<'a>(&'a self) -> &'a T {
    //..
}
```

返回的资源引用有与`self`相同的生命周期(`'a'`)。借用检查器因此确保`T`的引用比`self`的声明周期要短。

注意实现`Deref`不是这个模式的核心部分，这只是为了在用守卫时更加符合人体工程学。对守卫实现一个`get`方法也一样可以。

## 参阅

[Finalisation in destructors idiom](../../idioms/dtor-finally.md)

RAII is a common pattern in C++: [cppreference.com](http://en.cppreference.com/w/cpp/language/raii),
[wikipedia][wikipedia].

[wikipedia]: https://en.wikipedia.org/wiki/Resource_Acquisition_Is_Initialization

[Style guide entry](https://doc.rust-lang.org/1.0.0/style/ownership/raii.html)
(currently just a placeholder).
