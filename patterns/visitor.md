# 访问者模式

## 说明

访问者封装了在不同对象集合上运行的算法。它支持在不修改数据的情况下，支持不同算法。（或者它们的主要行为）

此外，访问者模式允许将对象集合的遍历与对每个对象执行的操作分离开来。

## 代码示例

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

use visit::*;
use ast::*;

// An example concrete implementation - walks the AST interpreting it as code.
struct Interpreter;
impl Visitor<i64> for Interpreter {
    fn visit_name(&mut self, n: &Name) -> i64 { panic!() }
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

可以实现更多的访问者，例如类型检查器，而不必修改AST数据。

## 出发点



当你想要讲一个算法用于不同数据的时候，访问器模式是很有用的。如果数据是相同种类的，你可以用一个类似迭代器模式。使用访问者对象（而不是函数式的方法）支持访问者带有状态，从而在节点之间传递信息。

## 讨论

`visit_*`通常返回空值（与示例中的相反）。在这种情况下，可以将遍历代码分解出来并在算法之间共享。（并且提供空的默认方法）。在Rust中，通常的方法是对每种数据提供一个`walk_*`函数，例如：

```rust,ignore
pub fn walk_expr(visitor: &mut Visitor, e: &Expr) {
    match *e {
        Expr::IntLit(_) => {},
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



在其他语言中（例如Java）通常是数据提供一个`accept`方法来履行同样的职责。

## 参阅

访问者模式是面向对象语言中的一个常见模式。

[访问者模式](https://en.wikipedia.org/wiki/Visitor_pattern)

[fold](fold.md)模式与访问者模式很相似，区别在于生成了被访问数据结构的新版本。

