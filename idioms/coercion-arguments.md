# 以借用类型为参数

## 说明

当你为函数选择参数类型时，使用带强制隐式转换的目标会增加你代码的复杂度。在这种情况下，函数将会接受更多的输入参数类型。

使用可切片类型或者胖指针类型没有限制。事实上，你应该总是用借用类型（**borrowed type**）而不是自有数据类型的借用（**borrowing the owned type**）。例如`&str` 而非 `&String`, `&[T]` 而非 `&Vec<T>`, 或者 `&T` 而非 `&Box<T>`.

当自有数据结构（owned type）的实例已经提供了一个访问数据的间接层时，使用借用类型可以让你避免增加间接层。举例来说，`String`类型有一层间接层，所以`&String`将有两个间接层。我们可以用`&Str`来避免这种情况，无论何时调用函数，强制`&String`转换为`&Str`。

## 例子

在这个例子中，我们将说明使用`&String`与`&Str`作为函数参数的区别。这个思路用于对比`&Vec<T>` 和 `&[T]`、 `&T`和`&Box<T>`也适用。

考虑一个我们想要确定一个单词是否包含3个连续的元音字母的例子。我们不需要获得字符串的所有权，所以我们将获取一个引用。

代码如下:

```rust
fn three_vowels(word: &String) -> bool {
    let mut vowel_count = 0;
    for c in word.chars() {
        match c {
            'a' | 'e' | 'i' | 'o' | 'u' => {
                vowel_count += 1;
                if vowel_count >= 3 {
                    return true
                }
            }
            _ => vowel_count = 0
        }
    }
    false
}

fn main() {
    let ferris = "Ferris".to_string();
    let curious = "Curious".to_string();
    println!("{}: {}", ferris, three_vowels(&ferris));
    println!("{}: {}", curious, three_vowels(&curious));

    // 至此运行正常，但下面两行就会失败:
    // println!("Ferris: {}", three_vowels("Ferris"));
    // println!("Curious: {}", three_vowels("Curious"));

}
```

这里能够正常运行是因为我们传的参数是`&String`类型。最后注释的两行运行失败是因为`&str`类型不能强制隐式转换为`&String`类型。我们靠修改参数类型即可轻松解决。

例如,如果我们把函数定义改为:

```rust, ignore
fn three_vowels(word: &str) -> bool {
```

那么两种版本都能编译通过并打印相同的输出。

```bash
Ferris: false
Curious: true
```

等等，这并不是全部！这里还有点说道。你可能对自己说，这没啥事，我永远不会用`&'static str`当输入参数（像我们刚刚输入`"Ferris"`这种情况）。即使不考虑这个特殊例子，你还会发现使用`&Str`类型将会比`&String`类型带给你更大的灵活性。

让我们现在考虑一个例子：当给定一个句子，我们需确定句子中是否有单词包含3个连续的元音字母。我们也许应该用刚刚写好的函数来对句子中的每个单词做判断。
An example of this could look like this:

```rust
fn three_vowels(word: &str) -> bool {
    let mut vowel_count = 0;
    for c in word.chars() {
        match c {
            'a' | 'e' | 'i' | 'o' | 'u' => {
                vowel_count += 1;
                if vowel_count >= 3 {
                    return true
                }
            }
            _ => vowel_count = 0
        }
    }
    false
}

fn main() {
    let sentence_string =
        "Once upon a time, there was a friendly curious crab named Ferris".to_string();
    for word in sentence_string.split(' ') {
        if three_vowels(word) {
            println!("{} has three consecutive vowels!", word);
        }
    }
}
```

运行我们`&Str`参数函数定义版本会输出：

```bash
curious has three consecutive vowels!
```

然而，使用`&String`版本的函数无法在这个例子中使用。这是因为字符串的切片是`&Str`类型而非`&String`类型，其转换为`&String`类型不是隐性的，然而`&String`转换为`&Str`是低开销且隐性的。

## 参阅

- [Rust Language Reference on Type Coercions](https://doc.rust-lang.org/reference/type-coercions.html)
- For more discussion on how to handle `String` and `&str` see
  [this blog series (2015)](https://web.archive.org/web/20201112023149/https://hermanradtke.com/2015/05/03/string-vs-str-in-rust-functions.html)
  by Herman J. Radtke III
