---
title: Rust入门:函数
date: 2023-06-10 22:29:15
tags: Rust
categories: Rust
description: 关于 rust 中函数定义的总结，以及 rust 中 statement、expression 的区别。
---

rust 不同于其他的编程语言，rust 有明确的 statement 和 expression 的概念。本文首先将讲解函数的定义，然后会解释 rust 中 statement 和 expression 的区别，然后进一步的了解函数的返回值的写法相关的知识。

## 函数的定义

在 rust 中，定义函数的关键字是 `fn`，一个函数应包括函数名称、函数参数、函数返回值类型、函数体。

比如多次出现的 main 函数：

```rust
fn main() {
	// something code here
}
```

`fn` 关键字表明当前是定义一个函数，`main` 是函数名称、`()` 里面是函数参数，只不过 main 函数没有参数，`{}` 里面则是函数要执行的内容。

我们再看一个要素比较齐全的函数：

```rust
fn do_something(x:i32, y:i32) -> i32 {
  // function body
}
```

这里 `(x:i32, y:i32)` 表示 `do_something` 这个函数的参数是两个 i32 类型的参数，参数名称分别为 x、y，`-> i32` 则是表示 `do_something` 函数的返回值类型为 i32 类型。

函数的调用倒是和其他常见的语言一样，就不赘述了。

> 之所以不赘述，是因为我觉得 rust 根本就不适合作为编程入门的第一门语言。

## 关于 statement、expression

在 rust 中，有明确的 statment、expression 的定义。

* Statement，是用于说明做了什么事，但并没有返回值。
* Expression，计算得到一个结果值，Expression 是 Statement 的一部分。

是不是有点迷糊？

举个例子：

```rust
let x = 6;
```

这是一个 statement，它做的事是：定义一个变量 x，并将变量 x 赋值为 6，换句话讲，只能说 x 的值是 6，不能说赋值这个动作的结果是 xxx。

```rust
x + 1
```

这是一个 expression，如果 x 等于 6，那么这个 expression 的计算结果就是 7。

## 函数返回值

在 rust 中，按照以往的编程语言的经验，我们定义一个函数可能是这样的：

```rust
fn sum(a:i32, b:i32) -> i32 {
    return a+b;
}
```

这样没问题，但是，会发现下面这两种竟然也能正常返回：

```rust
fn sum(a:i32, b:i32) -> i32 {
    return a+b // 少了最后的分号
}
```

```rust
fn sum(a:i32, b:i32) -> i32 {
    a+b // 不仅分号没了，连 return 关键字都省了
}
```

这是因为在这里 `a+b` 是一个 expression，本身是有返回值的。

> 前面讲到 expression 是有返回值的，所以我个人理解，它还隐含了 `return` 的语义在里面。

那如果我在没有 return 关键字的情况下，在后面加上一个分号呢？

```rust
fn sum(a:i32, b:i32) -> i32 {
    a+b;
}
```

然后 `cargo checkh` 就会有报错：

```
error[E0308]: mismatched types
  --> src/main.rs:33:25
   |
33 | fn sum(a:i32, b:i32) -> i32 {
   |    ---                  ^^^ expected `i32`, found `()`
   |    |
   |    implicitly returns `()` as its body has no tail or `return` expression
34 |     a+b;
   |        - help: remove this semicolon to return this value

For more information about this error, try `rustc --explain E0308`.
```

大概意思反正就是这里的需要一个 return 的 expression。

那么，就可以认为，当不加最后的分号时，`a+b`是一个 expression，本身有 return 语义存在，而价格分号之后，`a+b;` 变成了 statement，没有返回值了，也就没有 return 的语义在里面了，所以才会编译不通过。

> 关于 statement、expression 这两个其实还是有点让人迷糊的，但是只要记住 rust 中这种函数 return 的写法就行了。

## 参考文档

* [The Rust Programming Language](https://doc.rust-lang.org/book/ch03-03-how-functions-work.html)
