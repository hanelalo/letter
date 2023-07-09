---
title: Rust入门:HashMap
date: 2023-07-09 08:15:48
tags: Rust
categories: Rust
description: 初步了解 HashMap 常规用法。
---

在编程开发时，实在是避免不了要使用 key-value 这种键值对形式的数据结构。本文将简单介绍一下 Rust 中的 HashMap 的使用。

## 创建

直接通过 `HashMap::new()` 方法调用就能创建一个 HashMap，不过使用 HashMap 需要额外通过 `use` 关键字将 HashMap 引入：

```rust
use std::collections::HashMap;

fn main() {
    let mut scores = HashMap::new();
    println!("len={},cap={},vals:{:?}", scores.len(), scores.capacity(), scores);
    scores.insert(String::from("Blue"), 10);
    scores.insert(String::from("Red"), 10);
    scores.insert(String::from("Green"), 10);
    scores.insert(String::from("Black"), 10);
    println!("len={},cap={},vals:{:?}", scores.len(), scores.capacity(), scores);
}
```

```
len=4,cap=7,vals:{"Red": 10, "Blue": 10, "Black": 10, "Green": 10}
```

如果在创建 HashMap 之前就知道了这个 Map 所需要的容量大小，可以使用 `with_capacity()` 方法来初始化一个 HashMap 实例。

## 访问 HashMap 的数据

```rust
use std::collections::HashMap;

fn main() {
    let mut scores = HashMap::new();
    scores.insert(String::from("Blue"), 10);
    scores.insert(String::from("Red"), 10);
    scores.insert(String::from("Green"), 10);
    scores.insert(String::from("Black"), 10);
  
    let score = scores.get("White").copied().unwrap_or(0);
    println!("The black score is {score}");
}
```

直接通过 `get()` 方法来读取 HashMap 中的数据，方法的参数是一个 map 中 key 的类型的引用，返回的是一个 `Option<T>`。如果传入的 key 在 map 中存在，则返回一个 `Some<T>`，如果不存在自然是返回一个 `None`。

而后面的 `unwrap_or` 方法则是在 map 中不存在指定的 key 时，返回一个默认值，这里的默认值设置为 0。

这里还需要考虑一下当要写入到 HashMap 的 key 已经存在的情况。

对于 insert 方法，是直接新值覆盖旧值。

而如果插入的 key 已经存在就保留旧值的话，就使用 entry + or_insert 方法组合实现：

```rust
use std::collections::HashMap;

fn main() {
    let mut scores = HashMap::new();
    scores.insert(String::from("Green"), 10);
    scores.insert(String::from("Black"), 10);

    scores.entry(String::from("White")).or_insert(9);

    println!("{:#?}", scores);
}
```

此时的 scorce 的数据像下面这样：

```json
{
    "Green": 10,
    "White": 9,
    "Black": 10,
}
```

## 修改 HashMap 中的数据

```rust
use std::collections::HashMap;

fn main() {
    let text = "hello world wonderful world";
    let mut map = HashMap::new();
    for item in text.split_whitespace() {
        let count = map.entry(item).or_insert(1);
        *count += 1;
    }
    println!("{:#?}", map);
}
```

这里是统计每个单词出现的次数，统计结果：

```json
{
    "world": 2,
    "wonderful": 1,
    "hello": 1,
}
```

可以通过 `map.entry(item).or_insert(0)`，当 item 对应的数据不存在时，就写入 0，存在则会返回 item 对应的值的引用，所以 `*count += 1` 也就相当于直接修改了 item 对应的值。

## 关于所有权

需要知道的是，对于实现了 Copy trait 的类型，不管是作为 map 中的 key 还是 value，insert 时都是复制的一份值，所以不存在所有权的转移。

而如果是 String 或者其他的未实现 Copy trait 的类型，当 insert 之后，所有权便归 map 所有。

```rust
    use std::collections::HashMap;

    let field_name = String::from("Favorite color");
    let field_value = String::from("Blue");

    let mut map = HashMap::new();
    map.insert(field_name, field_value);
    // field_name 和 field_value 在这里已经是无效的变量了
```

## 参考文档

* https://doc.rust-lang.org/book/ch08-03-hash-maps.html
