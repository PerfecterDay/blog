# Cargo
{docsify-updated}

+ 查看 cargo 版本：`cargo --version`

## 使用 Cargo 创建项目
`cargo new hello_cargo` ,会创建一个 hello_cargo 目录，里边有一个 Cargo.toml 文件和一个 src 目录：
```
└─$ tree hello_cargo/
hello_cargo/
├── Cargo.toml
└── src
	└── main.rs
```

这也会在 hello_cargo 目录初始化了一个 git 仓库，以及一个 .gitignore 文件。如果在一个已经存在的 git 仓库中运行 cargo new，则这些 git 相关文件则不会生成；可以通过运行 `cargo new --vcs=git` 来覆盖这些行为。

## 构建并运行 Cargo 项目
在项目根目录下运行：`cargo build`，这个命令会创建一个可执行文件 `target/debug/hello_cargo` ，而不是放在目前目录下。由于默认的构建方法是调试构建（debug build），Cargo 会将可执行文件放在名为 debug 的目录中:
```
└─$ tree
.
├── Cargo.lock
├── Cargo.toml
├── src
│   └── main.rs
└── target
    ├── CACHEDIR.TAG
    └── debug
        ├── build
        ├── deps
        │   ├── hello_cargo-deaad4c17ac37e0b
        │   └── hello_cargo-deaad4c17ac37e0b.d
        ├── examples
        ├── hello_cargo
        ├── hello_cargo.d
        └── incremental
            └── hello_cargo-2pueg9mozk8ji
                ├── s-gxicz4d46o-1bykc7u-11wguxc3jq96bydmcaozl5au
                │   ├── 1jcp40zb4zxwgzzu.o
                │   ├── 22kjk1sxdhcblxcw.o
                │   ├── 2fltho8o95x6j6qq.o
                │   ├── 4x9bxtcyjq62i9bx.o
                │   ├── 54s9b2m4at8mbuh4.o
                │   ├── dep-graph.bin
                │   ├── jqh9v2bjf4umt2x.o
                │   ├── query-cache.bin
                │   └── work-products.bin
                └── s-gxicz4d46o-1bykc7u.lock
```

首次运行 `cargo build` 时，也会使 Cargo 在项目根目录创建一个新文件：`Cargo.lock`。这个文件记录项目依赖的实际版本。这个项目并没有依赖，所以其内容比较少。你自己永远也不需要碰这个文件，让 Cargo 处理它就行了。

前面我们使用 `cargo build` 构建了项目，并使用 ./target/debug/hello_cargo 运行了程序，也可以使用 `cargo run` 在一个命令中同时编译并运行生成的可执行文件。

Cargo 如果发现文件并没有被改变，它不会有重新编译，而是直接运行了可执行文件。如果修改了源文件的话，Cargo 会在运行之前重新构建项目。

`cargo check` 该命令快速检查代码确保其可以编译，但并不产生可执行文件：
```
┌──(admin㉿DESKTOP-CODER)-[~/rust-workspace/hello_cargo]
└─$ cargo check
    Checking hello_cargo v0.1.0 (/home/admin/rust-workspace/hello_cargo)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.08s

┌──(admin㉿DESKTOP-CODER)-[~/rust-workspace/hello_cargo]
└─$ tree
.
├── Cargo.lock
├── Cargo.toml
├── src
│   └── main.rs
└── target
    ├── CACHEDIR.TAG
    └── debug
        ├── build
        ├── deps
        │   ├── hello_cargo-8834076585394559.d
        │   └── libhello_cargo-8834076585394559.rmeta
        ├── examples
        └── incremental
            └── hello_cargo-2e0bekeiibgkq
                ├── s-gxid6hpfkn-4cqa94-69k2wdgv6sk6mt1irzppt3rw9
                │   ├── dep-graph.bin
                │   ├── query-cache.bin
                │   └── work-products.bin
                └── s-gxid6hpfkn-4cqa94.lock

10 directories, 10 files
```

