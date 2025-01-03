# 向闭包传递变量

## 描述

默认情况下，闭包通过借用来捕获其环境。或者你可以使用 `move` 闭包来移动整个环境。然而，通常你可能只想把某些变量移动到闭包中，给它一些数据的副本，通过引用传递，或者执行其他转换。

这种情况下，可以在单独的作用域中使用变量重绑定来实现。

## 示例

```rust
use std::rc::Rc;

let num1 = Rc::new(1);
let num2 = Rc::new(2);
let num3 = Rc::new(3);
let closure = {
    // `num1` 被移动
    let num2 = num2.clone();  // `num2` 被克隆
    let num3 = num3.as_ref();  // `num3` 被借用
    move || {
        *num1 + *num2 + *num3;
    }
};
```

替代以下写法：

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

被复制的数据与闭包定义被组织在一起，这使得它们的用途更加清晰，而且即使它们没有被闭包消费，也会立即被丢弃。

闭包使用与周围代码相同的变量名，无论数据是被复制还是移动。

## 缺点

闭包主体需要额外的缩进。
