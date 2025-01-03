# `#![deny(warnings)]`

## 描述

一个善意的 crate 作者想要确保他们的代码在编译时没有警告。因此他们在 crate 根部添加了以下注解：

## 示例

```rust
#![deny(warnings)]

// 一切正常
```

## 优点

这种方式简短，并且在出现任何问题时都会停止构建。

## 缺点

通过禁止编译器产生警告，crate 作者实际上放弃了 Rust 引以为豪的稳定性。有时新特性或旧的问题需要改变实现方式，因此会编写 lint 规则，在正式变为 `deny` 之前会先用 `warn` 给出一段宽限期。

例如，我们发现一个类型可能有两个具有相同方法的 `impl`。这被认为是一个不好的做法，但为了使过渡更平滑，引入了 `overlapping-inherent-impls` lint 来对遇到这种情况的人发出警告，然后在未来的版本中将其变为硬性错误。

同样，有时 API 会被废弃，所以它们的使用会发出警告，而之前没有任何提示。

所有这些都可能导致在某些变化发生时破坏构建。

此外，提供额外 lint 的 crates（例如 [rust-clippy]）将无法使用，除非移除这个注解。这个问题可以通过 [--cap-lints] 来缓解。`--cap-lints=warn` 命令行参数可以将所有 `deny` lint 错误转换为警告。

## 替代方案

有两种方法可以解决这个问题：首先，我们可以将构建设置与代码分离；其次，我们可以明确指定要 deny 的具体 lint。

以下命令行将把所有警告设置为 `deny`：

`RUSTFLAGS="-D warnings" cargo build`

任何开发者都可以这样做（或者在 Travis 等 CI 工具中设置，但要记住这可能会在某些变化时破坏构建），而不需要修改代码。

另外，我们可以在代码中指定想要 `deny` 的具体 lint。以下是一个（截至 rustc 1.48.0）可以安全 deny 的警告 lint 列表：

```rust,ignore
#![deny(
    bad_style,
    const_err,
    dead_code,
    improper_ctypes,
    non_shorthand_field_patterns,
    no_mangle_generic_items,
    overflowing_literals,
    path_statements,
    patterns_in_fns_without_body,
    private_in_public,
    unconditional_recursion,
    unused,
    unused_allocation,
    unused_comparisons,
    unused_parens,
    while_true
)]
```

此外，以下这些默认 `allow` 的 lint 也可能值得 `deny`：

```rust,ignore
#![deny(
    missing_debug_implementations,
    missing_docs,
    trivial_casts,
    trivial_numeric_casts,
    unused_extern_crates,
    unused_import_braces,
    unused_qualifications,
    unused_results
)]
```

有些人可能还想在列表中添加 `missing-copy-implementations`。

注意我们特意没有添加 `deprecated` lint，因为未来肯定会有更多的 API 被废弃。

## 另请参阅

- [所有 clippy lints 的集合](https://rust-lang.github.io/rust-clippy/master)
- [deprecate attribute] 文档
- 在你的系统上输入 `rustc -W help` 可以查看 lint 列表。同时输入 `rustc --help` 可以查看通用选项列表
- [rust-clippy] 是一个用于改进 Rust 代码的 lint 集合

[rust-clippy]: https://github.com/Manishearth/rust-clippy
[deprecate attribute]: https://doc.rust-lang.org/reference/attributes.html#deprecation
[--cap-lints]: https://doc.rust-lang.org/rustc/lints/levels.html#capping-lints
