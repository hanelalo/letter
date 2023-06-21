---
title: Rust入门:枚举和模式匹配
date: 2023-06-21 14:17:40
tags: Rust
categories: Rust
---



枚举是一种在很多编程语言中都有类型，在 Rust 中，枚举还有相应的一些模式匹配的用法在里面，本文将简单介绍一下枚举的定义，然后再稍微详细地讨论模式匹配。

## 枚举

和 Java 一样，Rust 中定义枚举也是使用 `enum` 关键字：

```rust
enum IpAddrKind {
  V4,
  V6
}
```

使用的时候，直接当成普通的结构体调用：

```rust
let ipv4 = IpAddrKind::V4;
let ipv6 = IpAddrKind::V6;
```

```rust
fn route(ip_kind: IpAddrKind) {}
```

枚举里面同样还可以再包含变量，甚至同一个枚举类型不同的值，包含的变量类型也可以不一样：

```rust
enum Message {
  Quit,
  Move{x: i32, y: i32},
  Write(String),
  ChangeColor(i32, i32, i32)
}
```

这里可能会想到为什么不直接定义 4 个下面这样的结构体：

```rust
struct Quit;
struct Move {x: i32, y: i32};
struct Write(content: String);
struct ChangeColor(r: i32, g: i32, b: i32);
```

想象有一个函数（或方法）的返回值，可能是返回这 4 中数据类型中的一种，我们没办法让一个函数（或方法）的返回值类型有可能有 4 种，而如果我们使用的是 Message 枚举的话，那么我们的函数（或方法）的返回值类型就可以是 Message，内部实现上就能返回枚举的任意一个类型。

在结构体里面，我们是可以为结构体定义方法的，同样，也可以为枚举定义方法，甚至语法也是一样的。

## 模式匹配

在 rust 中有一种控制结构，用于根据给定的模式对值进行匹配，并执行相关联的代码逻辑，叫作 `match`。这和 if 结构有些类似，但是不一样的是 if 必须有一个返回 bool 的表达式，而 match 支持字面值、变量名、通配符等很多类型。

下面是官方文档的一个例子：

```rust
enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter,
}

fn value_in_cents(coin: Coin) -> u8 {
    match coin {
        Coin::Penny => 1,
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter => 25,
    }
}
```

这里我们主要观察 match 的语法：

1. 首先是关键字 match，后面是给定的要进行匹配的值。
2. 然后就是要 match 的分支，都用大括号括起来。
3. 每个分支之间用都好分开。
4. 每个分支有两个部分，一个匹配的模式，比如 `Coin::Penny`，然后是要执行的代码逻辑，代码逻辑部分如果比较复杂，可以用大括号括起来，如果比较简单，则不需要。这两部分之间通过 `=>` 连接起来。

当 match 开始执行时，会比对给定的值和每个分支，看是否匹配，如果匹配就执行相关联的代码，并且，只会执行第一个匹配上的分支代码。

上面的例子，主要是为了结合枚举使用模式匹配，其实我个人认为还不能理解为什么叫模式匹配，因为前面我们讲到过模式甚至可以是一个通配符。比如下面这例子：

```rust
fn main() {
    let number = 5;

    match number {
        1 => println!("One"),
        2 => println!("Two"),
        3 | 4 => println!("Three or Four"),
        5..=10 => println!("Five to Ten"),
        _ => println!("Other"),
    }
}
```

这里分支模式有些甚至不是一个固定的值，可能是一个范围值，这样是不是更能贴切得把它叫作模式了？

> 站在 java 开发者的角度，match 感觉更像是结合了 if 和 switch 两种结构。

### `Option<T>` 的使用

接下来，看一个 Rust 中比较常用的模式匹配 `Option<T>`。

```rust
fn main() {
    let five = Some(5);
    let six = match plus_one(five) {
        None => -1,
        Some(i) => i
    };
    println!("six is {}", six);
    let none = match plus_one(None) {
        None => -1,
        Some(i) => i
    };
    println!("none is {}", none);
}

fn plus_one(x: Option<i32>) -> Option<i32> {
    match x {
        None => None,
        Some(i) => Some(i + 1),
    }
}
```

```bash
$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.00s
     Running `target/debug/match-pattern`
six is 6
none is -1
```

`plus_one` 函数接收一个 `Option<i32>` 类型的参数。Option 是 Rust 中常用的一个枚举，包含 `None`、`Some(T)` 两种值，其中 T 是泛型，以后学了再说。如果传入的 `Option<i32>` 类型变量是 `Some` 就将其中的 i32 类型的值加一，并用 `Some` 进行封装，即`Some(i+1)`，然后返回，如果是 `None`，就直接也返回一个 `None`。

而在 main 函数中也对返回值根据不同情况进行取值，如果是 `None` 就返回 -1，否则就返回 `Some` 中的值。

### 模式匹配中的默认处理逻辑

在模式匹配中，如果分支覆盖的范围不全会编译失败，比如下面的代码，我们只匹配了 1 到 10 的数字的情况，其他的没处理：

```rust
fn main() {
    let number = 5;

    match number {
        1 => println!("One"),
        2 => println!("Two"),
        3 | 4 => println!("Three or Four"),
        5..=10 => println!("Five to Ten"),
    }
}
```

```bash
$ cargo run                                                                       ok  22:11:25 
    Finished dev [unoptimized + debuginfo] target(s) in 0.00s
     Running `target/debug/match-pattern`
six is 6
none is -1

 ~/develop/rust-labs/match-pattern  master ?4  cargo run                                                                       ok  22:12:47 
   Compiling match-pattern v0.1.0 (/Users/hanelalo/develop/rust-labs/match-pattern)
error[E0004]: non-exhaustive patterns: `i32::MIN..=0_i32` and `11_i32..=i32::MAX` not covered
 --> src/main.rs:4:11
  |
4 |     match number {
  |           ^^^^^^ patterns `i32::MIN..=0_i32` and `11_i32..=i32::MAX` not covered
  |
  = note: the matched value is of type `i32`
help: ensure that all possible cases are being handled by adding a match arm with a wildcard pattern, a match arm with multiple or-patterns as shown, or multiple match arms
  |
8 ~         5..=10 => println!("Five to Ten"),
9 ~         i32::MIN..=0_i32 | 11_i32..=i32::MAX => todo!(),
  |

For more information about this error, try `rustc --explain E0004`.
```

会发现这里会编译失败，因为并没有处理所有的情况，当 number 大于 10 或者小于 1 时要如何处理？

而要处理剩余的所有情况，模式可以直接写成 `_`。

```rust
fn main() {
    let number = 5;

    match number {
        1 => println!("One"),
        2 => println!("Two"),
        3 | 4 => println!("Three or Four"),
        5..=10 => println!("Five to Ten"),
        _ => println!("Other"),
    }
}
```

### if let 语法

在前面讲到 match 会要求你的模式必须包含所有的可能，但有时可能我们并不需要写得如此复杂，所以 rust 提供了一种特殊的 match 语法糖，那就是 `if let` 模式匹配，基本的语法结构如下：

```
if let pattern = value {

} else {

}
```

当然，这里的 else 也可以不要。

实例代码：

```rust
fn main() {
    let number = Some(5);

    if let Some(value) = number {
        println!("Matched: {}", value);
    } else {
        println!("Not matched");
    }
}
```

> 但我觉得这种写法，太生涩了，虽然代码是简单很多，但是看着不舒服，太过于反常识。

## 参考文档

* [Enums and Pattern Matching](https://doc.rust-lang.org/book/ch06-00-enums.html#enums-and-pattern-matching)
