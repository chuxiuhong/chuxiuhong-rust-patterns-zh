# 使用借用类型作为函数参数

## 描述

使用 deref 强制转换的目标类型可以在为函数参数选择类型时增加代码的灵活性。通过这种方式，函数将能够接受更多的输入类型。

这不仅限于可切片类型或胖指针类型。实际上，你应该始终优先使用**借用类型**而不是**借用所有权类型**。
例如，使用 `&str` 而不是 `&String`，使用 `&[T]` 而不是 `&Vec<T>`，或使用 `&T` 而不是 `&Box<T>`。

使用借用类型可以避免在所有权类型已经提供了一层间接引用的情况下再增加额外的间接层。例如，`String` 已经有一层间接引用，所以 `&String` 将会有两层间接引用。我们可以通过使用 `&str` 来避免这种情况，并在函数被调用时让 `&String` 强制转换为 `&str`。

## 示例

在这个示例中，我们将说明使用 `&String` 作为函数参数与使用 `&str` 的一些区别，但这些思想同样适用于使用 `&Vec<T>` 与 `&[T]` 的对比，或使用 `&Box<T>` 与 `&T` 的对比。

考虑一个例子，我们想要判断一个单词是否包含三个连续的元音字母。我们不需要拥有这个字符串的所有权来进行判断，所以我们将使用引用。

代码可能看起来像这样：

```rust
fn three_vowels(word: &String) -> bool {
    let mut vowel_count = 0;
    for c in word.chars() {
        match c {
            'a' | 'e' | 'i' | 'o' | 'u' => {
                vowel_count += 1;
                if vowel_count >= 3 {
                    return true;
                }
            }
            _ => vowel_count = 0,
        }
    }
    false
}

fn main() {
    let ferris = "Ferris".to_string();
    let curious = "Curious".to_string();
    println!("{}: {}", ferris, three_vowels(&ferris));
    println!("{}: {}", curious, three_vowels(&curious));

    // 这样可以正常工作，但是下面两行会编译失败：
    // println!("Ferris: {}", three_vowels("Ferris"));
    // println!("Curious: {}", three_vowels("Curious"));
}
```

这段代码可以正常工作，因为我们传递的是 `&String` 类型的参数。如果我们取消最后两行的注释，示例将会失败。这是因为 `&str` 类型不会强制转换为 `&String` 类型。我们可以通过简单修改参数类型来解决这个问题。

例如，如果我们将函数声明改为：

```rust, ignore
fn three_vowels(word: &str) -> bool {
```

那么两个版本都能编译并打印相同的输出：

```bash
Ferris: false
Curious: true
```

但是等等，这还不是全部！这个故事还有更多内容。你可能会对自己说：这无关紧要，我永远不会使用 `&'static str` 作为输入（就像我们使用 `"Ferris"` 时那样）。即使忽略这个特殊的例子，你可能仍然会发现使用 `&str` 会比使用 `&String` 给你更多的灵活性。

让我们来看一个例子，假设有人给我们一个句子，我们想要确定句子中的任何单词是否包含三个连续的元音。我们应该利用已经定义的函数，简单地将句子中的每个单词传入即可。

示例可能如下所示：

```rust
fn three_vowels(word: &str) -> bool {
    let mut vowel_count = 0;
    for c in word.chars() {
        match c {
            'a' | 'e' | 'i' | 'o' | 'u' => {
                vowel_count += 1;
                if vowel_count >= 3 {
                    return true;
                }
            }
            _ => vowel_count = 0,
        }
    }
    false
}

fn main() {
    let sentence_string =
        "Once upon a time, there was a friendly curious crab named Ferris".to_string();
    for word in sentence_string.split(' ') {
        if three_vowels(word) {
            println!("{word} has three consecutive vowels!");
        }
    }
}
```

使用参数类型为 `&str` 的函数运行这个示例将输出：

```bash
curious has three consecutive vowels!
```

然而，当我们的函数声明使用 `&String` 作为参数类型时，这个示例将无法运行。这是因为字符串切片是 `&str` 而不是 `&String`，将其转换为 `&String` 需要分配内存，这种转换不是隐式的，而从 `String` 转换为 `&str` 则是廉价且隐式的。

## 参见

- [Rust 语言参考中关于类型强制转换的部分](https://doc.rust-lang.org/reference/type-coercions.html)
- 关于如何处理 `String` 和 `&str` 的更多讨论，请参见 Herman J. Radtke III 的
  [这个博客系列 (2015)](https://web.archive.org/web/20201112023149/https://hermanradtke.com/2015/05/03/string-vs-str-in-rust-functions.html)
- [Steve Klabnik 的博客文章 '什么时候应该使用 String vs &str?'](https://archive.ph/LBpD0)
