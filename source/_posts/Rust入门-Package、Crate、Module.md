---
title: Rust入门:Package、Crate、Module
date: 2023-06-23 19:01:43
tags: Rust
categories: Rust
description: 本文将介绍如何通过模块系统组织 Rust 项目代码。
---

## crate

在 Rust 里面，crate 是最小的可编译单元。crate 分为两种，一种是二进制 crate，一种是库 crate。

二进制 crate 是可以编译得到一个可执行文件的，换句话说，它一定有一个 main 函数。库 crate 则不能编译成一个可执行文件，但是它定义的函数可以暴露给第三方使用。

## package

package 是由一个或多个 crate 组成，提供一个完整的功能。一个 package 会包含 Cargo.toml 文件，一个 Package 里面可以包含多个库 crate，但是最多只能包含一个二进制 crate，并且，每个 pakcage 必须包含一个 crate。

> 这里的 package 可以认为就是一个项目了。Rust 中取名字确实和其他语言很不一样。

当通过 `cargo new ` 创建一个 rust 项目之后，项目会有一个 Cargo.toml 文件，说明这是是一个 package，在 `src` 下面还有一个 `main.rs`，但是在 Cargo.toml 文件中并没有提到这个文件，这是因为，二进制了 crate 的根，就是 main.rs，而这个 crate 和 package 同名，而库 crate 的根则是 `lib.rs`。

## module

Rust 官方文档对模块系统进行了总结：

> 1. 总是从 crate 跟开始。当编译一个 crate 时，首先会去寻找 crate 的根，如果是二进制 crate，根文件为 src/main.rs，如果是库 crate，根文件为 src/lib.rs。
> 2. 在 crate 的根文件里面，你可以定义一个 module，比如你要定义一个 garden module，可以通过 `mod garden;` 来定义，编译器会尝试在以下位置去寻找 garden 模块的代码：
>    1. 在 `mod garden` 后面的大括号中寻找代码，此时语句结尾不是分号，而是大括号。
>    2. 在 `src/garden.rs` 文件中寻找。
>    3. 在 `src/garden/mod.rs` 中。
> 3. 在除了 crate 根的其他文件中，还可以定义子 module。比如在 garden 中中定义一个子 module： `mod vegetables;`。编译器会尝试在父 module 中的以下位置去寻找 vegetables module 的代码：
>    1. 在 `mod vegetables` 后的大括号中寻找。
>    2. 在 `src/garden/vegetables.rs` 文件中寻找。
>    3. 在 `src/garden/vegetables/mod.rs` 文件中寻找。
> 4. module 的代码路径。在 crate 中的一个 module，只要规则允许，你可以在 crate 的任何地方引用它。比如一个 `Asparagus` 类型在  garden 的 vegetables 模块中，那么它的路径是：`crate::garden::vegetables::Asparagus`。
> 5. private 和 public。默认情况下，module 中的代码是私有的，如果要让它公开，则需要通过 `pub mod` 来定义 module。但是 module 内部的定义依然还是私有的，同样也需要在定义语句前面加上 `pub` 关键字。
> 6. `use` 关键字。你可以通过 `use crate::garden::vegetables::Asparagus` 来引入 Asparagus，在使用时，也只需要写 `Asparagus` 而不用写前面的一长串。

接下来，举个例子来理解一下，一个餐厅一般都有前台、后台，前台是顾客所在，其余的都在后台。接下来我们我们创建一个库 crate 来建立餐厅的模型。

### 一个 module 的例子

首先，通过 cargo 创建一个库类型的项目：

```bash
$ cargo new restaurant --lib
```

这里比之前多了一个 `--lib` 参数，因为 `cargo new` 默认创建的是二进制类型 crate，加了 `--lib` 参数就是告诉 cargo 要创建的是一个库类型的 crate。

然后就能发现在 restaurant 中没有 main.rs，而是有一个 lib.rs。

然后，在 lib.rs 中，我们不再写 main 函数，取而代之，我们定义一个 front_of_house 的模块。

文件名：src/lib.rs

```rust
mod front_of_house {
  mod hosting {
    fn add_to_waitlist() {}
    
    fn seat_at_table() {}
  }
  
  mod serving {
    fn take_order() {}
    
    fn server_order() {}
    
    fn take_payment() {}
  }
}
```

回忆一下前面讲到编译器会如何寻找模块相应的实现代码，我们知道：这里定义的一个 front_of_house 模块，内部又有 hosting、serving 两个子模块。

不过这里寻找每个模块的实现都是直接从 `mod <module_name>` 后面的大括号中找到的。

这就像一个文件目录：

```
crate
	front_of_house
		hosting
			add_to_waitlist
			seat_at_table
		serving
			take_order
			server_order
			take_payment
```

接下来的内容，会以这个示例展开。

### module 引用路径