+ `cargo new`: 创建项目。
+ `cargo build`: 构建项目。
+ `cargo run`: 一步构建并运行项目。
+ `cargo check`: 在不生成二进制文件的情况下构建项目来检查错误。
+ `cargo doc --open`: 来构建所有本地依赖提供的文档，并在浏览器中打开crate 使用说明文档
+ 有别于将构建结果放在与源码相同的目录，Cargo 会将其放到 `target/debug` 目录。

## 发布（release）构建
当项目最终准备好发布时，可以使用 `cargo build --release` 来优化编译项目。这会在 `target/release` 而不是 `target/debug` 下生成可执行文件。这些优化可以让 Rust 代码运行的更快，不过启用这些优化也需要消耗更长的编译时间。这也就是为什么会有两种不同的配置：
+ 一种是为了开发，你需要经常快速重新构建；
+ 另一种是为用户构建最终程序，它们不会经常重新构建，并且希望程序运行得越快越好。
 
如果你在测试代码的运行时间，请确保运行 `cargo build --release` 并使用 `target/release` 下的可执行文件进行测试。


## profile
Cargo 有两个主要的配置：
+ 运行 cargo build 时采用的 dev 配置
+ 运行 cargo build --release 的 release 配置。

dev 配置为开发定义了良好的默认配置，release 配置则为发布构建定义了良好的默认配置。

当项目的 Cargo.toml 文件中没有显式增加任何 `[profile.*]` 部分的时候，Cargo 会对每一个配置都采用默认设置。通过增加任何希望定制的配置对应的 `[profile.*]` 部分，我们可以选择覆盖任意默认设置的子集。例如，如下是 dev 和 release 配置的 opt-level 设置的默认值：
```
[profile.dev]
opt-level = 0

[profile.release]
opt-level = 3
```

## 发布crate到Crates.io


### 创建 Crates.io 账号
在发布任何 crate 之前，需要在 crates.io 上注册账号并获取一个 API token。为此，访问位于 crates.io 的首页并使用 GitHub 账号登录。（目前 GitHub 账号是必须的，不过将来该网站可能会支持其他创建账号的方法）一旦登录之后，查看位于 https://crates.io/me/ 的账户设置页面并获取 API token。接着使用该 API token 运行 cargo login 命令，像这样：
```
cargo login abcdefghijklmnopqrstuvwxyz012345
```
这个命令会通知 Cargo 你的 API token 并将其储存在本地的 `~/.cargo/credentials` 文件中。注意这个 token 是一个 秘密（secret）且不应该与其他人共享。如果因为任何原因与他人共享了这个信息，应该立即到 crates.io 撤销并重新生成一个 token。

### 向新 crate 添加元信息
比如说你已经有一个希望发布的 crate。在发布之前，你需要在 crate 的 Cargo.toml 文件的 `[package]` 部分增加一些本 crate 的元信息（metadata）。

```
[package]
name = "guessing_game"
version = "0.1.0"
edition = "2021"
description = "A fun game where you guess what number the computer has chosen."
license = "MIT OR Apache-2.0"

[dependencies]
```

### 发布到 Crates.io
现在我们创建了一个账号，保存了 API token，为 crate 选择了一个名字，并指定了所需的元数据，你已经准备好发布了！发布 crate 会上传特定版本的 crate 到 crates.io 以供他人使用。

发布 crate 时请多加小心，因为发布是 永久性的（permanent）。对应版本不可能被覆盖，其代码也不可能被删除。crates.io 的一个主要目标是作为一个存储代码的永久文档服务器，这样所有依赖 crates.io 中的 crate 的项目都能一直正常工作。而允许删除版本没办法达成这个目标。然而，可以被发布的版本号却没有限制。

使用 `cargo publish` 命令发布 crate

