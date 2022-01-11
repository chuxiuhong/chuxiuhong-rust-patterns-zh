# 栈上动态分发

## 说明

我们可以动态分发多个值，然而为了实现此功能，需要声明多个变量来绑定不同类型的对象。我们可以使用延迟条件初始化（deferred conditional initialization）来扩展生命周期，如下所示：

## 例子

```rust
use std::io;
use std::fs;

# fn main() -> Result<(), Box<dyn std::error::Error>> {
# let arg = "-";

// 它们必须活的比 `readable`长, 因此先声明:
let (mut stdin_read, mut file_read);

// We need to ascribe the type to get dynamic dispatch.
let readable: &mut dyn io::Read = if arg == "-" {
    stdin_read = io::stdin();
    &mut stdin_read
} else {
    file_read = fs::File::open(arg)?;
    &mut file_read
};

// Read from `readable` here.

# Ok(())
# }
```

## 出发点

Rust默认是单态的代码。这就意味着对每个类型都要生成相对应的代码并且单独优化。这种模式虽然在热路径（hot path）上执行的很快，但是它空间上将非常臃肿。当性能不是致命关键的时候，我们还是要考虑考虑编译时间和cache的使用。

幸运的是，Rust允许我们使用动态分发，但是我们需要显式的声明。

## 优点

我们不用在堆上申请任何空间。既不用初始化任何用不上的东西，也不用单态化全部代码，便可同时支持`File`和`Stdin`。

## 缺点

这样写代码比使用`Box`实现的版本需要更多活动部件（moving parts）：

```rust,ignore
// We still need to ascribe the type for dynamic dispatch.
let readable: Box<dyn io::Read> = if arg == "-" {
    Box::new(io::stdin())
} else {
    Box::new(fs::File::open(arg)?)
};
// Read from `readable` here.
```

## 讨论

初学Rust之人通常会学到Rust需要所有变量在使用前需要初始化，所以常会忽略没有用到的变量可能不会初始化的问题。Rust付出大量工作来确保只有初始化过的值在离开作用域时会销毁。

上面这个例子符合我们所有的限制条件：

* 所有的变量都在使用前初始化（这个例子中是借用）
* 每个变量都只有单一类型。在我们的例子中，`stdin`对应`Stdin`类型，`file`对应`File`类型，`readable`对应`&mut dyn Read`类型
* 每个借用的值的生命周期都比借用他们的场。

## 参阅

* [Finalisation in destructors](dtor-finally.md) and
[RAII guards](../patterns/behavioural/RAII.md) can benefit from tight control over lifetimes.
* For conditionally filled `Option<&T>`s of (mutable) references, one can
initialize an `Option<T>` directly and use its [`.as_ref()`] method to get an
optional reference.

[`.as_ref()`]: https://doc.rust-lang.org/std/option/enum.Option.html#method.as_ref