在 Rust 中，要调用函数时，首先要知道要调用的函数的路径，在 rust 中，路径有两种形式：

1. 绝对路径，是从 crate 的根开始。如果是外部的 crate，则是从 crate 的名称开始，如果是内部的 crate，则是从 `crate` 关键字开始。
2. 相对路径，是在当前 module 中使用，从 self、super 关键字或者在当前 module 中的名称开始。

而不管是那种路径，各层级之间都是通过 `::` 分隔。

比如现在，在 lib.rs 中调用 add_to_waitlist 函数：

文件：src/lib.rs

```rust
mod front_of_house {
  mod hosting {
    fn add_to_waitlist() {}
    
    fn seat_at_table() {}
  }
  
  mod serving {
    fn take_order() {}
    
    fn server_order() {}
    
    fn take_payment() {}
  }
}

fn eat_at_restaurant() {
  // 绝对路径
  crate::front_of_house::hosting::add_to_waitlist();
  // 相对路径
  front_of_hosue::hosting::add_to_waitlist();
}
```

> 选择相对路径还是绝对路径，取决于你的项目是否会将定义 crate 的代码和使用这个 crate 的代码一起移动，如果会一起移动，那么意味着相对路径不会变，那可以考虑使用相对路径，如果不会一起移动，那么也就意味着相对路径会变化，那就建议使用绝对路径，这样能保证在移动 crate 时需要改动的代码更少。

然后，上面的代码，编译一下会报错：

```bash
$ cargo check                                                              
    Checking restaurant v0.1.0 (/Users/hanelalo/develop/rust-labs/restaurant)
error[E0603]: module `hosting` is private
  --> src/lib.rs:19:28
   |
19 |     crate::front_of_house::hosting::add_to_waitlist();
   |                            ^^^^^^^ private module
   |
note: the module `hosting` is defined here
  --> src/lib.rs:2:5
   |
2  |     mod hosting {
   |     ^^^^^^^^^^^

error[E0603]: module `hosting` is private
  --> src/lib.rs:21:21
   |
21 |     front_of_house::hosting::add_to_waitlist();
   |                     ^^^^^^^ private module
   |
note: the module `hosting` is defined here
  --> src/lib.rs:2:5
   |
2  |     mod hosting {
   |     ^^^^^^^^^^^

For more information about this error, try `rustc --explain E0603`.
```

主要的报错信息就是 `hosting` 模块是私有的。

现在要把 `hosting` 暴露出来，需要在定义时，加上 pub 关键字：

```rust
pub mod hosting {}
```

但是在此检查还是会报错：

```rust
$ cargo check                                                                   
    Checking restaurant v0.1.0 (/Users/hanelalo/develop/rust-labs/restaurant)
error[E0603]: function `add_to_waitlist` is private
  --> src/lib.rs:19:37
   |
19 |     crate::front_of_house::hosting::add_to_waitlist();
   |                                     ^^^^^^^^^^^^^^^ private function
   |
note: the function `add_to_waitlist` is defined here
  --> src/lib.rs:3:9
   |
3  |         fn add_to_waitlist() {}
   |         ^^^^^^^^^^^^^^^^^^^^

error[E0603]: function `add_to_waitlist` is private
  --> src/lib.rs:21:30
   |
21 |     front_of_house::hosting::add_to_waitlist();
   |                              ^^^^^^^^^^^^^^^ private function
   |
note: the function `add_to_waitlist` is defined here
  --> src/lib.rs:3:9
   |
3  |         fn add_to_waitlist() {}
   |         ^^^^^^^^^^^^^^^^^^^^

For more information about this error, try `rustc --explain E0603`.
```

现在是 `add_to_waitlist` 函数无法访问，因为这个函数依然是私有的，所以需要写加上一个 `pub` 关键字。

```rust
pub fn add_to_waitlist() {}
```

在此尝试编译就会发现已经没有编译错误了，只是可能有一些告警而已。

### 最佳实践

前面提到过一个 package 可以同时包含二进制 crate、库 crate。通常来讲，包含库和可执行二进制模块的包往往在可执行二进制模块中只包含足够的代码来启动一个调用库模块代码的可执行程序。这样也使得其他项目也能使用这个 package 中的一些功能。

模块树应该在 `src/lib.rs` 中定义。然后，通过以包的名称作为路径的起始部分，可以在可执行二进制模块中使用任何公开的代码。可执行二进制 crate 成为库 crate 的用户，就像外部 crate 使用库 crate 一样：它只能使用公开的 API。这有助于设计一个良好的 API；你不仅是作者，也是一个客户端！

### super 关键字

