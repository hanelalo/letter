---
title: Rust入门:控制流
date: 2023-06-11 15:18:51
tags: Rust
categories: Rust
description: 简单记录一下 rust 中的控制流语法，以及一些可能反常识的语法。
---

本文会讲到 `if`、 `loop`、`while`、`for`语法，还会讲到在 rust 中的一些特殊语法，以及在使用上和其他编程语言的一些不同的地方。

## if

if 语句的语法其实没什么特殊的：

```rust
if x==0 {
	// do something
} else if x == 1{
	// do something
} else {
	// do something
}
```

但是需要注意的是，在某些编程语言中对于下面这种写法也是支持的，但是在 rust 中并不支持：

```rust
let x = 2;
if x {
	// do something
}
```

比如在 js 语言中，这里 x 如果等于 0，那就是 false，否则就是 true，但在 rust 中，编译就会报错。

然后，在对某个变量赋值时，如果需要根据条件进行赋值的话，也能通过 `if...else...` 的形式进行。

```rust
let x = if condition {5} else {6};
```

需要注意的是，这里的 x，不论最后是 5 还是 6，最终的变量变量类型都是确定的，只是值不确定而已，如果 if 和 else 返回的值的类型不一致，是否支持呢？

```rust
fn main() {
		let a = 3;
    let val = if a == 3 {5} else {"six"};
    println!("the value of val is:{val}");
}
```

```bash
$ cargo run
error[E0308]: `if` and `else` have incompatible types
  --> src/main.rs:32:35
   |
32 |     let val = if a == 3 {5} else {"six"};
   |                          -        ^^^^^ expected integer, found `&str`
   |                          |
   |                          expected because of this

For more information about this error, try `rustc --explain E0308`.
```

因为 if 和 else 的返回值类型不一致，所以编译会直接报错。

## loop

`loop` 看名称就能知道这是一种循环结构的语法。

```rust
fn main() {
	loop {
		println!("Hello");
	}
}
```

上面就是一个简单的 loop 循环体结构，一旦运行，它就是会不停地输出 "Hello"，知道按下 Ctrl + C 才会停止，换句话说，这就是一个死循环。

rust 提供了 break 语句用来退出循环。

```rust
fn main() {
    let mut counter = 0;
    let result = loop {
        counter += 1;
        if counter == 10 {
            break counter * 2;
        }
    };
    println!("The value of result is:{}", result);
}
```

这里是定义了一个可变的 counter 变量，然后每次循环会将 counter 加 1，如果 counter 等于 10，循环结束，并将 counter 的值乘以 2 作为返回值，返回值会赋值给 result 变量。

首先是，就以往的编程经验而言，这里的 `break counter * 2;` 改成 `return counter * 2;` 应该是可行的，但是实际上并不行，中介 loop 循环必须使用 break，如果有返回值的话，也必须写在 break 后面，且有分号在最后，以构成一个完整的 Statement。

> 因为这里不支持 `return conunter * 2;` 这种写法，所以其实也是不支持直接写 `counter * 2` 这种写法的。

然后就是，多重循环的退出问题。

```rust
loop {
  loop {
    if condition {
      break;
    }
  }
}
```

上面的 break 只能退出内层循环，而外层循环只能等循环结束，或者为外层循环写一个 break。

在 rust 中，可以为 loop 命名，break 语句可以指定退出的是哪个循环。

```rust
fn main() {
    let mut count = 0;
    'counting_up: loop {
        println!("count = {count}");
        let mut remaining = 10;
        loop {
            println!("remaining = {remaining}");
            if remaining == 9 {
                break;
            }
            if count == 2 {
                break 'counting_up;
            }
            remaining -= 1;
        }
        count += 1;
    }
    println!("End count = {count}");
}
```

```
$ cargo run
count = 0
remaining = 10
remaining = 9
count = 1
remaining = 10
remaining = 9
count = 2
remaining = 10
End count = 2
```

采用 `'counting_up: loop {}` 的形式对 loop 进行命名， `'counting_up:` 就是为 loop 命名的语法，前面的 `'` 和后面的冒号都是必须，中间则是名称。

而在 break 指定退出的循环时，使用 `break 'counting_up;` 语法即可。 

## while

```rust
fn main() {
    let mut count = 0;
    while count != 10 {
        count+=1;
        println!("count is {count}");
        if count == 7 {
            break
        }
    }
}
```

```
count is 1
count is 2
count is 3
count is 4
count is 5
count is 6
count is 7
```

while 循环相对来说就比较好理解了，其结构如下：

```rust
while condition {
	// loop
}
```

condition 为 true 就执行一次循环逻辑，然后再判断 condition 是否为 true，直到 condition 为 false 时，while 循环才会结束。

当然，在循环体内，依然可以使用 break 来终止循环。

## for

rust 的 for 循环结构体是 `for val in vals {}` 形式的。

```rust
fn main() {
    let arrays = [1, 2, 3, 4, 5];
    for arr in arrays {
        println!("{arr}");
    }
    for i in 1..=6 {
        println!("{i}");
    }
}
```

## 参考文档

* [The Rust Programming Language](https://doc.rust-lang.org/book/ch03-05-control-flow.html)

