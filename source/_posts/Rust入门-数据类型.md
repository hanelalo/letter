---
title: Rust入门:数据类型
date: 2023-06-10 09:41:45
tags: Rust
categories: Rust
description: 详细了解 rust 的各种数据类型。
---

rust 的数据类型分为基础类型和复合类型。

> 官方文档说的是 Scalar 和 Compound。

其中 Scalar 类型有 4 种：

1. integer，整数。
2. Floating-point numbers，浮点数。
3. Booleans，布尔类型。
4. Characters，字符类型。

> 对应 Java 中的 byte、short、int、long、float、double、boolean、char。

Compound 可以将多个值分为一个类型。rust 原生提供的 Compound 类型有 2 种：

1. Tuples 类型。
2. Arrays 类型。

> 对应 Java 中的数组。

## Integer 类型

Integer 类型分有符号、无符号两大类，每一类又根据内存占用分成 5 种。

| Memory  | Signed | Unsigned |
| ------- | ------ | -------- |
| 8 bit   | i8     | u8       |
| 16 bit  | i16    | u16      |
| 32 bit  | i32    | u32      |
| 64 bit  | i64    | u64      |
| 128 bit | i128   | u128     |
| arch    | isize  | usize    |

其中 arch 一栏，不管是使用 isize 类型还是 usize 类型，占用的内存取决于 rust 的运行环境，如果是在 64 位操作系统环境下，isize、usize 就和 i64、u64 一样，如果是运行在 32 位操作系统环境下，isize、usize 就和 i32、u32 一样。

因为没种类型的内存占用有限制，所以没中类型能够表示的数值大小也是有范围的，如果占用内存是 n bit，则有符号的类型的取值范围是 `-(2^(n-1)) ~ 2^(n-1)-1`，无符号类型的取值范围是 `0 ~ 2^n - 1`。

> 既然有上限，那势必会有溢出的时候，在 rust 中，如果是 debug 模式，integer 类型的变量溢出的话，会终止程序运行，而在 release 模式下，则默认不会终止程序，而是会把取值范围当成一个圈来处理。什么意思呢？比如 `u8` 类型的取值范围是 0 ～ 255，那么当一个 `u8` 类型的变量被赋值位 256 时，实际得到的值为 0，如果赋值为 257，则实际值为 2，以此类推。
>
> ```rust
> fn main() {
>     let mut x: u8 = 255;
>     println!("The value of x is:{x}");
>     x = x + 1;
>     println!("The value of x is:{x}");
>     x = x + 1;
>     println!("The value of x is:{x}");
> }
> ```
>
> 在 debug 模式下运行：
>
> ```bash
> $ cargo run
> The value of x is:255
> thread 'main' panicked at 'attempt to add with overflow', src/main.rs:15:9
> note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
> ```
>
> 在 release 模式下运行：
>
> ```bash
> $ cargo run --release
> The value of x is:255
> The value of x is:0
> The value of x is:1
> ```

Integer 类型的字面值支持十进制、十六进制、八进制、二进制、byte 这 5 种。

| 子面值类型 | 示例          |
| ---------- | ------------- |
| 十进制     | 98_222、98222 |
| 十六进制   | 0xff          |
| 八进制     | 0o77          |
| 二进制     | 0b111         |
| byte       | b'A'          |

其中 byte 只支持 `u8` 类型。

## Floating-Point Numbers 浮点数类型

浮点数类型只有 `f32`、`f64` 两种类型，都是有符号的类型。不明确指定的情况下，默认是 f64 类型，因为现在的 cpu 上，f32 和 f64 性能一样，但是 f64 有更高的精度。 

```rust
fn main() {
	let x = 2.0; // f64
	let y: f32 = 2.0; // f32
}
```

## boolean 类型

当一个变量赋值为 true 或者 false 时，自动就会定义为布尔类型，只不过在 rust 里面布尔类型的类型关键字是 `bool`。

```rust
let x = true; //bool
let x:bool = false; //bool
```

在 rust 中，布尔类型只能是 true 或者 false，只会占用 1 byte 的内存。

## Character 类型

Rust 中，字符类型使用单引号，而字符串使用双引号。

```rust
let x = 'A'; // 字符类型
let y = "A"; // 字符串类型
```

并且，字符类型采用的是 Unicode 编码，换句话说，它的值并不是只在 ASCII 码表的范围内，它可能是一个中文、英文、日文、韩文字符，甚至还能是一个 emoji。

## Tuple 类型

tuple 类型可以包含多个不同类型的值，但是，tuple 类型的长度是固定的，从定义 tuple 类型变量时指定，此后不能修改。

定义一个 tuple 类型的变量的语法如下：

```rust
let tup: (i32,f64,bool) = (16,3.14,true);
let tup = (16,3,14,true);
```

对于 tuple 的定义，也可以不显式指定类型。

在从 tuple 中取数据时，有两种方式。

```rust
let tup: (i32,f64,bool) = (16,3.14,true);
let (quantity, price, paid) = tup;
```

一种是定义相同数量的变量，直接从 tup 中取值，`(quantity, price, paid)` 的括号是必须的。

一种则是按索引的方式取值：

```rust
let tup: (i32,f64,bool) = (16,3.14,true);
let quantity = tup.0;
let price = tup.1;
let paid = tup.2;
println!("Quantity:{quantity},price:{price},Paid:{paid}");
```

和其他编程语言一样，索引下标也是从 0 开始。

## Array 数组类型

array 和 tuple 类似，只不过 array 要求其中的元素必须都是同一个类型，同样，array 的长度也是固定的。

**要定义一个 array 只需要用中括号将元素括起来，每个元素之间用逗号分割即可。**

```rust
let arr = [1, 2, 3, 4, 5];
```

还有明确指定元素的数据类型和 array 的长度的方式：

```rust
let arr:[i32;5] = [1, 2, 3, 4, 5];
```

这里`[i32;5]` 的意思是定义一个元素类型为 i32，长度为 5 的数组。

最后，还有直接指定数组长度和初始化值的定义方式：

```rust
let arr = [3; 5];
```

这里的 `[3; 5]` 的意思是数组中的元素初始值为 3，数组长度为 5，所以这里的 arr 实际的值应该是 `[3, 3, 3, 3, 3]`。

> 到目前位置，讲到的 tuple 和 array 都是长度不可变的，对于需要动态改变长度的数组，rust 提供了 vector 类型，这个在后面会讲到。

然后我们再看看，要如何访问数组中的值。

如果是数组的话，倒是和 Java 一样，用 `arr[i]` 的形式就能随机访问数组中的元素。

```rust
fn main() {
		let arr = [1, 3, 5, 7, 9];
    println!("The first item in arr is:{}",arr[0]);
    println!("The second item in arr is:{}",arr[1]);
}
```

```
The first item in arr is:1
The second item in arr is:3
```