## 工作空间
随着项目开发的深入，库 crate 持续增大，而你希望将其进一步拆分成多个库 crate。Cargo 提供了一个叫 **工作空间**（*workspaces*）的功能，它可以帮助我们管理多个相关的协同开发的包。

### 创建工作空间
**工作空间** 是一系列共享同样的 *Cargo.lock* 和输出目录的包。让我们使用工作空间创建一个项目 —— 这里采用常见的代码以便可以关注工作空间的结构。有多种组织工作空间的方式，所以我们只展示一个常用方法。我们的工作空间有一个二进制项目和两个库。二进制项目会提供主要功能，并会依赖另两个库。一个库会提供 `add_one` 方法而第二个会提供 `add_two` 方法。这三个 crate 将会是相同工作空间的一部分。让我们以新建工作空间目录开始：

```console
$ mkdir add
$ cd add
```

接着在 *add* 目录中，创建 *Cargo.toml* 文件。这个 *Cargo.toml* 文件配置了整个工作空间。它不会包含 `[package]` 部分。相反，它以 `[workspace]` 部分作为开始，并通过指定 *adder* 的路径来为工作空间增加成员，如下会加入二进制 crate：

```toml
[workspace]

members = [
    "adder",
]
```

接下来，在 *add* 目录运行 `cargo new` 新建 `adder` 二进制 crate：

```console
$ cargo new adder
     Created binary (application) `adder` package
```

到此为止，可以运行 `cargo build` 来构建工作空间。*add* 目录中的文件应该看起来像这样：

```text
├── Cargo.lock
├── Cargo.toml
├── adder
│   ├── Cargo.toml
│   └── src
│       └── main.rs
└── target
```

工作空间在顶级目录有一个 *target* 目录；`adder` 并没有自己的 *target* 目录。即使进入 *adder* 目录运行 `cargo build`，构建结果也位于 *add/target* 而不是 *add/adder/target*。工作空间中的 crate 之间相互依赖。如果每个 crate 有其自己的 *target* 目录，为了在自己的 *target* 目录中生成构建结果，工作空间中的每一个 crate 都不得不相互重新编译其他 crate。通过共享一个 *target* 目录，工作空间可以避免其他 crate 重复构建。

### 在工作空间中创建第二个包

接下来，让我们在工作空间中指定另一个成员 crate。这个 crate 位于 *add_one* 目录中，所以修改顶级 *Cargo.toml* 为也包含 *add_one* 路径：
```toml
[workspace]

members = [
    "adder",
    "add_one",
]
```

接着新生成一个叫做 `add_one` 的库：

```console
$ cargo new add_one --lib
     Created library `add_one` package
```

现在 *add* 目录应该有如下目录和文件：

```text
├── Cargo.lock
├── Cargo.toml
├── add_one
│   ├── Cargo.toml
│   └── src
│       └── lib.rs
├── adder
│   ├── Cargo.toml
│   └── src
│       └── main.rs
└── target
```

在 *add_one/src/lib.rs* 文件中，增加一个 `add_one` 函数：
```rust,noplayground
pub fn add_one(x: i32) -> i32 {
    x + 1
}
```

现在我们有了二进制 `adder` 依赖库 crate `add_one`。首先需要在 *adder/Cargo.toml* 文件中增加 `add_one` 作为路径依赖：
```toml
[dependencies]
add_one = { path = "../add_one" }
```

cargo 并不假定工作空间中的 Crates 会相互依赖，所以需要明确表明工作空间中 crate 的依赖关系。

接下来，在 `adder` crate 中使用（ `add_one` crate 中的）函数 `add_one`。打开 *adder/src/main.rs* 在顶部增加一行 `use` 将新 `add_one` 库 crate 引入作用域。接着修改 `main` 函数来调用 `add_one` 函数，如下所示：
```rust
use add_one;

fn main() {
    let num = 10;
    println!("Hello, world! {num} plus one is {}!", add_one::add_one(num));
}
```

