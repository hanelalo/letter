---
title: Rust入门:引用和借用
date: 2023-06-17 21:43:18
tags: Rust
categories: 文档翻译
description: 本文将介绍 rust 中引用和借用的概念。同样本文依然是官方文档的中文翻译。
---

> 本文是 Rust 官方文档的翻译，原文：https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html

在代码清单 4-5 中，我们调用 `calculate_length`，该函数返回了一个 String 类型，我们调用完这个函数之后，依然还能使用它返回的 String 类型的变量，这是因为这个 String 变量的所有权发生了转移。我们还能提供一个 String 类型的变量的引用，引用就想一个指向某个内存地址的指针，我们可以通过引用访问到这个内存地址上的数据，而这个内存地址上的数据，有可能是属于其他变量的；和指针不同的是，引用在引用的生命周期内保证指向的一定是有效的数据。

下面的代码展示了你要如何以传递一个引用作为函数参数的方式调用 `calculate_length` 函数：

文件名：src/main.rs

```rust
fn main() {
    let s1 = String::from("hello");
    let len = calculate_length(&s1);
    println!("The length of '{}' is {}.", s1, len);
}

fn calculate_length(s: &String) -> usize {
    s.len()
}
```

首先，注意这里所有的 tuple 相关的代码，包括返回值使用 tuple 的代码，都已经删除。

然后，需要注意调用 `calculate_length` 时，传入的参数是 `&s1`，并且在其函数的定义中，参数类型是 `&String` 而不是 `String`，这种写法就是*引用*，它允许你使用某些值，但并不需要拥有这个值的所有权。图 4-5 描绘了这种概念。

