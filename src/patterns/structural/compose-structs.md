# 结构体分解以实现独立借用

## 描述

有时一个大型结构体会导致借用检查器出现问题 - 虽然字段可以被独立借用，但有时整个结构体会被一次性使用，从而阻止其他使用。一个解决方案是将结构体分解成几个更小的结构体，然后将这些结构体组合成原始结构体。这样每个结构体都可以被单独借用，从而获得更灵活的行为。

这种模式通常会带来其他方面的更好设计：应用这种设计模式往往能揭示出更小的功能单元。

## 示例

这里有一个人为构造的例子，展示了借用检查器是如何阻碍我们使用结构体的计划的：

```rust
struct Database {
    connection_string: String,
    timeout: u32,
    pool_size: u32,
}

fn print_database(database: &Database) {
    println!("Connection string: {}", database.connection_string);
    println!("Timeout: {}", database.timeout);
    println!("Pool size: {}", database.pool_size);
}

fn main() {
    let mut db = Database {
        connection_string: "initial string".to_string(),
        timeout: 30,
        pool_size: 100,
    };

    let connection_string = &mut db.connection_string;
    print_database(&db); // 此处发生对 `db` 的不可变借用
                         // *connection_string = "new string".to_string();  // 此处使用可变借用
}
```

我们可以应用这种设计模式，将 `Database` 重构为三个更小的结构体，从而解决借用检查的问题：

```rust
// Database 现在由三个结构体组成 - ConnectionString、Timeout 和 PoolSize
// 让我们将其分解为更小的结构体
#[derive(Debug, Clone)]
struct ConnectionString(String);

#[derive(Debug, Clone, Copy)]
struct Timeout(u32);

#[derive(Debug, Clone, Copy)]
struct PoolSize(u32);

// 然后我们将这些小结构体重新组合成 `Database`
struct Database {
    connection_string: ConnectionString,
    timeout: Timeout,
    pool_size: PoolSize,
}

// print_database 现在可以接收 ConnectionString、Timeout 和 Poolsize 结构体作为参数
fn print_database(connection_str: ConnectionString, timeout: Timeout, pool_size: PoolSize) {
    println!("Connection string: {connection_str:?}");
    println!("Timeout: {timeout:?}");
    println!("Pool size: {pool_size:?}");
}

fn main() {
    // 使用三个结构体初始化 Database
    let mut db = Database {
        connection_string: ConnectionString("localhost".to_string()),
        timeout: Timeout(30),
        pool_size: PoolSize(100),
    };

    let connection_string = &mut db.connection_string;
    print_database(connection_string.clone(), db.timeout, db.pool_size);
    *connection_string = ConnectionString("new string".to_string());
}
```

## 动机

当你有一个包含许多需要独立借用的字段的结构体时，这种模式最为有用。通过这种方式，最终可以获得更灵活的行为。

## 优点

结构体的分解让你能够绕过借用检查器的限制。而且通常能产生更好的设计。

## 缺点

这可能会导致代码更加冗长。有时，较小的结构体并不是好的抽象，因此我们最终可能得到更糟糕的设计。这可能是一个"代码异味"，表明程序应该以某种方式进行重构。

## 讨论

这种模式在没有借用检查器的语言中是不需要的，因此在某种意义上是 Rust 独有的。然而，创建更小的功能单元通常会带来更清晰的代码：这是一个广泛认可的软件工程原则，与编程语言无关。

这种模式依赖于 Rust 的借用检查器能够独立借用字段。在示例中，借用检查器知道 `a.b` 和 `a.c` 是不同的，可以独立借用，它不会尝试借用整个 `a`，这使得这种模式变得有用。
