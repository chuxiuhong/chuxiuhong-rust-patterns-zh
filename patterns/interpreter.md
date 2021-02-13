# 解释器

## 说明

如果一个问题经常出现并且需要很多且重复的步骤来解决，那么问题应该被抽象为一个简单的语言并且一个解释器对象能通过解释这种语言的句子来解决问题。

基本上，对于我们定义的任何类型的问题有如下三点：

- [领域专用语言](https://en.wikipedia.org/wiki/Domain-specific_language)，
- 这种语言的语法,
- 解决问题实例的解释器

## 出发点

我们的目标是转换简单的数学表达式为后缀表达式。（[逆波兰表达式](https://en.wikipedia.org/wiki/Reverse_Polish_notation)）。
为简单起见，表达式包含十个数字`0`,...`9`和`+`,`-`两种操作。举例来说，`2 + 4`被翻译为`2 4 +`。

## 问题的上下文无关文法

我们的任务是将中缀表达式转为后缀表达式。我们对包含`0`,...`9`和`+`,`-`的中缀表达式定义上下文无关文法包括：

- 终结符号: `0`, ..., `9`, `+`, `-`
- 非终结符号: `exp`, `term`
- 开始符号 `exp`
- 还有下述的生成规则

```ignore
exp -> exp + term
exp -> exp - term
exp -> term
term -> 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9
```

这个语法应该根据我们要用它做什么来进一步转换。举例来说，我们也许需要消除左递归。更多细节请看[Compilers: Principles,Techniques, and Tools](https://en.wikipedia.org/wiki/Compilers:_Principles,_Techniques,_and_Tools)

## 解决方案

我们只需实现一个递归下降解析器。为了简单起见，当表达式语法错误时，代码会恐慌。（例如根据语法定义，`2-34`或者`2+5-`是错误的）

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
                panic!("Unexpected symbol '{}'", op);
            }
        }
    }
    fn term(&mut self, out: &mut String) {
        match self.next_char() {
            Some(ch) if ch.is_digit(10) => out.push(ch),
            Some(ch) => panic!("Unexpected symbol '{}'", ch),
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

可能有一种错误的看法，即解释器设计模式是关于形式语言的语法设计和语法分析器的实现。事实上，这个模式是用更具体的方式表达问题实例，并实现解决这些问题实例的函数/类/结构。Rust语言有`macro_rules!`支持定义特殊语法和如何展开这种语法为源代码的规则。

在下面的例子中我们创建了一个简单的宏来计算n维向量的[欧式长度](https://en.wikipedia.org/wiki/Euclidean_distance)。写`norm!(x,1,2)`也许比打包`x,1,2`到`Vec`中然后调用函数计算要更有表达力和效率。

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

## See also

- [解释器模式](https://en.wikipedia.org/wiki/Interpreter_pattern)
- [上下文无关文法](https://en.wikipedia.org/wiki/Context-free_grammar)
- [macro_rules!](https://doc.rust-lang.org/rust-by-example/macros.html)