---
title: Rust入门:切片类型
date: 2023-06-17 21:45:15
tags: Rust
categories: 文档翻译
description: 本文是 Rust 官方文档翻译，主要在所有权的角度介绍切片类型。
---

> 本文是 Rust 官方文档的翻译，原文：https://doc.rust-lang.org/book/ch04-03-slices.html

## 切片类型

> 译者注：相信使用过 python 的对这个概念并不陌生，这里切片的原文是 slice。本文中切片和 slice 是等价的。

切片允许你引用一个集合中的连续序列，而不是整个集合。因为切片本就是一种引用，所以它没有所有权。

这里有一个小问题：写一个函数，将字符串按空格分开，找到并返回它的第一个单词，如果找不到空格，那这个字符串必是一个单词，所以要返回整个字符串。

让我们先以不使用切片的方式解决这个问题，方便更深入理解切片的意义。

```rust
fn first_word(s: &String) -> ?
```

`first_word` 函数的参数是 `&String`，我们并不想要转移所有权，所以这没问题。但是这个函数的返回值类型应该是什么？

我们目前没办法只去操作字符串的一部分，但是我们可以返回以空格分隔之后的单词的索引。

```rust
fn first_word(s: &String) -> usize {
  let bytes = s.as_bytes();
  
  for (i, &item) in bytes.iter().enumerate() {
    if item == b' ' {
      return i;
    }
  }
  
  s.len()
}
```

代码清单 4-7：first_word 返回一个 String 中的索引值



因为我们需要逐个便利字符串的每一个字符，确定是不是空格，我们会使用 `as_bytes` 方法将字符串转换成一个 byte 数组。

```rust
let bytes = s.as_bytes();
```

然后，我们通过 `iter` 方法为这个数字创建一个迭代器：

```rust
for (i, &item) in bytes.iter().enumerate() {
```

