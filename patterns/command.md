# 命令模式

## 说明

命令模式的基本概念是，将动作分离为单独的对象，并且作为参数传递它们

## 出发点

假设我们有一连串的动作或事务被封装为对象。
我们希望这些动作或命令在以后的不同时间以某种顺序执行或调用，
这些命令也可以作为某些事件的结果被触发。例如，当用户按下某个按钮，或某个数据包到达时。
此外，这些命令应该可以撤销。这对于编辑器的操作可能很有用。我们可能想存储命令日志，
这样，如果系统崩溃，我们可以在之后重新应用这些修改。

## 示例

定义两个数据库操作，`建表`和`加字段`。每个操作都是一个命令，它知道如何撤销命令。例如，`删表`和`删字段`。当用户调用数据库迁移操作时，每条命令都会按照定义的顺序执行。而当用户调用回滚操作时，整个命令集会以相反的顺序调用。

## 使用trait对象

我们定义了一个trait，将我们的命令封装成两个操作，`execute`和`rollback`。所有命令`结构体`必须实现这个trait。

```rust
pub trait Migration {
    fn execute(&self) -> &str;
    fn rollback(&self) -> &str;
}

pub struct CreateTable;
impl Migration for CreateTable {
    fn execute(&self) -> &str {
        "create table"
    }
    fn rollback(&self) -> &str {
        "drop table"
    }
}

pub struct AddField;
impl Migration for AddField {
    fn execute(&self) -> &str {
        "add field"
    }
    fn rollback(&self) -> &str {
        "remove field"
    }
}

struct Schema {
    commands: Vec<Box<dyn Migration>>,
}

impl Schema {
    fn new() -> Self {
        Self { commands: vec![] }
    }

    fn add_migration(&mut self, cmd: Box<dyn Migration>) {
        self.commands.push(cmd);
    }

    fn execute(&self) -> Vec<&str> {
        self.commands.iter().map(|cmd| cmd.execute()).collect()
    }
    fn rollback(&self) -> Vec<&str> {
        self.commands
            .iter()
            .rev() // reverse iterator's direction
            .map(|cmd| cmd.rollback())
            .collect()
    }
}

fn main() {
    let mut schema = Schema::new();

    let cmd = Box::new(CreateTable);
    schema.add_migration(cmd);
    let cmd = Box::new(AddField);
    schema.add_migration(cmd);

    assert_eq!(vec!["create table", "add field"], schema.execute());
    assert_eq!(vec!["remove field", "drop table"], schema.rollback());
}
```

## 使用函数指针

我们可以采用另一种方法。将每个单独的命令创建为不同的函数，并存储函数指针,
以便以后在不同的时间调用这些函数。因为函数指针实现了`Fn`、
`FnMut`和`FnOnce`这三个特性，我们也可以传递和存储闭包。

```rust
type FnPtr = fn() -> String;
struct Command {
    execute: FnPtr,
    rollback: FnPtr,
}

struct Schema {
    commands: Vec<Command>,
}

impl Schema {
    fn new() -> Self {
        Self { commands: vec![] }
    }
    fn add_migration(&mut self, execute: FnPtr, rollback: FnPtr) {
        self.commands.push(Command { execute, rollback });
    }
    fn execute(&self) -> Vec<String> {
        self.commands.iter().map(|cmd| (cmd.execute)()).collect()
    }
    fn rollback(&self) -> Vec<String> {
        self.commands
            .iter()
            .rev()
            .map(|cmd| (cmd.rollback)())
            .collect()
    }
}

fn add_field() -> String {
    "add field".to_string()
}

fn remove_field() -> String {
    "remove field".to_string()
}

fn main() {
    let mut schema = Schema::new();
    schema.add_migration(|| "create table".to_string(), || "drop table".to_string());
    schema.add_migration(add_field, remove_field);
    assert_eq!(vec!["create table", "add field"], schema.execute());
    assert_eq!(vec!["remove field", "drop table"], schema.rollback());
}
```

## 使用 `Fn` trait对象

最后，我们可以在vector中分别存储实现的每个命令，而不是定义一个命令trait。

```rust
type Migration<'a> = Box<dyn Fn() -> &'a str>;

struct Schema<'a> {
    executes: Vec<Migration<'a>>,
    rollbacks: Vec<Migration<'a>>,
}

impl<'a> Schema<'a> {
    fn new() -> Self {
        Self {
            executes: vec![],
            rollbacks: vec![],
        }
    }
    fn add_migration<E, R>(&mut self, execute: E, rollback: R)
    where
        E: Fn() -> &'a str + 'static,
        R: Fn() -> &'a str + 'static,
    {
        self.executes.push(Box::new(execute));
        self.rollbacks.push(Box::new(rollback));
    }
    fn execute(&self) -> Vec<&str> {
        self.executes.iter().map(|cmd| cmd()).collect()
    }
    fn rollback(&self) -> Vec<&str> {
        self.rollbacks.iter().rev().map(|cmd| cmd()).collect()
    }
}

fn add_field() -> &'static str {
    "add field"
}

fn remove_field() -> &'static str {
    "remove field"
}

fn main() {
    let mut schema = Schema::new();
    schema.add_migration(|| "create table", || "drop table");
    schema.add_migration(add_field, remove_field);
    assert_eq!(vec!["create table", "add field"], schema.execute());
    assert_eq!(vec!["remove field", "drop table"], schema.rollback());
}
```

## 讨论

如果我们的命令很小，可以定义成函数，或作为闭包传递，那么使用函数指针可能更好，因为它不需要动态分发。但如果我们的命令是个完整的结构，有一堆函数和变量被分别定义为独立的模块，那么使用trait对象会更合适。一个应用案例可以在[`actix`](https://actix.rs/)中找到，它在为例程注册handler函数时使用了trait对象。在使用`Fn` trait对象时，我们可以用和函数指针相同的方式创建和使用命令。

说到性能，在性能和代码的简易性、组织性间我们总需要权衡。静态分发可以提供更好的性能，而动态分发在我们组织应用程序时提供了灵活性。

## 另见

- [命令模式](https://zh.wikipedia.org/wiki/%E5%91%BD%E4%BB%A4%E6%A8%A1%E5%BC%8F)

- [其他`command`模式的例子](https://web.archive.org/web/20210223131236/https://chercher.tech/rust/command-design-pattern-rust)
