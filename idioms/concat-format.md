# 用`format!`连接字符串

## 说明

对一个可变的`String`类型对象使用`push`或者`push_str`方法，或者用`+`操作符可以构建字符串。然而，使用`format!`常常会更方便，尤其是结合字面量和非字面量的时候。

## 例子

```rust
fn say_hello(name: &str) -> String {
    // 我们可以手动构建字符串
    // let mut result = "Hello ".to_owned();
    // result.push_str(name);
    // result.push('!');
    // result

    // 但是用format! 更好
    format!("Hello {}!", name)
}
```

## 优点
使用`format!` 连接字符串通常更加简洁和易于阅读。

## 缺点

它通常不是最有效的连接字符串的方法。对一个可变的`String`类型对象进行一连串的`push`操作通常是最有效率的（尤其这个字符串已经预先分配了足够的空间）
