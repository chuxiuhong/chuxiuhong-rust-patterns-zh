# 解释器模式

## 描述

如果一个问题经常出现且需要长期重复的步骤来解决，那么这类问题实例可以用一种简单的语言来表达，并通过解释器对象来解释这种简单语言中的语句来解决问题。

基本上，对于任何类型的问题，我们需要定义：

- 一个[domain specific language](https://en.wikipedia.org/wiki/Domain-specific_language)（领域特定语言）
- 这个语言的语法规则
- 一个用于解决问题实例的解释器

## 动机

我们的目标是将简单的数学表达式转换为后缀表达式（也称为[Reverse Polish notation](https://en.wikipedia.org/wiki/Reverse_Polish_notation)，逆波兰表示法）。
为了简单起见，我们的表达式仅包含数字`0`到`9`和两种运算符`+`、`-`。例如，表达式`2 + 4`将被转换为`2 4 +`。

## 问题的上下文无关文法

我们的任务是将中缀表达式转换为后缀表达式。让我们为包含`0`到`9`、`+`和`-`的中缀表达式集合定义一个上下文无关文法，其中：

- 终结符号：`0`、...、`9`、`+`、`-`
- 非终结符号：`exp`、`term`
- 开始符号是`exp`
- 以下是产生式规则

```ignore
exp -> exp + term
exp -> exp - term
exp -> term
term -> 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9
```

**注意：** 根据具体使用场景，这个语法可能需要进一步转换。例如，我们可能需要消除左递归。更多详细信息请参考[Compilers: Principles,Techniques, and Tools](https://en.wikipedia.org/wiki/Compilers:_Principles,_Techniques,_and_Tools)（又称龙书）。

## 解决方案

我们实现了一个简单的递归下降解析器。为了简单起见，当表达式在语法上不正确时（例如`2-34`或`2+5-`根据语法定义是错误的），代码会直接panic。

```rust
pub struct Interpreter<'a> {
    it: std::str::Chars<'a>,
}

impl<'a> Interpreter<'a> {
    pub fn new(infix: &'a str) -> Self {
        Self { it: infix.chars() }
    }

    fn next_char(&mut self) -> Option<char> {
        self.it.next()
    }

    pub fn interpret(&mut self, out: &mut String) {
        self.term(out);

        while let Some(op) = self.next_char() {
            if op == '+' || op == '-' {
                self.term(out);
                out.push(op);
            } else {
                panic!("Unexpected symbol '{op}'");
            }
        }
    }

    fn term(&mut self, out: &mut String) {
        match self.next_char() {
            Some(ch) if ch.is_digit(10) => out.push(ch),
            Some(ch) => panic!("Unexpected symbol '{ch}'"),
            None => panic!("Unexpected end of string"),
        }
    }
}

pub fn main() {
    let mut intr = Interpreter::new("2+3");
    let mut postfix = String::new();
    intr.interpret(&mut postfix);
    assert_eq!(postfix, "23+");

    intr = Interpreter::new("1-2+3-4");
    postfix.clear();
    intr.interpret(&mut postfix);
    assert_eq!(postfix, "12-3+4-");
}
```

## 讨论

可能有一种误解认为解释器设计模式是关于为形式语言设计语法并实现这些语法的解析器。实际上，这个模式是关于以更具体的方式表达问题实例，并实现用于解决这些问题实例的函数/类/结构体。Rust语言有`macro_rules!`，它允许我们定义特殊语法并规定如何将这种语法展开为源代码。

在下面的例子中，我们创建了一个简单的`macro_rules!`来计算`n`维向量的[Euclidean length](https://en.wikipedia.org/wiki/Euclidean_distance)（欧几里得长度）。相比将`x,1,2`打包成一个`Vec`并调用计算长度的函数，写成`norm!(x,1,2)`可能更容易表达且更高效。

```rust
macro_rules! norm {
    ($($element:expr),*) => {
        {
            let mut n = 0.0;
            $(
                n += ($element as f64)*($element as f64);
            )*
            n.sqrt()
        }
    };
}

fn main() {
    let x = -3f64;
    let y = 4f64;

    assert_eq!(3f64, norm!(x));
    assert_eq!(5f64, norm!(x, y));
    assert_eq!(0f64, norm!(0, 0, 0));
    assert_eq!(1f64, norm!(0.5, -0.5, 0.5, -0.5));
}
```

## 参见

- [Interpreter pattern](https://en.wikipedia.org/wiki/Interpreter_pattern)（解释器模式）
- [Context free grammar](https://en.wikipedia.org/wiki/Context-free_grammar)（上下文无关文法）
- [macro_rules!](https://doc.rust-lang.org/rust-by-example/macros.html)
