# Visitor（访问者模式）

## 描述

访问者模式封装了一个可以在异构对象集合上操作的算法。它允许在相同的数据上编写多个不同的算法，而无需修改数据（或其主要行为）。

此外，访问者模式允许将对象集合的遍历与对每个对象执行的操作分离开来。

## 示例

```rust,ignore
// The data we will visit
mod ast {
    pub enum Stmt {
        Expr(Expr),
        Let(Name, Expr),
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

// The abstract visitor
mod visit {
    use ast::*;

    pub trait Visitor<T> {
        fn visit_name(&mut self, n: &Name) -> T;
        fn visit_stmt(&mut self, s: &Stmt) -> T;
        fn visit_expr(&mut self, e: &Expr) -> T;
    }
}

use ast::*;
use visit::*;

// An example concrete implementation - walks the AST interpreting it as code.
struct Interpreter;
impl Visitor<i64> for Interpreter {
    fn visit_name(&mut self, n: &Name) -> i64 {
        panic!()
    }
    fn visit_stmt(&mut self, s: &Stmt) -> i64 {
        match *s {
            Stmt::Expr(ref e) => self.visit_expr(e),
            Stmt::Let(..) => unimplemented!(),
        }
    }

    fn visit_expr(&mut self, e: &Expr) -> i64 {
        match *e {
            Expr::IntLit(n) => n,
            Expr::Add(ref lhs, ref rhs) => self.visit_expr(lhs) + self.visit_expr(rhs),
            Expr::Sub(ref lhs, ref rhs) => self.visit_expr(lhs) - self.visit_expr(rhs),
        }
    }
}
```

我们可以实现更多的访问者，比如类型检查器，而无需修改 AST 数据结构。

## 动机

当你需要对异构数据应用算法时，访问者模式非常有用。如果数据是同构的，你可以使用迭代器类的模式。使用访问者对象（而不是函数式方法）允许访问者保持状态，从而在节点之间传递信息。

## 讨论

通常情况下，`visit_*` 方法返回 void 类型（与示例中不同）。在这种情况下，可以将遍历代码提取出来，在不同算法之间共享（同时提供默认的空操作方法）。在 Rust 中，常见的做法是为每个数据类型提供 `walk_*` 函数。例如：

```rust,ignore
pub fn walk_expr(visitor: &mut Visitor, e: &Expr) {
    match *e {
        Expr::IntLit(_) => {}
        Expr::Add(ref lhs, ref rhs) => {
            visitor.visit_expr(lhs);
            visitor.visit_expr(rhs);
        }
        Expr::Sub(ref lhs, ref rhs) => {
            visitor.visit_expr(lhs);
            visitor.visit_expr(rhs);
        }
    }
}
```

在其他语言中（如 Java），数据通常会有一个 `accept` 方法来完成相同的功能。

## 参见

访问者模式是大多数面向对象语言中的常见模式。

[Wikipedia 文章](https://en.wikipedia.org/wiki/Visitor_pattern)

[fold](../creational/fold.md)（折叠）模式与访问者模式类似，但会产生被访问数据结构的新版本。