简单点解释就是调用父 module 的代码时，需要使用 super 关键字。

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {
            super::in_service();
        }

        fn seat_at_table() {}
    }

    mod serving {
        fn take_order() {}

        fn server_order() {}

        fn take_payment() {}
    }

    fn in_service() {
    }
}
```

这里需要在 hosting 的 add_to_waitlist 函数中调用父 module 中的 in_service 函数，可以直接通过 `super::in_service()` 的方式调用。

### 结构体和枚举类型 public

主要需要知道的有 3 点：

1. 结构体和枚举在没专门指定时，默认就是 private 的，如果要 public，就得使用 `pub` 关键字。

2. 结构体定义时，就算使用了 `pub struct`，结构体内部的字段依然是默认 private 的，如果某个字段要 public 出来，在相应的字段前面也要加上 `pub`。

   ```rust
   pub struct User {
     pub username: String,
     age: u32
   }
   ```

3. 在枚举类型中，只要定义枚举类型时加上了 `pub`，枚举类型的的每一种值都是 public 的。

## use 关键字

在前面介绍了相对路径和绝对路径，但是每次通过 `crate::front_of_house::hosting::add_to_waitlist` 或者 `front_of_house::hosting::add_to_waitlist` 这样的方式进行调用，那未免太过麻烦，所以 Rust 提供了 `use` 关键字，用于将 crate 或者 crate 中的函数等引入到当前的上下文中。

```rust
mod front_of_house {
  mod hosting {
    fn add_to_waitlist() {}
    
    fn seat_at_table() {}
  }
  
  mod serving {
    fn take_order() {}
    
    fn server_order() {}
    
    fn take_payment() {}
  }
}

use crate::front_of_house::hosting::add_to_waitlist;

fn eat_at_restaurant() {
  add_to_waitlist();
}
```

这里通过 use 关键字将 `add_to_waitlist` 引入到了当前的上下文，所以使用时，直接以 `add_to_waitlist()` 的方式调用即可。当然这里使用 use 关键字时，也可以只指定到 `use crate::front_of_house:hosting;`，然后通过 `hosting::add_to_waitlist()` 的方式进行调用。

Rust 官方建议按照 `hosting::add_ti_waitlist` 的方式进行调用，因为相对来说，后者其实更加的优雅，一方面，这样能够很直观知道这里调用的是哪个 crate 中的函数；另一方面，在使用各种 crate 时，如果某两个 crate 中有同名的结构体，使用这种方式能更好的避免不必要的错误。

然后，需要注意的是，use 的作用范围只在当前的 module 中，子 module 中依然还是需要通过 use 关键字导入。

## as 关键字

在讲到 `use` 关键字时，有提到会存在两个 crate 中出现同名的结构体的情况。这种的通过只导入其所在 module 的方式可以解决：

```rust
use std::fmt;
use std::io;

fn function1() -> io::Result {
    
}

fn function2() -> fmt::Result<()> {
    
}
```

除此之外，Rust 还提供了 as 关键字，可以为某个 use 语句起一个别名，后续使用时就是使用这个别名即可。

```rust
use std::fmt::Result;
use std::io::Result as IoResult;

fn function1() -> Result {
    // --snip--
}

fn function2() -> IoResult<()> {
    // --snip--
}
```

## 重新导出

在 Rust 中，`pub use` 是一种用于重新导出（re-export）项的机制。它允许将一个项从一个模块重新导出到另一个模块，并使其对外可见。

通过使用 `pub use`，可以在当前模块中创建一个公共的接口，使外部模块可以直接访问和使用被重新导出的项，而无需知道其实际定义的位置。

`pub use` 的基本语法如下：

```rust
pub use path::item;
```

其中，`path` 是要重新导出的项的路径，`item` 是要重新导出的具体项。

以下是一个示例，展示了如何使用 `pub use`：

```rust
// 在 my_module.rs 模块中重新导出 sub_module 的 greet 函数
pub use sub_module::greet;

mod sub_module {
    pub fn greet() {
        println!("Hello from sub_module!");
    }
}
```

在上述代码中，`sub_module` 模块包含了一个函数 `greet`。通过 `pub use`，我们将 `sub_module` 的 `greet` 函数重新导出到了当前模块中。这意味着在外部使用 `my_module::greet()` 时，实际上是调用了 `sub_module::greet()`。

这种机制在设计公共接口或库的时候非常有用。它可以隐藏内部实现细节，同时提供清晰的公共接口给用户使用。

需要注意的是，`pub use` 并不是简单的重命名，它提供了对重新导出项的访问和使用。同时，通过 `pub use` 重新导出的项仍然需要满足可见性规则，即原始的 module、函数或者结构体必须是公开的才能被重新导出。

## 简化 use 语句

如果要使用一个 crate 中的很多 api，可能会有多个 use 语句，比如这样：

```rust
use crate::front_of_house::hosting::add_to_waitlist;
use crate::front_of_house::serving;
```

这里只有 2 个导入其实看起来还好，但如果特别多的时候，看起来就不是很好了，所以 Rust 允许一个 use 语句解决：

```
use craet::front_of_house::{hosting::add_to_waitlist, serving};
```

 如果是既导入了 hosting，又导入了它的 add_to_waitlist 函数，可以直接使用 self 代替：

```rust
// use crate::front_of_house::hosting;
// use crate::front_of_house::hosting::add_to_waitlist;

