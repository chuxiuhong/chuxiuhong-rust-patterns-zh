# 分解结构体



## 说明

有时候一个很大的结构体会在借用的时候产生问题——当有多个可变借用（每个只改变其中一部分字段）的时候会相互冲突。解决方法是将这个大结构体分解成更小的结构体，然后再把这些小结构组装成大结构体，这样结构体中的每个部分都可以单独的借用。

这通常在其他方面带来更好的设计：用这种模式可以展露出更小的功能模块。

## 示例

下面是一个设计出的借用检查器会阻止我们使用结构体的示例：

```rust
struct A {
    f1: u32,
    f2: u32,
    f3: u32,
}

fn foo(a: &mut A) -> &u32 { &a.f2 }
fn bar(a: &mut A) -> u32 { a.f1 + a.f3 }

fn baz(a: &mut A) {
    // The later usage of x causes a to be borrowed for the rest of the function.
    let x = foo(a);
    // Borrow checker error:
    // let y = bar(a); // ~ ERROR: cannot borrow `*a` as mutable more than once
                       //          at a time
    println!("{}", x);
}
```



我们可以用前面讲的模式重构A为两个更小的结构体，这样就可以解决借用检查的问题：

```rust
// A is now composed of two structs - B and C.
struct A {
    b: B,
    c: C,
}
struct B {
    f2: u32,
}
struct C {
    f1: u32,
    f3: u32,
}

// These functions take a B or C, rather than A.
fn foo(b: &mut B) -> &u32 { &b.f2 }
fn bar(c: &mut C) -> u32 { c.f1 + c.f3 }

fn baz(a: &mut A) {
    let x = foo(&mut a.b);
    // Now it's OK!
    let y = bar(&mut a.c);
    println!("{}", x);
}
```

## 出发点

TODO Why and where you should use the pattern

## 优点

这可以让你挣脱借用检查器的限制，常常会带来更好的设计。

## 缺点



需要更多的代码。

有时更小的结构体没有明确的抽象意义，最终导致做出坏设计。这种情况是一种“代码气味”（code smell），表明程序需要重构。

## 讨论

在没有借用检查器的语言里中是不需要这种模式的，所以它是Rust独有的设计模式。不过，将功能分解成更小的单元是很多有名的软件设计原则中都赞同的，这一点与语言无关。

这种模式依赖于Rust的借用检查器能够分清结构体内部的字段。在上面的例子中，借用检查器知道`a.b`和`a.c`是相互独立的，就不会尝试去借用整个`a`。

