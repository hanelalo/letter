---
title: Rust入门:变量和常量
date: 2023-06-09 22:35:13
tags: Rust
categories: Rust
description: 初步了解 rust 语言中的变量和常量。
---

## Rust 介绍

rust 是一种内存安全的、无内存 GC 的编程语言。在性能上远超 Java，甚至比 C++ 还要快，和 C 语言比起来虽然有差距但是并不大。

rust 没有像 Java 这种语言的 GC 机制， Java 是异步回收，最明显的弊端就是 Stop The World 的强制停顿，虽然近年来的 G1、ZGC 等垃圾收集器有明显改善，但依然有停顿。而 rust 也是存在内存释放的机制的，等到后续更深入的学习再详细讲解。

rust 虽然有优点，比如安全，甚至能保证只要编译通过，代码基本就不会有安全问题。相应的，学习曲线比较陡峭，比如其中的所有权、不可变的变量等机制更是让人难以掌握。

> 如果你发现自己学会了 rust，那么大概率还处于入门都还算不上的阶段。

## 安装 Rust

如果是在 MacOS 或者 Linux 上，可以使用下面的脚本进行安装：

```bash
$ curl --proto '=https' --tlsv1.2 https://sh.rustup.rs -sSf | sh
```

安装完成后，会有下面一行输出：

```
Rust is installed now. Great!
```

这里其实还安装了一个叫 `rustup` 的工具。`rustup` 是一个用来管理 rust 版本的命令行工具。后续可以使用 `rustup update` 命令来升级 rust 的版本。

然后执行下面的指令，检查 rust 是否安装成功。

```bash
$ rustc --version
```

然后再检查一下 rust 的包管理器 cargo 是否安装成功。

``` bash
$ cargo --version
```

## 从"Hello World"开始

Rust 的源代码文件是 `.rs` 格式的，和 Java 类似，有一个 main 函数作为整个程序的入口。

```rust
// main.rs
fn main() {
	println!("Hello World!");
}
```

保存了 main.rs 之后，执行 `rustc main.rs`，会发现出现了一个 main 文件：

```bash
$ rustc main.rs
$ ls
main	main.rs
```

这里的 main 文件是一个可执行文件，直接执行看看。

```bash
$ ./main
Hello World
```

像这样单个文件还行，但是如果是要真正做一个 rust 开发的项目的话，势必会有很多包以来，所以一般都是使用 cargo，而不是 rustc。

使用 cargo 的话，首先得通过 cargo 新建一个项目。

```bash
$ cargo new hello_world
```

然后进入项目会发现有以下目录结构：

```
hello_world\
	src\
		main.rs
	Cargo.toml
```

src 是源代码目录，Cargo.toml 是项目的配置文件，其中包括项目的一些元数据，以及项目要依赖的第三方包（暂且叫包吧，其实在 rust 里面不叫包）。

```toml
# Cargo.toml
[package]
name = "hello_world"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
```

接下来会介绍 3 个 cargo 的命令：

* cargo build

  构建项目，这时会导入项目的依赖，并生成一个 target 文件夹，在 target/debug 文件夹中有一个和项目同名的可执行文件，同时，还会生成一个 Cargo.lock 文件，这个文件用于稳定项目的依赖，因为 cargo 支持语意化的版本管理，比如 `^0.8.0` 这种写法，并不是指定版本是 0.8.0，而是版本至少是 0.8.0，但必须小于 0.9.0，而项目依赖的版本将在第一次 build 时确定，为了保证稳定性，依赖的具体版本讲通过 Cargo.lock 文件进行锁定，后面不管重新 build 多少次，版本都不会变。

  普通的 cargo build 是开发时使用的，如果要编译生成 release 版本，应该之中 `cargo build --release` 或者 `cargo build -r`，会在项目目录下生成 release 文件夹。

* cargo check

  cargo check 则是检查你的代码是不是可执行的，并不会生成可执行文件。

