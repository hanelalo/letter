---
title: Rust入门:结构体
date: 2023-06-19 21:32:03
tags: Rust
categories: Rust
description: 本文将介绍 Rust 中的结构体的概念，结构体就和 Java 中的类比较类似。
---

Rust 的官方文档讲结构体时，说结构体和 tuple 数据类型比较类似，因为都支持将不同类型的数据集合到一起。但区别在于：访问 tuple 中的数据必须按下标来访问，而结构体并不用；结构体可以为这个多种数据类型的集合命名，而 tuple 不行。

因为我是一个 Java 开发者，所以按我的理解，其实 Rust 和 Java 中的类比较类似。

## 定义结构体

在 rust 中，定义结构体的关键字是 `struct`，后面接结构体的名称，然后是一堆大括号，大括号内就是类的字段。

```rust
struct Rectangle {
  width: u32,
  height: u32
}
```

我们还可以通过 struct 的语法将相同的 tuple 数据类型定义成不同的类型：

```rust
struct Color(i32, i32, i32);
struct Point(i32, i32, i32);
```

上面的 `Color` 和 `Point` 虽然都是由 3 个 `i32` 类型的数据组成，但是对应结构体的名称却不一样，如果一个函数的入参是 `Color`，就算 `Point` 和 `Color` 内部维护的数据类型一样，但依然不能将 `Point` 作为参数来传递。

最后就是，还可以像下面这样定义：

```rust
struct AlwaysEquals;
```

是的，这个结构体就只剩下一个名字了，这种结构体在 rust 中叫作 Unit-Like，这种定义方式，在后面讲到自定义 trait 时会细说，这里就不关注了。

还有一点，需要特别注意：常规情况下，结构体中的数据的所有权归结构体的变量所有，所以结构体中的字段类型，不允许是引用类型。

接下来的内容，我们主要会以第一种常规定义方式定义的结构体 `Rectangle` 为例进行。

## 使用结构体

结构体，其实就是一种自定义的复杂数据类型。

如果我们要定义一个 Rectangle 类型的变量，直接以结构体名称开头，后接大括号，大括号内是结构体的字段以 `key: value` 的形式定义，以及最后，有一个分号，表明这是一个完整的语句。

```rust
let rect1  = Rectangle {
	width: 20,
	height: 40
};
```

当然你写成下面这样也行：

```rust
let rect1  = Rectangle {
	height: 40,
  width: 20
};
```

换句话说，在定义各个字段值时，不需要按照结构体定义中的字段顺序来。

这样其实有点麻烦，接下来将介绍几种定义某中结构体的变量时的快捷操作。

### 结构体字段名和变量名一致

当结构体的字段名和变量名一致时，定义解构体是，可以直接省略 `key:value` 中的 `key:` 部分。

```
let height = 20;
let rect1 = Rectangle {
	height,
	width: 20
};
```

### 复制

当我们已经有了一个变量 rect1，现在需要在此基础上新建一个 rect2，你可能会这样做：

```rust
let rect1 = Rectangle {
  width: 30,
  height: 40
};

let rect2 = Rectangle {
  width: rect1.width,
  height: rect1.height
};
```

但其实，你可以这样做：

```rust
let rect1 = Rectangle {
  width: 30,
  height: 40
};

let rect2 = Rectangle {,
  height: 50,
  ..rect1
};
```

这样写的意思是，rect2 的 height 是 50，其余字段取 rect1 的相同字段的值。

等等，这里我们需要确认一件事，那就是结构体里面的数据的所有权问题。如果字段类型是像 `u32`、`f32` 这种基础的标量类型，自然是不用考虑的，因为这种数据是直接赋值，而不是进行所有权的转移或者借用。但如果是一个字符串呢？

我们来做个实验：

```rust
#[derive(Debug)]
struct User {
    username: String,
    email: String,
    age: u32,
}

fn main() {
    let user1 = User {
        username: String::from("hanelalo"),
        email: String::from("hanelalo@163.com"),
        age: 25
    };
    println!("user1: {:?}", user1);
    let user2 = User {
        username: String::from("killer"),
        ..user1
    };
    println!("user2: {:?}", user2);
    println!("user1.username: {:?}", user1.username);
    println!("user1.age: {:?}", user1.age);
    println!("user1.email: {:?}", user1.email);
}
```