在 *add* 目录中运行 `cargo build` 来构建工作空间！

```console
$ cargo build
   Compiling add_one v0.1.0 (file:///projects/add/add_one)
   Compiling adder v0.1.0 (file:///projects/add/adder)
    Finished dev [unoptimized + debuginfo] target(s) in 0.68s
```

为了在顶层 *add* 目录运行二进制 crate，可以通过 `-p` 参数和包名称来运行 `cargo run` 指定工作空间中我们希望使用的包：

```console
$ cargo run -p adder
    Finished dev [unoptimized + debuginfo] target(s) in 0.0s
     Running `target/debug/adder`
Hello, world! 10 plus one is 11!
```

这会运行 *adder/src/main.rs* 中的代码，其依赖 `add_one` crate


#### 在工作空间中依赖外部包
还需注意的是工作空间只在根目录有一个 *Cargo.lock*，而不是在每一个 crate 目录都有 *Cargo.lock*。这确保了所有的 crate 都使用完全相同版本的依赖。如果在 *Cargo.toml* 和 *add_one/Cargo.toml* 中都增加 `rand` crate，则 Cargo 会将其都解析为同一版本并记录到唯一的 *Cargo.lock* 中。使得工作空间中的所有 crate 都使用相同的依赖意味着其中的 crate 都是相互兼容的。让我们在 *add_one/Cargo.toml* 中的 `[dependencies]` 部分增加 `rand` crate 以便能够在 `add_one` crate 中使用 `rand` crate：

```toml
[dependencies]
rand = "0.8.5"
```

现在就可以在 *add_one/src/lib.rs* 中增加 `use rand;` 了，接着在 *add* 目录运行 `cargo build` 构建整个工作空间就会引入并编译 `rand` crate：

```console
$ cargo build
    Updating crates.io index
  Downloaded rand v0.8.5
   --snip--
   Compiling rand v0.8.5
   Compiling add_one v0.1.0 (file:///projects/add/add_one)
warning: unused import: `rand`
 --> add_one/src/lib.rs:1:5
  |
1 | use rand;
  |     ^^^^
  |
  = note: `#[warn(unused_imports)]` on by default

