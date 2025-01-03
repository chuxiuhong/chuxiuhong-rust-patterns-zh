# Fold（折叠模式）

## 描述

在数据集合的每个项目上运行算法以创建新项目，从而创建一个全新的集合。

这个模式的词源对我来说并不清晰。术语 'fold' 和 'folder' 在 Rust 编译器中被使用，尽管在我看来，它更像是一个 map 而不是通常意义上的 fold。详见下面的讨论部分。

## 示例

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

在 AST 上运行 `Renamer` 的结果是一个新的 AST，它与原来的完全相同，但所有名称都被更改为 `foo`。在实际应用中，folder 可能会在结构体本身中保存一些在节点之间传递的状态。

folder 也可以被定义为将一个数据结构映射到另一个不同（但通常相似）的数据结构。例如，我们可以将 AST 折叠成 HIR 树（HIR 代表高级中间表示）。

## 动机

对数据结构中的每个节点执行某些操作来映射数据结构是一种常见需求。对于简单的数据结构和简单的操作，可以使用 `Iterator::map` 来完成。但对于更复杂的操作，比如前面的节点可能影响后面节点的操作，或者数据结构的迭代不那么简单的情况，使用 fold 模式会更合适。

与访问者模式类似，fold 模式允许我们将数据结构的遍历与对每个节点执行的操作分离开来。

## 讨论

这种方式的数据结构映射在函数式编程语言中很常见。在面向对象语言中，更常见的做法是直接修改数据结构。在 Rust 中，"函数式"方法很常见，主要是因为 Rust 倾向于不可变性。在大多数情况下，使用新的数据结构而不是修改旧的数据结构，可以使代码更容易理解。

通过改变 `fold_*` 方法接受节点的方式，可以在效率和可重用性之间进行权衡。

在上面的示例中，我们操作的是 `Box` 指针。由于这些指针独占其数据，原始数据结构的副本就无法重用。另一方面，如果节点没有改变，重用它是非常高效的。

如果我们操作借用引用，原始数据结构可以被重用；但是，即使节点未更改，也必须克隆它，这可能会很耗费资源。

使用引用计数指针可以兼顾两者的优点 - 我们可以重用原始数据结构，并且不需要克隆未更改的节点。然而，它们使用起来不太符合人体工程学，并且意味着数据结构不能是可变的。

## 相关模式

迭代器有一个 `fold` 方法，但是这个方法是将数据结构折叠成一个值，而不是折叠成一个新的数据结构。迭代器的 `map` 更类似于这种 fold 模式。

在其他语言中，fold 通常用于 Rust 迭代器的意义，而不是这种模式。一些函数式语言有强大的结构来对数据结构进行灵活的映射。

[visitor](../behavioural/visitor.md)（访问者）模式与 fold 模式密切相关。它们都有遍历数据结构并对每个节点执行操作的概念。然而，访问者不会创建新的数据结构，也不会消耗旧的数据结构。