更详细的关于迭代起的讨论在[第十三章](https://doc.rust-lang.org/book/ch13-02-iterators.html)。现在我们只需要知道 `iter` 方法返回集合中的每一个元素，而 `enumerate` 方法将其返回的元素使用 tuple 进行封装，以 tuple 的方式返回每个元素。tuple 中的第一个元素是在 bytes 中的索引值，tuple 的第二个元素是对于 bytes 中索引值对应的数据的引用。

因为 `enumerate` 方法返回一个 tuple，我们可以使用模式来解构这个 tuple。我们会在[第六章](https://doc.rust-lang.org/book/ch06-02-match.html#patterns-that-bind-to-values)讨论*模式*。而在 for 这种循环结构里，我们指定的模式是，`i` 是 tuple 中的索引，`&item` 是 tuple 中单个字节。因为我们从 `.iter().enumerate()` 中获取的是引用，所以这里使用了 `&`。

在 for 循环中，我们以二进制的方式搜索空格，如果找到了空格，就返回当前的索引值 `i`，如果最终没找到，就通过 `s.len()` 返回字符串的长度。

```rust
      if item == b' ' {
        return i;
      }
    }
    
    s.len()
```

我们现在有办法找到字符串中的第一个单词结尾的索引，但是还有一个问题，我们只是返回了一个 `usize` 值，但它仅仅只是在 `&String` 这个上下文中有意义的数字而已。换句话说，因为它是独立于 `String` 的一个值，并不能保证它未来也有效。所以可以考虑像代码清单 4-8 这样使用代码清单 4-7 中的 `first_word` 函数。

```rust
fn main() {
    let mut s = String::from("hello world");

    let word = first_word(&s); // word 的值为 5

    s.clear(); // 这会清空字符串，让它等于 ""
		// word 的值一直会是 5，但是没有其他关于字符串的任何东西了
  	// 我们可以继续使用这个有意义的 5，但是现在完全不知道这个单词是啥
}
```

代码清单 4-8: 保存 `first_word` 的返回值并清空字符串



必须考虑 `word` 中的索引与 `s` 中的数据不同步的问题，这是是一件复杂且容易出错的事情！如果编写一个 `second_word` 函数，管理这些索引会变得更加脆弱。其函数签名应该如下所示：

```rust
fn second_word(s: &String) -> (usize, usize) {
```

现在，我们需要跟踪起始索引和结束索引，我们有更多的值是从特定状态的数据计算出来的，但它们与该状态没有任何关联。我们有三个不相关的变量在周围浮动，需要保持同步。

幸运的是，Rust 对这个问题有一个解决方案：字符串切片（string slices）。

### 字符串切片

字符串切片是对字符串的一部分的引用，看起来就像这样：

```rust
let s = String::from("hello world");

let hello = &s[0..5];
let world = &s[6..11];
```

这里并不是引用整个字符串，变量 `hello` 只是引用了字符串的一部分，引用的部分通过 `[0..5]` 来提取。我们通过`[string_index..end_index]` 的形式来圈定创建切片的范围，`start_index` 是切片在字符串中的起始位置，`end_index` 则是比切片的最后一个位置多一位（译者注：其实就是 start_index、end_index 都是索引，而且是前闭后开的区间）。在切片内部存储着切片的起始位置，以及切片的长度（即 end_index - start_index）。所以在 `let world = &s[6..11];` 中，`world` 变量是一个切片，这个切片包含一个指针指向索引为 6 的位置，并且记录长度为 5。

图 4-6 解释了这种切片结构。

![图 4-6：引用部分字符串的字符串切片](http://image.hanelalo.cn/image/202306172255528.png)

图 4-6：引用部分字符串的字符串切片



使用 Rust 中的范围语法 `..` 时，如果 start_index 想从 0 开始，则可以删除 `..` 前面的数字。简单点讲，下面这两种切片的写法是一样的：

```rust
let s = String::from("hello");

let slice = &s[0..2];
let slice = &s[..2];
```

同理，如果你的切片要包含字符串的最后一个字符，你可以将 `..` 后面的数字删掉，也就是说，下面这两种写法是等价的：

```rust
let s = Sring::from("hello");

let len = s.len();

let slice = &s[3..len];
let slice = &s[3..];
```

如果将 `..` 前后的数字都删除，则表示这个切片引用的是整个字符串，所以下面这两种写法是等价的：

```rust
let s = Sring::from("hello");

let len = s.len();

let slice = &s[0..len];
let slice = &s[..];
```

> 注意：字符串切片范围必须出现在有效的 utf-8 的字符边界处。如果你想在某个字节中间进行字符串切片操作，程序将因为错误而退出。为了方便介绍字符串切片，本章的仅假设字符串在 ASCII 编码范围内。更多关于 UTF-8 的处理将在[第八章](https://doc.rust-lang.org/book/ch08-02-strings.html#storing-utf-8-encoded-text-with-strings)进行讨论。

结合以上所学，我们重新实现 `first_word` 函数，并让它返回一个 slice，在返回值类型这里，字符串切片类型的写法是 `&str`：

```rust
fn first_word(s: &String) -> &str {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[..i];
        }
    }
    &s[..]
}
```

我们使用和代码清单 4-7 相同的方法得到了第一个单词的默认索引，当我们找到一个空格时，我们返回一个切片，切片引用范围是从 0 到这个空格的索引。现在我们调用 `first_word`，得到的是一个引用了从字符串的索引 0 开始到第一个空格之间的数据的切片。

这种返回切片的方式，同样适用于 `second_word` 函数。

```rust
fn second_word(s: &String) -> &str {
```

现在我们有一个明确的获取第一个单词的 api，就更加不容易出错了，并且编译器会保证这个切片引用不会是无效引用。还记得代码清单 4-8 中的 bug 吗？它在获取了第一个单词末尾的索引下标后，将字符串清空了，此时我们拿到的索引也就没用了，这是一个 bug 呀，但是依然编译通过了。而使用新版本的 `first_word` 则会在编译时就报错：

```rust
fn main() {
    let mut s = String::from("hello world");

    let word = first_word(&s);

    s.clear(); // error!

    println!("the first word is: {}", word);
}
```

```
$ cargo run
   Compiling ownership v0.1.0 (file:///projects/ownership)
error[E0502]: cannot borrow `s` as mutable because it is also borrowed as immutable
  --> src/main.rs:18:5
   |
16 |     let word = first_word(&s);
   |                           -- immutable borrow occurs here
17 |
18 |     s.clear(); // error!
   |     ^^^^^^^^^ mutable borrow occurs here
19 |
20 |     println!("the first word is: {}", word);
   |                                       ---- immutable borrow later used here

For more information about this error, try `rustc --explain E0502`.
error: could not compile `ownership` due to previous error
```

回忆借用的规则：同一作用域中，有一个不可变引用之后，就不能在有可变引用。因为 `clear` 方法会清空字符串，它需要拿到一个可变引用，而 `println!` 在调用 `clear` 方法之后还在使用 word 变量，所以 word 在这一刻都依然有效。Rust 不允许 `clear` 中的可变引用和 `word` 这个不可变引用同时存在，所以会编译失败。Rust 不仅让 api 更易用，它还在编译时消除了一整类错误。

### 字符串字面值的切片

回忆一下字符串字面值，它是硬编码到了二进制文件中。现在我们学习了切片，我们也可以正确理解字符串字面值的切片了。

```rust
let s = "Hello, world!";
```

这里的 `s` 的类型是 `&str`，它是一个指向二进制文件中特定位置的切片。这也是为什么字符串字面值是不可变的，因为 `&str` 是一个不可变引用。

### 字符串切片作为参数

在学习了字符串和字符串字面值的切片之后，我们对 `first_word` 又进行了一次优化，这是它的函数签名：

```rust
fn first_word(s: &String) -> &str {
```

很多 Rust 开发者都会将函数签名写成代码清单 4-9 这样，因为这样写，参数能支持 `&String` 和 `&str` 两种。

```rust
fn first_word(s: &str) -> &str {
```

代码清单 4-9：优化 first_word 函数参数为字符串切片



如果我们有一个字符串切片，我们能直接作为参数传递，如果有一个 String，则可以直接穿一个 String 或者它的引用。这样的操作利用了一个叫作 `deref` 的特性，我们将在[第 15 章](https://doc.rust-lang.org/book/ch15-02-deref.html#implicit-deref-coercions-with-functions-and-methods)讨论这个特性。

定义一个函数来接收字符串切片而不是一个对 `String` 的引用，可以使我们的 API 更加通用和有用，而不会失去任何功能：

```rust
fn main() {
    let my_string = String::from("hello world");

    // first_word在参数是 String 切片时，能正常工作
    let word = first_word(&my_string[0..6]);
    let word = first_word(&my_string[..]);

    // first_word在参数是 String 的引用时，也能正常工作
    let word = first_word(&my_string);

    let my_string_literal = "hello world";

    // first_word在参数是字符串字面值切片时，能正常工作
    let word = first_word(&my_string_literal[0..6]);
    let word = first_word(&my_string_literal[..]);
    // 因为字符串字面值本身就是一种切片，所以直接作为参数传递也行
    let word = first_word(my_string_literal);    
}
```

## 其他类型的切片

字符串切片，如你所想，是特定于字符串的操作。但是还有一些其他类型的切片，考虑有下面这样一个数组：

```rust
let a = [1, 2, 3, 4, 5];
```

就像我们想要引用字符串的一部分一样，我们可能想要引用一个数组的一部分。我们可以像下面这样做：

```rust
let a = [1, 2, 3, 4, 5];
let slice = &a[1..3];
assert_eq!(slice, &[2, 3]);
```

这个切片的类型是 `&[i32]`。它和字符串切片的工作机制一样，内部存储了切片的开始索引和切片的长度。你可以在各种集合上使用这种切片，我们在第八章讲 Vector 时，会详细讨论这些集合。

## 总结

所有权、借用和切片这些概念，都是在编译时就能保证程序的内存安全。Rust 可以像其他编程语言那样控制内存的使用，只不过当数据超出范围时，有数据的所有者自动清理这些数据，这意味着你不必额外写代码来控制内存的申请和释放。
