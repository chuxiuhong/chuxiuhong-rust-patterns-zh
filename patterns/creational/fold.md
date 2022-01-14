# Fold

## 说明

对集合中的每个数据执行算法来创建新的项，从而创建一个全新的集合。

这里的词源对我来说是不清晰的。Rust编译器用"fold"和"folder"的说法，即使它对我来说在通常意义上更像是map而不是fold。看下面的讨论了解更多细节。

## 代码示例

```rust,ignore
// The data we will fold, a simple AST.
mod ast {
    pub enum Stmt {
        Expr(Box<Expr>),
        Let(Box<Name>, Box<Expr>),
    }

    pub struct Name {
        value: String,
    }

    pub enum Expr {
        IntLit(i64),
        Add(Box<Expr>, Box<Expr>),
        Sub(Box<Expr>, Box<Expr>),
    }
}

// The abstract folder
mod fold {
    use ast::*;

    pub trait Folder {
        // A leaf node just returns the node itself. In some cases, we can do this
        // to inner nodes too.
        fn fold_name(&mut self, n: Box<Name>) -> Box<Name> { n }
        // Create a new inner node by folding its children.
        fn fold_stmt(&mut self, s: Box<Stmt>) -> Box<Stmt> {
            match *s {
                Stmt::Expr(e) => Box::new(Stmt::Expr(self.fold_expr(e))),
                Stmt::Let(n, e) => Box::new(Stmt::Let(self.fold_name(n), self.fold_expr(e))),
            }
        }
        fn fold_expr(&mut self, e: Box<Expr>) -> Box<Expr> { ... }
    }
}

use fold::*;
use ast::*;

// An example concrete implementation - renames every name to 'foo'.
struct Renamer;
impl Folder for Renamer {
    fn fold_name(&mut self, n: Box<Name>) -> Box<Name> {
        Box::new(Name { value: "foo".to_owned() })
    }
    // Use the default methods for the other nodes.
}
```

对AST执行`Renamer`的结果是创建一个与旧AST相同的AST，但是每个name都改为`foo`。

folder也可以定义为将一个数据结构映射到不同（但基本相似）的数据结构。例如，我们可以把一个AST转换到一个高级中间代码表示树(HIR Tree)。

## 出发点

通过对数据结构中的每个节点执行一些操作来映射一个数据结构是常见的。对于简单结构上的简单操作，可以用`Iterator::map`来实现。对于更复杂的操作，或者前面的节点会影响后面节点的操作，或者数据结构上的循环是非平凡的，用fold模式更为妥帖。

类似访问者模式，fold模式允许我们将数据结构的遍历与对每个节点执行的操作分开。

## 讨论

采用这种方式映射数据结构在函数式语言中很常见。在面向对象语言中，更常见的是就地修改数据结构。Rust中常见的是"函数式"的方法，主要是因为引用的不可变性。
采用新生成数据结构而不是修改原来的结构，使在大多数情况下对代码推理更容易。

效率和可重用性之间的权衡可以通过改变`fold_*`方法对节点的接受方式来调整。

在上面的例子里我们通过`Box`指针来操作。因为独占数据，原始的数据结构不能再被使用。另一方面如果一个节点不再修改，重用它将会更高效。

如果我们对借用的引用进行操作，原来的数据结构就能被重用。不过一个节点哪怕没修改也必须克隆才能保证独占。

使用计数指针可以兼得二者——我们既可以重用原始数据结构并且我们不需要克隆没有被改变的节点。不过这不太符合人体工程学并且意味着数据结构不能是可变的。

## 参阅

迭代器有`fold`方法，不过这个fold是将数据结构压缩成一个值而不是产生一个新的数据结构。迭代器的`map`更像是这里说的fold模式。

在其他语言中，更常见的是Rust迭代器中的fold形式而不是这里说的fold模式。一些函数式语言中有对数据结构进行复杂转换的支持。

[访问者模式](../behavioural/visitor.md)和fold高度相关。 它们共享遍历数据结构的概念——在每个节点上执行操作。不过访问者模式不创建新的数据结构也不消耗原来的数据。
