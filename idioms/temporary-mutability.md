# 临时可变性

## 说明

有的时候我们需要准备和处理一些数据，当处理完之后就只会读取而不修改。这种情况可以变量重绑定将其改为不可变的。

也可以在代码块里将处理过程和重定义写在一起。

## 示例

要求向量在使用前必须排序。

用代码块:

```rust,ignore
let data = {
    let mut data = get_vec();
    data.sort();
    data
};

// Here `data` is immutable.
```

用变量重绑定:

```rust,ignore
let mut data = get_vec();
data.sort();
let data = data;

// Here `data` is immutable.
```

## 优点

编译器可以确保你之后不会意外修改数据。

## 缺点

多增加了一些本不必要的代码，代码结构更复杂。
