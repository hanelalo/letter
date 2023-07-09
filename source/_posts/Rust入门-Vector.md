---
title: Rust入门:Vector
date: 2023-07-02 09:57:44
tags: Rust
categories: Rust
description: Vector 入门。
---

Vector 只能存储相同数据类型的数据。和数组的区别在于，Vector 有自动扩容机制，对于运行时用来存储一个列表更加友好，因为运行时更多的是不确定一个数据集合的长度的。

本文将介绍 Vector 常见的用法，并且还会简单讲一下 Vector 的一些原理。

## 创建一个 Vector

通常情况下，创建 Vector 有两种方式：一种是直接调用 Vec 结构体的 `new` 方法创建一个 Vector 实例；一种是使用 `vec!` 宏。

```rust
let mut v = Vec::new(); // 第一种方式
v.push(1);
v.push(2);
v.push(3);

let v = vec![1,2,3]; // 第二种方式
```

会发现这里并没有指定 Vector 中的元素数据类型，在这种情况下，Rust 会自动探测它的类型。

我们查看 Vec 结构体的代码，能发现 Vector 还使用了一种叫作泛型的技术：

```rust
pub struct Vec<T, #[unstable(feature = "allocator_api", issue = "32838")] A: Allocator = Global> {
    buf: RawVec<T, A>,
    len: usize,
}
```

看见 Vec 结构体的定义种有类似 `<T>` 的字样，说明使用了泛型，这个以后会讲到，这里只是提一下。

很多时候，可能即使在运行时，我们依然可以提前动态计算出接下来要创建的 Vector 的元素个数，此时我们还可以通过 `Vec::with_capacity(capacity: usize)` 方法来创建一个指定了 capacity 的 Vector。

而使用 `vec!` 宏的话，同样也有 `vec![x;n]` 这样的语法来创建一个 capacity 为 n 的 Vector，并且会把所有元素都初始化为 x。

不管那种，这种指定 capacity 的方式创建 Vector 依然还是具体自动扩容（后面原理浅析小节中讨论）的机制，而指定 capacity 只是为了避免不必要的扩容而已，算是一个使用时的小优化。

## 向 Vector 中增加元素

如前面的代码所见，其实一般情况下直接调用 `push` 方法即可。

## 访问 Vector 中的数据

和其他编程语言一样，在 Vector 中也有 index 概念，同样也是从 0 开始，通过 index 访问 Vector 数据时，有下面两种方式：

```rust
    let mut v = vec![1, 2, 3, 4, 5];
    let first = &v[1];
    let first = v.get(1);
```

`&v[1]` 这种方式，返回的就是 v 中 index 为 1 的数据引用，如果传入的 index 超过了这个 v 的最大 index，就会有一个 panic 结束掉程序运行。

>  对照 Java 中也就是数组越界异常。

而 `v.get(1)` 这种形式则是返回一个 `Option<T>`，其中 T 是 Vector 中的数据类型，如果传入的 index 过大，这个 Option 返回值就是 None，否则就是 `Some<T>` 这样就不会直接 panic。

## 遍历

```rust
    let mut v = vec![1, 2, 3, 4, 5];
    for i in &v {
        println!("{i}");
    }
```

在 for 循环语句中，使用的是 `&v` 而不是 `v`，这里的 i 是 v 中元素的**不可变引用**，并未发生所有权转移，如果我们把 `&` 删掉，会发现循环完之后，v 这个变量就没法访问了。

如果想在循环的时候修改 v 中元素的值，自然要将 i 变成可变引用：

```rust
    for i in &mut v {
        *i += 5;
    }
```

 而这里因为 i 是引用，所以需要通过 `*` 操作符拿到真实的值之后再进行操作。

## 原理浅析

首先，需要知道的是，Vector 的数据结构，只包含了 3 个东西：

* len，当前 Vector 中存储数据个数。
* cap，当前 Vector 的容量 capacity。
* ptr，指向堆上内存的指针。

ptr 指向堆上内存，意味着 Vector 中的元素是存在堆上的，尽管像 i32 这种类型的长度是固定的，但是在 vector 中依然是在堆上分配一块内存进行存储。

`Vec::new()` 会创建一个 capacity 和 length 都是 0 的 Vector，这意味着在执行完这个调用之后，其实还没有在堆上分配内存。

> 这个和 Java 很不一样，Java 种创建一个 ArrayList 都会有默认 capacity，同时会创建一个长度为 capacity 的数组，所以在 new 的时候就会发生内存分配。

官方文档也提到，只有当 `capacity * mem::size_if::<T>() > 0` 时才会在堆上分配内存。

在前面提到了在 Vector 中有 len 和 cap 两个属性，这两个是不一定相等的，始终会保证 `cap >= len`，而扩容的条件就是 push 时发现 `cap == len`，扩容后，capacity 等于第一个比原来的 capacity 大的 2 的 n 次幂，比如原来 capacity 是 5，那么扩容后变成了 8。 

因为 Vector 本身是要分配一块连续的内存，所以扩容时，还需要移动现有的元素到新的内存地址，这就以为者，当 vector 有一个引用还存在于当前上下文时，如果调用了 push 这种可能会引发扩容操作的方法，会导致编译失败，不然的话，可能会导致之前的引用指向的内存地址的数据无效。

更进一步，向 Vector 中不断 push 元素会引发扩容，但是如果将一个很大的 Vector 中的元素一个一个删除，并不会自动缩小容量来释放内存。

## 参考文档

* https://doc.rust-lang.org/book/ch08-01-vectors.html
* https://doc.rust-lang.org/std/vec/struct.Vec.html