* cargo run

  编译代码，并执行，相当于执行 `cargo build` 之后，再执行生成的可执行文件。

## 变量

### 变量不能变？

先看一个例子：

```rust
fn main() {
	let x = 5;
	println!("The value of x is:{x}");
	x = 6;
	println!("The value of x is:{x}");
}
```

这个时候会编译失败。

```
error[E0384]: cannot assign twice to immutable variable `x`
 --> src/main.rs:4:5
  |
2 |     let x = 5;
  |         -
  |         |
  |         first assignment to `x`
  |         help: consider making this binding mutable: `mut x`
3 |     println!("The value of x is:{x}");
4 |     x = 6;
  |     ^^^^^ cannot assign twice to immutable variable

For more information about this error, try `rustc --explain E0384`.
```

大概意思是不能为不可变的变量赋值两次。

在 rust 里面，变量默认是不可变的，换句话说，变量一旦赋值，就不能再重新赋值。

rust 提供了 `mut` 关键字来标识某个变量是可变的。

```rust
fn main() {
	let mut x = 5;
	println!("The value of x is:{x}");
	x = 6;
	println!("The value of x is:{x}");
}
```

这样，编译就不会报错了。

### 一段代码里面还能有同名的变量？

```rust
fn main() {
	let x = 5;
	println!("The value of x is:{x}");
	let x = "Hello";
	println!("The value of x is:{x}");
}
```

上面的代码先定义一个变量 x，赋值为 5，然后又重新定义了变量 x，赋值为 "Hello World"，按照以往的编程的习惯，这里应该会报错才对，但是在 rust 中依然能编译成功。

这里不仅是名字一样，变量的数据类型也从 i32 变成了 String 类型。rust 里面把这种机制叫做 shadow。个人理解应是遮蔽的意思，就是第二次定义的变量将第一次定义的变量遮住了，所以后续使用时只能看见第二次定义的变量。

> 我觉得这样做能减缓一点变量命名困难的病。。。

在 rust 中，变量也有scope 的概念，即作用域，而 shadow 机制还会受到 scope 的影响。

```rust
fn main() {
    let mut x = 5;
    println!("The value of x is:{x}");
    x = x + 1;
    println!("The value of x is:{x}");
    let x = x + 1;
    println!("The value of x is:{x}");
    {
        let x = x * 2;
        println!("The value of inner x is:{x}");
    }
    println!("The value of x is:{x}");
}
```

我们看看输出：

```bash
$ cargo run
   Compiling variables v0.1.0 (/Users/hanelalo/develop/rust-labs/variables)
    Finished dev [unoptimized + debuginfo] target(s) in 0.14s
     Running `target/debug/variables`
The value of x is:5
The value of x is:6
The value of x is:7
The value of inner x is:14
The value of x is:7
```

可以看见的是，对于变量的 shadow 只在当前的 scope 中有效，从当前的 scope 离开后，就不存在 shadow 了。

所以，最后打印出来的 x 的值依然是 7，而不是 14。

## 常量

常量和不可变的变量有点像，但也有些区别：

* 常量不允许使用 mut 关键字，因为常量是任何情况下都不可变，并不是像变量那样默认不可变。
* 常量使用 const 关键字定义，而不是 let。
* 常量必须明确指定数据类型。

前面两点比较好理解，针对第 3 点给个例子，比如变量的定义可以是 `let x = 5`，会自动识别成 `i32` 的数据类型，或者也可以明确指定类型 `let x:i32 = 5`，换句话说，定义变量时，数据类型可以不指定，由 rust 自行判断类型，而定义常量则不行，必须指定类型：`const MAX_ALLOW_REQUEST_PER_SECOND:i32 = 10000`。

## 参考文档

* [The Rust Programming Language](https://rust-book.cs.brown.edu/ch01-01-installation.html)

* [语义化版本](https://semver.org/lang/zh-CN/)