use crate::front_of_house::hosting::{self, add_to_waitlist};
```

或者，你想直接将 hosting 中所有东西都导入，可以直接以 `*` 代替：

```
use crate::front_of_house::hosting::*;
```

当然，这一些的一切，不管如何导入，都是必须遵循可见性原则。

## 分离 module 到多个文件

还是以前面的 restaurant 为例：

文件：src/lib.rs

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {
        }

        fn seat_at_table() {}
    }

    mod serving {
        fn take_order() {}

        fn server_order() {}

        fn take_payment() {}
    }
}

fn eat_at_restaurant() {
    // 绝对路径
    crate::front_of_house::hosting::add_to_waitlist();
    // 相对路径
    front_of_house::hosting::add_to_waitlist();
}
```

在真正的项目中，肯定不能将所有 module、所有代码都放到一个文件中，这样并不好维护。

接下来，展示代码的同时，还会列出当前 src 文件夹下的完整目录结构。

现在的目录结构是：

```
src/
	lib.rs
```

接下来我们尝试将 front_of_house 这个 module 拆分成单独的文件。

前面讲过编译器会在和 module 同名的文件中或者在和 module 同名的子文件夹中寻找实现的代码。

所以这里我们首先尝试只单独拆一个 `front_of_house.rs` 文件出来。

文件：src/front_of_house.rs

```rust
pub mod hosting {
    pub fn add_to_waitlist() {
    }

    fn seat_at_table() {}
}

mod serving {
    fn take_order() {}

    fn server_order() {}

    fn take_payment() {}
}
```

文件：src/lib.rs

```rust
mod front_of_house;

fn eat_at_restaurant() {
    // 绝对路径
    crate::front_of_house::hosting::add_to_waitlist();
    // 相对路径
    front_of_house::hosting::add_to_waitlist();
}
```

 此时，依然能编译通过，文件结构如下：

```
src/
	lib.rs
	front_of_house.rs
```

此时，在 front_of_house 中还有 hosting、serving 两个子 module，我们尝试将两个子 module 也拆分出来。

首先，我们尝试将 hosting、serving 两个 module 的代码文件放到和 `front_of_house.rs` 同级文件夹。 

文件：src/front_of_house.rs

```rust
pub mod hosting;

mod serving;
```

文件：src/hosting.rs

```rust
pub fn add_to_waitlist() {
}

fn seat_at_table() {}
```

文件：src/serving.rs

```rust
fn take_order() {}

fn server_order() {}

fn take_payment() {}
```

此时执行编译，会报错，反正就是 lib.rs 中调用的 add_to_waitlist 函数找不到在哪里。

现在的目录结构是：

```
src/
	lib.rs
	front_of_house.rs
	hosting.rs
	serving.rs
```

可以知道，**现在这个目录结构是错误的。**

接下里，我们新建一个 `front_of_house` 文件夹，并将 `hosting.rs`、`serving.rs` 两个文件移进去，注意此时没改变任何一个文件的内容，仅仅移动了位置，再编译一下，然后编译通过了，说明这样是没问题的，现在的目录结构是：

```
src/
	lib.rs
	front_of_house.rs
	front_of_house/
		hosting.rs
		serving.rs
```

这样基本就可以了，但是我们知道编译器还有一种寻找 module 实现代码的方式，那就是通过 `xxx/mod.rs` 文件。

我们再试试，直接在 `front_of_house` 文件夹中新建 `mod.rs`：

文件：`src/front_of_house/mod.rs`

```rust
pub mod hosting;

mod serving;
```

此时编译，编译失败，依然是找不到 `add_to_waitlist` 函数，此时的目录结构：

```
src/
	lib.rs
	front_of_house.rs
	front_of_house/
		mod.rs
		hosting.rs
		serving.rs
```

这说明，**当使用两种方式定义同一个 module 时，是会报错的。**

然后我们将 `front_of_house.rs` 文件删掉再试试，还是能编译通过，此时的目录结构：

```
src/
	lib.rs
	front_of_house/
		mod.rs
		hosting.rs
		serving.rs
```

这说明，Rust 是兼容了 `src/front_of_house.rs` 和 `src/front_of_house/mod.rs` 两种方式的，建议还是使用 `src/front_of_house.rs` + `src/front_of_hosue` 文件夹的这种方式，不然项目里面会有很多的 mod.rs 文件。

## 参考文档

* [Managing Growing Projects with Packages, Crates, and Modules](https://doc.rust-lang.org/book/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html)