![图 4-5：&String s 指向 String s1](http://image.hanelalo.cn/image/202306162221740.png)

图 4-5：`&String s` 指向 `String s1`



> 注意：与引用相反的是取消引用，通过 `*` 操作符就能实现取消引用，我们会在第 8 章看见一些对 `*` 的使用，会在第 15 章详细讨论取消引用。

让我们仔细看看这段代码：

```rust
let s1 = String::from("hello");
let len = calculate_length(&s1);
```

`&s1` 这种写法，会创建一个引用，指向 s1 的值，但是并不会发生所有权的变化。因为这里没有发生所有权的变化，所以当这个这个引用超出作用域时，并不会调用 drop，也不会释放内存。

同样，使用 `&` 这个标识符号，标识这里的变量类型是某个数据类型的引用类型。

```rust
fn calculate_length(s: &String) -> usize {// s 是一个 String 的引用
  s.len()
}// s 超出作用域，但是因为 s 是引用类型，没有相应数据的所有权，所以不会释放内存
```

s 的作用域与函数的任何参数的有效作用域范围一样，只不过当 s 不再使用时，它指向的值不会被清除，因为 s 没有数据的所有权。当函数的参数是变量的引用，而不是变量真正的值时，我们不需要在函数中返回变量的值来归还所有权，因为从来没拥有过变量值的所有权。

我们把这种创建一个引用的操作叫作借用（原文：borowwing）。在真实生活中，一个人有某中东西，你可以找他接，用完后你还得还回去，但你从来没有拥有过这个东西。

所以，如果我们对借用的数据进行修改会怎样？试试代码清单 4-6，其实它会报错。

```rust
fn main() {
    let s = String::from("hello");
    change(&s);
}

fn change(some_string: &String) {
    some_string.push_str(",world");
}
```

代码清单 4-6：尝试修改借用的值



```bash
$ cargo run
   Compiling ownership v0.1.0 (file:///projects/ownership)
error[E0596]: cannot borrow `*some_string` as mutable, as it is behind a `&` reference
 --> src/main.rs:8:5
  |
7 | fn change(some_string: &String) {
  |                        ------- help: consider changing this to be a mutable reference: `&mut String`
8 |     some_string.push_str(", world");
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ `some_string` is a `&` reference, so the data it refers to cannot be borrowed as mutable

For more information about this error, try `rustc --explain E0596`.
error: could not compile `ownership` due to previous error
```

和变量默认就是不可变的，引用也是一样默认不可变的。我们并不允许修改引用的数据。

## 可变引用

我们可以修改代码清单 4-6，通过使用可变引用，就能修改借用的值。

```rust
fn main() {
    let mut s = String::from("hello");
    change(&mut s);
}

fn change(some_string: &mut String) {
    some_string.push_str(",world");
}
```

首先，修改变量 s 为可变的。然后，我们通过 `&mut s` 创建一个可变引用，并作为调用 `change` 函数的参数。修改 `change` 函数的参数裂变，接受一个可变引用 `some_string: &mut String`。这很明确的表示，函数 `change` 很可能使其借用的值发生变化。

可变引用有一个很重要的强制限制：如果一个变量已经有一个可变引用，那它将不能再有任何其他引用。下面的代码尝试为变量 `s`  创建两个可变的引用，这明显会编译失败：

文件名：src/main.rs

```rust
let mut s = String::from("hello");

let r1 = &mut s;
let r2 = &mut s;

println!("{}, {}", r1, r2);
```

```
$ cargo run
   Compiling ownership v0.1.0 (file:///projects/ownership)
error[E0499]: cannot borrow `s` as mutable more than once at a time
 --> src/main.rs:5:14
  |
4 |     let r1 = &mut s;
  |              ------ first mutable borrow occurs here
5 |     let r2 = &mut s;
  |              ^^^^^^ second mutable borrow occurs here
6 |
7 |     println!("{}, {}", r1, r2);
  |                        -- first borrow later used here

For more information about this error, try `rustc --explain E0499`.
error: could not compile `ownership` due to previous error
```

这里报错就是因为我们在一个作用域中不能对 s 有两个可变引用。

s 的第一个可变引用 r1 知道最后调用 println 都依然有效，但是在创建这个引用到使用这个引用之间，有定义了一个 s 的可变引用 r2。

总之，Rust 允许你对引用的数据作出更改，但是增加了限制，以保证这种改变是可控的。这是大多数 Rust 开发者难以理解的问题，因为大多数语言都允许你随意改变。有这个限制的好处在于，Rust 能在编译时就就发现数据竞争问题。当出现以下三种情况时就会发生数据竞争：

1. 两个或以上的指针在同一时间访问同一数据。
2. 至少一个指针在写这些数据。
3. 没有任何用于同步访问数据的机制。

数据竞争会导致发生一些无法预测的事，而在运行时去发现并修复这个问题，也是比较困难的。而 Rust 解决这个问题的方案则是在编译时如果发现了这种问题，就编译失败。

同样，我还可以通过大括号 `{` 创建一个新的作用域，来实现对变量的多个可变引用。

```rust
let mut s = String::from("hello");

{
	let r1 = &mut s;
} // r1 在这里超出了作用域，所有在此之后，我们可以再创建一个新的可变引用

let r2 = &mut s;
```

Rust 还有一个关于同事使用可变引用和不可变引用的强制规则，且先看看下面的代码：

```rust
    let mut s = String::from("hello");

    let r1 = &s; // no problem
    let r2 = &s; // no problem
    let r3 = &mut s; // BIG PROBLEM

    println!("{}, {}, and {}", r1, r2, r3);
```

这里的报错：

```
$ cargo run
   Compiling ownership v0.1.0 (file:///projects/ownership)
error[E0502]: cannot borrow `s` as mutable because it is also borrowed as immutable
 --> src/main.rs:6:14
  |
4 |     let r1 = &s; // no problem
  |              -- immutable borrow occurs here
5 |     let r2 = &s; // no problem
6 |     let r3 = &mut s; // BIG PROBLEM
  |              ^^^^^^ mutable borrow occurs here
7 |
8 |     println!("{}, {}, and {}", r1, r2, r3);
  |                                -- immutable borrow later used here

For more information about this error, try `rustc --explain E0502`.
error: could not compile `ownership` due to previous error
```

当作用域内对某个变量值有一个不可变引用时，不能再有一个可变引用。

因为对于不可变引用来讲，并不希望突然发现引用的数据变了。然后，Rust 是允许对同一个变量有多个不可变引用的，因为这些不可变引用没有任何一个引用会对数据进行修改。

需要注意引用的作用域是从引用定义时，一直贯穿到最后一次使用。举个例子，下面的代码中之所以能编译，是因为不可变引用的最后一次使用在 `println!`，并且发生在定义可变引用之前：

```rust
    let mut s = String::from("hello");

    let r1 = &s; 
    let r2 = &s; 
    println!("{}, {}", r1, r2);

    let r3 = &mut s;
    println!("{}", r3);
```

不可变引用 `r1`、`r2` 在调用 `println!` 之后就无效了，因为这是它们最后一次被使用，而这发生在 `r3` 声明之前，这里没有作用域的重叠，所以才能编译通过。形象点讲，编译器可以在作用域结束前的某个点就知道引用不会再使用。

尽管借用产生的错误比较烦，但是记住这个是 Rust 让你能在编译时而不是运行时就能发现潜在的 bug，并告诉你问题在哪里，而你也不用在运行时才来找寻数据的问题到底在哪里。

## 悬垂引用

在具有指针的编程语言中，很容易错误地创建悬垂指针（dangling pointer）——指向内存中可能已经分配给其他对象的位置的指针——这是因为在释放一些内存时仍然保留指向该内存的指针。然而，在 Rust 中，编译器保证引用永远不会有悬垂引用：如果你拥有对某个数据的引用，编译器将确保在引用消失之前数据不会超出作用域。这为避免悬垂引用提供了安全保证。

让我们看一下当尝试创建一个悬垂引用时，Rust 在编译时会如何处理。

文件名：src/main.rs

```rust
fn main() {
    let reference_to_nothing = dangle();
}

fn dangle() -> &String {
    let s = String::from("hello");
  
    &s
}
```

```
$ cargo run
   Compiling ownership v0.1.0 (file:///projects/ownership)
error[E0106]: missing lifetime specifier
 --> src/main.rs:5:16
  |
5 | fn dangle() -> &String {
  |                ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but there is no value for it to be borrowed from
help: consider using the `'static` lifetime
  |
5 | fn dangle() -> &'static String {
  |                 +++++++

For more information about this error, try `rustc --explain E0106`.
error: could not compile `ownership` due to previous error
```

这里的报错提到了 Rust 的另一个特性：生命周期。将在[第十章](https://doc.rust-lang.org/book/ch10-00-generics.html)谈论这个特性。

这里的错误消息里面也明确给出了编译不通过的原因：

```
help: this function's return type contains a borrowed value, but there is no value for it to be borrowed from
```

我们在代码中加一些注释来帮助理解一下 `dangle` 函数：

```rust
fn dangle() -> &String { // dangle 返回一个 String 的引用
    let s = String::from("hello"); // s 是一个新的 String 变量
  
    &s // 我们返回了一个变量 s 的引用
} // s 超出作用域，并且调用 drop，自动释放内存
```

因为 `s` 是在函数 `dangle` 内创建的，当 `dangle` 函数执行完成后，`s` 的内存将被释放。但是我们尝试返回一个 `s` 的引用，这意味着返回的这个引用指向的是无效的 String。这并不可行，Rust 建议这个函数这样写：

```rust
fn no_dangle() -> String {
	let s = String::from("hello");
	
	s
}
```

这样就没问题了，返回时 s 的所有权会转移，当 `no_dangle` 执行完后，s 的内存也不会被释放。

## 引用的规则

让我们复习一下以上关于引用的讨论：

* 在任何时间点，你只能在声明一个可变引用和同时声明多个不可变引用之间二选一。
* 引用必须保证始终有效。

接下来，我们将讨论一种不同的引用：切片。