warning: `add_one` (lib) generated 1 warning
   Compiling adder v0.1.0 (file:///projects/add/adder)
    Finished dev [unoptimized + debuginfo] target(s) in 10.18s
```

现在顶级的 *Cargo.lock* 包含了 `add_one` 的 `rand` 依赖的信息。然而，即使 `rand` 被用于工作空间的某处，也不能在其他 crate 中使用它，除非也在它们的 *Cargo.toml* 中加入 `rand`。例如，如果在顶级的 `adder` crate 的 *adder/src/main.rs* 中增加 `use rand;`，会得到一个错误：

```console
$ cargo build
  --snip--
   Compiling adder v0.1.0 (file:///projects/add/adder)
error[E0432]: unresolved import `rand`
 --> adder/src/main.rs:2:5
  |
2 | use rand;
  |     ^^^^ no external crate `rand`
```

为了修复这个错误，修改顶级 `adder` crate 的 *Cargo.toml* 来表明 `rand` 也是这个 crate 的依赖。构建 `adder` crate 会将 `rand` 加入到 *Cargo.lock* 中 `adder` 的依赖列表中，但是这并不会下载 `rand` 的额外拷贝。Cargo 确保了工作空间中任何使用 `rand` 的 crate 都采用相同的版本，这节省了空间并确保了工作空间中的 crate 将是相互兼容的。

#### 为工作空间增加测试
作为另一个提升，让我们为 `add_one` crate 中的 `add_one::add_one` 函数增加一个测试：
```rust
pub fn add_one(x: i32) -> i32 {
    x + 1
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_works() {
        assert_eq!(3, add_one(2));
    }
}
```

在顶级 *add* 目录运行 `cargo test`。在像这样的工作空间结构中运行 `cargo test` 会运行工作空间中所有 crate 的测试。：

```console
$ cargo test
   Compiling add_one v0.1.0 (file:///projects/add/add_one)
   Compiling adder v0.1.0 (file:///projects/add/adder)
    Finished test [unoptimized + debuginfo] target(s) in 0.27s
     Running unittests src/lib.rs (target/debug/deps/add_one-f0253159197f7841)

running 1 test
test tests::it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

     Running unittests src/main.rs (target/debug/deps/adder-49979ff40686fa8e)

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests add_one

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

输出的第一部分显示 `add_one` crate 的 `it_works` 测试通过了。下一个部分显示 `adder` crate 中找到了 0 个测试，最后一部分显示 `add_one` crate 中有 0 个文档测试。

也可以选择运行工作空间中特定 crate 的测试，通过在根目录使用 `-p` 参数并指定希望测试的 crate 名称：

```console
$ cargo test -p add_one
    Finished test [unoptimized + debuginfo] target(s) in 0.00s
     Running unittests src/lib.rs (target/debug/deps/add_one-b3235fea9a156f74)

running 1 test
test tests::it_works ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

   Doc-tests add_one

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

输出显示了 `cargo test` 只运行了 `add_one` crate 的测试而没有运行 `adder` crate 的测试。

如果你选择向 [crates.io](https://crates.io/)发布工作空间中的 crate，每一个工作空间中的 crate 需要单独发布。就像 `cargo test` 一样，可以通过 `-p` 参数并指定期望发布的 crate 名来发布工作空间中的某个特定的 crate。

现在尝试以类似 `add_one` crate 的方式向工作空间增加 `add_two` crate 来作为更多的练习！

随着项目增长，考虑使用工作空间：每一个更小的组件比一大块代码要容易理解。如果它们经常需要同时被修改的话，将 crate 保持在工作空间中更易于协调 crate 的改变。

## 使用 `cargo install` 安装二进制文件
`cargo install` 命令用于在本地安装和使用二进制 crate。它并不打算替换系统中的包；它意在作为一个方便 Rust 开发者们安装其他人已经在 [crates.io](https://crates.io/)<!-- ignore --> 上共享的工具的手段。只有拥有二进制目标文件的包能够被安装。**二进制目标** 文件是在 crate 有 *src/main.rs* 或者其他指定为二进制文件时所创建的可执行程序，这不同于自身不能执行但适合包含在其他程序中的库目标文件。通常 crate 的 *README* 文件中有该 crate 是库、二进制目标还是两者兼有的信息。

所有来自 `cargo install` 的二进制文件都安装到 Rust 安装根目录的 *bin* 文件夹中。如果你是使用 *rustup.rs* 来安装 Rust 且没有自定义任何配置，这个目录将是 `$HOME/.cargo/bin`。确保将这个目录添加到 `$PATH` 环境变量中就能够运行通过 `cargo install` 安装的程序了。

例如，第十二章提到的叫做 `ripgrep` 的用于搜索文件的 `grep` 的 Rust 实现。为了安装 `ripgrep` 运行如下：

```console
$ cargo install ripgrep
    Updating crates.io index
  Downloaded ripgrep v13.0.0
  Downloaded 1 crate (243.3 KB) in 0.88s
  Installing ripgrep v13.0.0
--snip--
   Compiling ripgrep v13.0.0
    Finished release [optimized + debuginfo] target(s) in 3m 10s
  Installing ~/.cargo/bin/rg
   Installed package `ripgrep v13.0.0` (executable `rg`)
```

最后一行输出展示了安装的二进制文件的位置和名称，在这里 `ripgrep` 被命名为 `rg`。只要你像上面提到的那样将安装目录加入 `$PATH`，就可以运行 `rg --help` 并开始使用一个更快更 Rust 的工具来搜索文件了！