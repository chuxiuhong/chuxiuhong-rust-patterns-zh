# 向闭包传递变量

## 说明

默认情况下，闭包从环境中借用捕获。或者你可以用`move`闭包来将环境的所有权全给闭包。然而，一般情况下你是想传递一部分变量到闭包中，如一些数据的拷贝、传引用或者执行一些其他操作。

这种情况应在不同的作用域里进行变量重绑定。

## 示例

像这样

```rust
use std::rc::Rc;

let num1 = Rc::new(1);
let num2 = Rc::new(2);
let num3 = Rc::new(3);
let closure = {
    // `num1` is moved
    let num2 = num2.clone();  // `num2` is cloned
    let num3 = num3.as_ref();  // `num3` is borrowed
    move || {
        *num1 + *num2 + *num3;
    }
};
```

而不是

```rust
use std::rc::Rc;

let num1 = Rc::new(1);
let num2 = Rc::new(2);
let num3 = Rc::new(3);

let num2_cloned = num2.clone();
let num3_borrowed = num3.as_ref();
let closure = move || {
    *num1 + *num2_cloned + *num3_borrowed;
};
```

## 优点

这样在闭包定义的时候就把哪些是复制的数据搞清楚，这样结束时无论闭包有没有消耗掉这些值，都会及早drop掉。

闭包能用与上下文相同的变量名来用那些复制或者move进来的变量。

## 缺点

增加了闭包内的实现代码行数。
