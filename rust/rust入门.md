# Rust 入门

## Linux 安装 Rust

* `curl --proto '=https' --tlsv1.2 https://sh.rustup.rs -sSf | sh` 或者 `brew install rust rustup`
* `apt install gcc`
* 验证: `rustc --version`
* 升级: `rustup update`
* 卸载: `rustup self uninstall`

## Hello World
创建一个文件 hello.rs，定义函数：
```Rust
fn main() {
    println!("Hello, world!");
}
```
Rust 的缩进风格使用 4 个空格，而不是 1 个制表符（tab）。

`println!` 调用了一个 Rust 宏（macro）。如果是调用函数，则应输入 `println` （没有!）。当看到符号 ! 的时候，就意味着**调用的是宏而不是普通函数**，并且宏并不总是遵循与函数相同的规则。

语句以分号结尾（;），这代表一个表达式的结束和下一个表达式的开始。大部分 Rust 代码行以分号结尾。

编译：`rustc hello.rs`