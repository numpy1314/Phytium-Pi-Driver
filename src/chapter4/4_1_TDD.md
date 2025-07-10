# 4.1 测试驱动开发

测试驱动开发（TDD）是一种软件开发方法，强调在编写实际代码之前先编写测试用例。它的主要步骤包括：

1. 先写一个失败的单元测试

2. 编写实现代码，使得该测试通过

3. 重构代码，保持所有测试通过状态

优点：

1. 改善设计：在编写测试的过程中，可以从使用者的角度去思考问题，这有助于提前发现设计上的问题，从而提高代码的质量。

2. 降低风险：有了充分的单元测试，我们可以放心地进行重构，因为一旦我们的修改破坏了原有的功能，测试会立刻发现。

3. 提高效率：虽然TDD需要在初期投入更多的时间，但是随着项目的推进，越来越多的测试将会大大减少因为错误和回归带来的时间损失。

## 实施

### 测试工具

- [qemu](https://www.qemu.org/) - 一个开源的虚拟机，可以模拟多种硬件平台。
- [cargo test](https://doc.rust-lang.org/cargo/commands/cargo-test.html)
- [bare-test](https://crates.io/crates/bare-test) - 是一个轻量级的测试框架，专为裸机环境设计。它允许在没有操作系统支持的情况下运行测试(本身是一个小型操作系统)。
- [gdb-multiarch](https://www.gnu.org/software/gdb/) - GNU调试器，支持多种架构。

### 环境搭建

#### Linux

`apt`、`cargo` 等工具直接安装

#### Windows

推荐

- [msys2](https://www.msys2.org/) - 一个轻量级的Linux环境，可以在Windows上运行。可方便的安装`qemu`、`gdb-multiarch`等工具。

### 自定义测试原理

1. `Cargo test` 默认如何工作?

    `cargo test` 命令会编译项目中的所有测试代码，`libtest` 收集 `#[test]` 标记的函数，成一个可执行文件，然后运行这个可执行文件来获取执行结果。

2. 我们从哪里开始使用自定义测试框架？

    在 `Cargo.toml` 中添加

    ```toml
    [[test]]
    name = "test"
    path = "tests/test.rs"
    harness = false
    ```

3. 定义测试函数声明规范

    ...

4. 收集所有测试函数

    ...

5. 构建`no-std`裸机程序

    ...

6. 修改 `Cargo runner`为`Qemu`

    ...

### 项目搭建

[https://github.com/drivercraft/bare-test-template](https://github.com/drivercraft/bare-test-template)

点击 "Use this template" 按钮，创建一个新的仓库。

修改 `Cargo.toml` 文件:

```toml
[package]
edition = "2024"
name = "project-name"
version = "0.1.0"
```

`project-name` 替换为你的项目名称。

`src/lib.rs` 中构建你的驱动程序

`tests/test.rs` 中编写测试代码。

### 运行测试

安装 `ostool`

```bash
cargo install ostool
```

运行测试

```bash
cargo test --package project-name --test test -- tests --show-output
# 带uboot的开发板测试
cargo test --package project-name --test test -- tests --show-output --uboot 
```

### 完善测试工作流

github\gitee ci

CI步骤：

1. `Rust`环境
2. `Qemu`安装
3. 代码质量检查
4. 编译和测试