```bash
$ cargo run
   Compiling structor-test v0.1.0 (/Users/hanelalo/develop/rust-labs/structor-test)
error[E0382]: borrow of moved value: `user1.email`
  --> src/main.rs:22:35
   |
15 |       let user2 = User {
   |  _________________-
16 | |         username: String::from("killer"),
17 | |         ..user1
18 | |     };
   | |_____- value moved here
...
22 |       println!("user1.email: {:?}", user1.email);
   |                                     ^^^^^^^^^^^ value borrowed here after move
   |
   = note: move occurs because `user1.email` has type `String`, which does not implement the `Copy` trait
   = note: this error originates in the macro `$crate::format_args_nl` which comes from the expansion of the macro `println` (in Nightly builds, run with -Z macro-backtrace for more info)

For more information about this error, try `rustc --explain E0382`.
error: could not compile `structor-test` (bin "structor-test") due to previous error
```

在创建了 user2 之后，我们再尝试访问 user1 的字段，发现编译时，在最后一行访问 `user1.email` 是编译不通过的，因为它的值已经  move 走了。

也就是说，创建 user2 时使用 `..user1` 这种语法，里面的复杂类型的所有权会发生转移，这里是从 user1 转移到 user2。

> 同样，这里不管创建 user1 还是 user2，String 类型的字段的值都是调用 `String::from` 来生成，而不是直接写一个 `"hanelalo"` 这样的字符串字面值，因为字符串字面值的类型是 &str，其实是一个引用。

## 方法

方法和函数类似，定义的语法也基本相同，但是不同在于，方法是依托于结构体而存在的。

还是上面的 `Rectangle` 结构体，如果我现在有一个 Rectangle，我要计算它的面积，可能你会这样写：

```rust
fn main() {
	let rect = Rectangle {
    width: 30,
    height: 50
  };
  println!("The area is {}", area(&rect));
}

fn area(rect: &Rectangle) -> u32 {
  rect.width * rect.height
}
```

但是直觉上，我使用 Rectangle 的时候，我需要它的面积，应该是我我定义的这个 Rectangle 本身有一个 api 能够计算出面积，而不是还需要调用方自己再写一个 area 函数来计算。

> 其实意思就是，按照 Java 面向对象设计的话，计算面积应该是 Rectangle 对象本身所具有的一个行为。

Rust 也支持了为某个结构体定义方法：

```rust
impl Rectangle {
  fn area(&self) -> u32 {
    self.width * self.height
  }
}
```

`impl Rectangle {` 表示这里面都是为 Rectangle 这个结构体定义的方法。

而内部的方法定义的语法倒是和函数定义一致，只不过，可以看见 `area` 方法的参数是 `&self`，这里竟然没指定参数类型，这里可以理解为 `self: &Self`，而 `Self` 就是当前这个 Rectangle 变量本身。因为有 `&` 符号，所以这里是借用关系，不会发生所有权转移。

然后，看看这个方法要如何使用：

```rust
fn main() {
	let rect = Rectangle {
    width: 30,
    height: 50
  };
  println!("The area is {}", rect.area());
}
```

> 如果学过 C++ 或者 Golang，看见 `&` 应该会想起来，在这两种语言中，有写方法只能通过对象引用调用，有些则是通过真正的对象实例调用。而在 Rust 中没这个区别。

在定义结构体的方法时，除了 `self: &Self` 简写成了 `&self` 之外，其他的参数还是需要按照正常语法书写。

### 关联函数

> 在官方文档里面叫作 [Associated Functions](https://doc.rust-lang.org/book/ch05-03-method-syntax.html#associated-functions)。

最明显的区别就是，方法函数里面没有 `&self`，比如常用的 `String::from`。

```rust
impl From<&str> for String {
    #[inline]
    fn from(s: &str) -> String {
        s.to_owned()
    }
}
```

这种方法调用时是通过结构体名称后跟`::`，然后再跟上方法名称的方式进行调用。

比如我们写一个 Rectangle 的构造函数：

```rust
impl Rectangle {
  fn new(width: u32, height: u32) -> Rectangle {
    Rectangle {
      width,
      height
    }
  }
}
```

## 参考文档

* [The Rust Programming Language](https://doc.rust-lang.org/book/ch05-03-method-syntax.html)
