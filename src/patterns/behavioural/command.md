# Command 模式

## 描述

Command 模式的基本思想是将动作分离为独立的对象，并将其作为参数传递。

## 动机

假设我们有一系列被封装为对象的动作或事务。我们希望这些动作或命令能够在之后的不同时间点按某种顺序执行或调用。这些命令也可能由某些事件触发，例如，当用户按下按钮时，或在接收到数据包时。此外，这些命令可能是可撤销的。这在编辑器的操作中可能很有用。我们可能想要存储已执行命令的日志，以便在系统崩溃时可以重新应用这些更改。

## 示例

定义两个数据库操作：`create table` 和 `add field`。每个操作都是一个命令，它们都知道如何撤销命令，例如 `drop table` 和 `remove field`。当用户调用数据库迁移操作时，每个命令按定义的顺序执行；当用户调用回滚操作时，整个命令集按相反的顺序执行。

## 方法一：使用 trait 对象

我们定义一个包含 `execute` 和 `rollback` 两个操作的通用 trait。所有命令 `struct` 都必须实现这个 trait。

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

## 方法二：使用函数指针

我们可以采用另一种方法，将每个独立的命令创建为不同的函数，并存储函数指针以在之后的不同时间点调用这些函数。由于函数指针实现了 `Fn`、`FnMut` 和 `FnOnce` 这三个 trait，我们也可以传递和存储闭包而不是函数指针。

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

## 方法三：使用 `Fn` trait 对象

最后，我们可以不定义通用的命令 trait，而是分别在向量中存储实现了 `Fn` trait 的每个命令。

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

如果我们的命令较小且可以定义为函数或作为闭包传递，那么使用函数指针可能更可取，因为它不使用动态分发。但如果我们的命令是一个包含大量函数和变量的完整结构体，并且被定义为独立模块，那么使用 trait 对象会更合适。一个应用实例可以在 [`actix`](https://actix.rs/) 中找到，它在为路由注册处理函数时使用了 trait 对象。在使用 `Fn` trait 对象的情况下，我们可以像使用函数指针一样创建和使用命令。

在性能方面，总是需要在性能和代码简洁性、组织性之间进行权衡。静态分发提供更快的性能，而动态分发在我们构建应用程序时提供更多的灵活性。

## 参见

- [Command pattern](https://en.wikipedia.org/wiki/Command_pattern)

- [Another example for the `command` pattern](https://web.archive.org/web/20210223131236/https://chercher.tech/rust/command-design-pattern-rust)
