---
title: Rust入门:什么是Ownership?
date: 2023-06-14 22:43:57
tags: Rust
categories: 文档翻译
description: Ownership 是 rust 语言独有的特性，对 rust 语言的其他部分有着很深的影响。
---

> 本文是 rust 官方文档中所有权章节翻译的第一篇，原文地址：https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html

Ownership 是 rust 语言独有的特性，对 rust 语言的其他部分有着很深的影响。它保证 rust 在不需要垃圾回收（即开发人员常说的 GC，Garbage Collection），所以理解 Ownership 的工作机制是十分重要的。在本章中，我们将讨论 Ownership 及其相关的几个功能：borrowing、slices，以及 rust 的内存布局。

> 译者注：
>
> * borrowing 暂且翻译为借用，slices 暂且翻译为切片，后文不再专门说明。如果以后顿悟了，有更好的翻译再来改。
> * 在不特别解释的情况下，本文中所说的所有权，就是指 Ownership。
> * 后续的内容中会按原文加上个人理解进行翻译，如果有插入非原文翻译内容，会专门用"译者注：" 来标识。
> * 有些名词还会在小括号中给出英文原文，比如：堆（原文：heap）。

#  什么是 Ownership？

所有权是 rust 用于管理内存的一系列的规则。所有编程语言都必须要管理自己使用计算机内存的方式。有些语言有垃圾回收机制，在程序运行期间，会定期查找不再使用的内存；还有其他的一些语言，开发者必须明确分配、释放内存。Rust 使用了第三种方式：内存通过所有权系统进行管理，编译器有一套相应的检查规则。违背了任何规则，编译都会失败。所有权的任何特性都不会让程序的运行性能降低。

因为所有权对于大多数开发人员来说是一个新的概念，它确实需要一些时间来理解和适应。好消息是，你对 rust 的所有权机制的理解越深，你就越容易开发出安全高效的代码，请坚持下去。

当你了解了所有权，你就有了一个坚实的基础用于理解 rust 的独特之处。在这一章，你将通过常见的 string 数据结构相关的一些例子来学习所有权。

> **堆和栈：**
>
> 很多的编程语言并不要求你理解关于堆和栈相关的知识。但是在像 Rust 这种系统编程语言中，值定义在栈还是堆上，会影响语言的行为方式，以及相应的一些必须做出的决定。解关于堆和栈相关的所有权知识将在本章后面的内容讲解，这里先简单讲解一下，以备不时之需。
>
> 堆和栈都是程序运行时内存的一部分，但是它们的结构各不相同。栈按获取值的顺序有序存储，并按相反的顺序删除值，这被成为先进后出。想象有一堆盘子：当你添加盘子时，把新添加的盘子放到最上面，而你要取一个盘子时，也是从最上面取。从中间取或者放盘子则不是很好操作！添加数据到栈中的操作叫做压栈（原文：pushing onto），从栈中删除数据，叫做弹栈（原文：popping off）。所有存在栈上的数据必须要有固定且已知的大小。编译时大小未知或者大小会改变的数据必须存储在堆中。
>
> 堆的数据比较乱：当你将数据写入到堆中，会申请一部分内存空间。内存分配器会在堆中寻找一块空的、足够大的内存，并标记为已被使用，然后返回一个指针。这个处理过程，叫做在堆内存分配（原文：allocating on the heap），有时候也直接简单叫作分配（写数据到栈里面时不需要考虑内存分配问题）。因为返回的指针的大小是已知且固定的，所以可以将指针存储在栈上面，但是当你想要获取真实的数据时，就必须通过指针来获取真实数据。想象一下你们团队到餐厅聚餐，当你进餐厅时，会说明你们有多少人，服务员会为你找到一个足够大的桌子，如果团队中有人比你晚到，那么他只需要问你在几号桌就能找到你。
>
> 写数据到栈比写数据到堆中快，因为写数据到栈里面时，内存分配器不需要去搜索新的内存空间来存储新数据，新的数据都是直接存在栈顶的。相较而言，在堆上分配内存要做的工作更多，因为内存分配器首先要找到一块足够大的内存，然后还要记录下来，为下一次内存分配做准备。
>
> 在堆上访问数据比在栈上访问数据要慢，因为访问堆上的数据需要通过指针进行寻址。现代的处理器在内存中跳跃越少，处理得就越快。以此类推，想象一下服务员在餐厅的很多个餐桌上为顾客点菜。最高效的方式是在 A 桌点完所有菜之后，再到 B 桌点菜，这样保证在每一桌都是完成了所有的点菜工作之后，再到下一桌。如果一会儿在 A 桌点个菜，一会儿在 B 桌点个菜，一会儿又会 B 桌点个菜，这样反复的效率是很低的。同样的道理，一种情况是处理器要处理的数据离当前位置比较近（比如在栈上面），另一种情况是接下来要处理的数据离当前位置很远（比如在堆上），前者肯定比后者更快。
>
> 当程序调用了一个函数，函数的参数值（可能包含堆上数据的指针）和函数的本地变量都通过压栈的方式写入到栈中，当函数执行完成，这些数据会通过弹栈操作删除。
>
> 跟踪代码的每一部分使用了堆上的什么数据，让堆上的重复数据量最小化，及时清除无用的堆上数据以保证不会内存耗尽，这些都是所有权系统要解决的问题。一旦理解了所有权，你通常不需要考虑堆、栈的问题，但是了解所有权系统的目标是管理堆上的数据有助于解释并理解所有权系统的工作方式。

## 所有权规则

首先，我们看看所有权的规则。在学习本章的示例时，请记住这几个所有权的规则：

* Rust 中每一个值都有所有者（原文：Owner）。
* 每个值在每个时刻，只会数据一个所有者。
* 当所有者从作用域中离开时，值会被清除。

## 变量作用域

现在我们已经学习了 Rust 的基本语法，所有后面的例子中不会再包含定义 main 函数的代码，所以如果你要继续往下学习的话，请手动讲例子中的代码写到 main 函数中。总之，我们的例子会比较简洁，让我们更关注真实的细节，而不是样板代码。

所有权的第一个例子，我们将研究变量的作用域问题。作用域是程序中某一部分的有效范围。定义以下的变量：

```rust
let s = "hello";
```

变量 `s` 指向一个字符串，这个字符串的值是硬编码到了程序中的。这个变量的有效范围是从它声明的点开始，一直到当前的作用域结束。代码清单 4-1 通过注释解释了变量 s 的有效范围。

```rust
    {                      // s 在这里是无效的，因为这里 s 还未声明
        let s = "hello";   // s 从这里开始有效

        // 声明变量 s 之后的处理逻辑
    }                      // 作用域到这里就结束了，s 也不在有效

```

代码清单 4-1: 变量和它的有效作用域



换言之，这里有两个比较重要的点：

* 当 s 进入作用域，它就有效。
* 在离开作用域前，它一定是有效的。

到这里，作用域和和有效变量之间的关系和其他的编程语言是相似的。现在我们会基于当前所理解的知识，再来理解 String 类型。

## String 类型

要理解所有权的规则，我们需要一种比第 3 章的“[Data Types](https://doc.rust-lang.org/book/ch03-02-data-types.html#data-types)”还要复杂的数据类型。之前讲到的所有数据类型都是大小已知，并且可以存储在栈上，当其从作用域离开时，可以从栈上删除，并且当代码的其他部分需要相同值而作用域不同时，可以很快速、琐碎得复制一个新的、完全独立的变量。但是我们想要的是存储在堆上数据，想要探究 Rust 是如何清理这些数据的，而 `String` 类型是一个很好例子。

我们的讨论会聚焦的 String 类型的所有权相关的知识。这些知识也适用于其他的一些复杂数据类型，不管它们是标准库创建的，还是第三方库创建的，或者是你自己所创建的。更深度的关于 String 的讨论请参看[第 8 章](https://doc.rust-lang.org/book/ch08-02-strings.html)。

我们知道字符串的字面值都是硬编码到代码中（译者注：这里是指前面的代码清单 4-1）。当我们想要处理一些文本时，字符串字面值是很有用的，但是它并不是用于所有的场景，比如：我们想要存储用户输入的文本，要怎么办？这种场景下，Rust 有第二种解决方案，那就是 `String` 类型。这个类型的数据在堆上进行管理，所以它能存储支持在编译时大小不可知的文本。你可以使用 `from` 函数，通过一个字符串字面值来创建一个 String 类型的变量。

```rust
let s = String::from("hello");
```

这里的双引号 `::` 操作，允许我们将这个特定的 `from` 函数的命名空间放在 String 类型下，而不是使用类似 `string_from` 的名称。我们会在第 5 章的“[Method Syntax](https://doc.rust-lang.org/book/ch05-03-method-syntax.html#method-syntax)”一节讨论这个操作符号，刚提到的命名空间会在第 7 章的 “[Paths for Referring to an Item in the Module Tree](https://doc.rust-lang.org/book/ch07-03-paths-for-referring-to-an-item-in-the-module-tree.html)”一节进行讨论。

上面定义的这种字符串是可以随时发生改变的：

```rust
    let mut s = String::from("hello");

    s.push_str(", world!"); // push_str() 添加一个字面值到字符串中

    println!("{}", s); // 这里会输出: hello, world!
```

这里到底有什么不一样？为什么 `String` 可以改变，而字符串字面值不能改变？区别在于这两种类型在内存的处理上不一样。

## 内存 & 分配

在使用字符串字面值时，我们在编译时就知道它的内容，所以这些文本会硬编码到最终的可执行文件中。这也是字符串字面值既快又高效的原因。但是这些特点都因为它有这不可变的特性。不幸的是，对于在编译时大小未知，或者在运行时大小会发生变化的每一段文本，我们并不可能都分别放一块内存到二进制文件中。

对于 String 类型，为了支持可变、可增长的一段文本，我们需要在堆上分配一段内存（编译时未知）用来保存其内容，这意味着：

* 这段内存必须在运行时向内存分配器申请。
* 当我们使用完这块内存之后，我们需要一种方式能够将内存返还给内存分配器。

对于第一点，由我们自己完成：当我们调用 `String::from` 时，它的内部实现会申请所需要的内存。这在编程语言中是通用的行为。

然而，第二点却不是这样。在有 GC 的编程语言中，GC 会跟踪并清理没有被任何地方使用的内存，并且我们并不需要关心它。在大多数没有 GC 的编程语言中，我们需要知道内存何时不再使用，并且必须明确的释放这段内存，就像我们申请内存时那样。要正确做到这一点，在编程历史上一直是一个难题。如果我们忘了，将会浪费内存，如果我们提前释放了内存，那将会得到无效的标变量，而如果我们释放了两次内存，那就是一个 bug。我们需要保证恰好一个 allocate 调用对应一个 free 调用。

 Rust 却选择了不同的路：一旦程序运行超出了变量的作用域，那么内存会自动释放。这是代码清单 4-1 的 String 版本：

```rust
{
	let s = String::from("hello"); // s 从这里开始生效
	// 声明变量 s 后的处理逻辑
} // 作用域到此结束，s 不再有效
```

当 s 超出作用域时，Rust 会自动调用一个特定的函数来释放内存，这个函数就是 `drop`，在 drop 函数中，String 类型变量的所有者可以放置代码来释放内存。Rust 会在闭合的花括号除自动调用 drop 函数。

>**Note：**
>
>在C++中，将资源在项目生命周期的末尾释放的这种模式有时被称为"资源获取即初始化"（Resource Acquisition Is Initialization，RAII）。如果您使用过RAII模式，那么Rust中的drop函数会很熟悉。

这种模式对于编写 Rust 代码有着深远的影响。这现在看起来很简单，但是当我们想让多个变量使用堆上的内存时，情况变得复杂。现在让我们来看一些例子。

### 变量和数据交互：移动

> 译者注：这里的“移动”对应原文的单词是“move”。

在 Rust 中，多个变量可以通过不同的方式和同一数据进行交互。我们一起看看代码清单 4-2：

```rust
let x = 5;
let y = x;
```

代码清单 4-2：将 integer 值赋值给变量 `x` 和 `y`



我们能猜到这里做了什么：绑定 `5` 到变量 `x`；然后复制 `x` 的值并绑定到变量 `y` 上。我们现有有两个都等于 5 的变量，x 和 y。事实确实是这样，因为 integer 是一种简单的、大小已知且固定的类型，所以这里的 `5` 是放在栈里面的。

接下来我们看一个 String 类型的版本：

```rust
let s1 = String::from("hello");
let s2 = s1;
```

这看起来和前面的代码很像，我们可能会认为它和前面的代码清单 4-2 是一样的工作流程，比如第二行是复制变量 s1 的值并绑定到变量 s2 上。但事实并非如此。

让我们看看图 4-1，探究这里的 String 到底发生了什么。一个 String 类型变量由图左边的 3 部分组成：一个指向内存中保存字符串内容的内存的指针，一个 length 用于记录字符串长度，一个 capacity 用于记录字符串的容量。这一组数据存储在栈上。右边的图是堆上用来保存字符串内容的内存。 

![图 4-1：String 变量 s1 的内存结构](http://image.hanelalo.cn/image/202306122255974.png)

图 4-1：String 变量 s1 的内存结构



length 是当前 String 变量的字符串内容的内存使用量，单位是字节（byte）。capacity 是变量从内存分配器申请的内存总量，单位也是字节。length 和 capacity 的差异很重要，但是在这里我们不关注 capacity。

当我们为 s1 和 s2 赋值时，字符串的数据会复制一份，这里所说的复制，复制的是栈上的指针、length、capacity，并不会赋值指针所指向的堆中的数据，内存中的数据之间的关系就像图 4-2 这样：

![图 4-2：复制 s1 的指针、length、capacity 得到 s2](http://image.hanelalo.cn/image/202306122258484.png)

图 4-2：复制 s1 的指针、length、capacity 得到 s2



并不会像图 4-3 这样，如果 Rust 将堆上的数据也复制一份，那么当数据非常大是，这中复制操作的代价是非常大的。

![图 4-3：当执行 s1 = s1 时，可能发生的其他情况](http://image.hanelalo.cn/image/202306122303392.png)

图 4-3：当执行 s1 = s1 时，可能发生的其他情况



在前面，我们说当变量超出作用域时，Rust 会自动调用 drop 函数释放内存。但是在图 4-2 中，两个指针指向了同一份内存数据，这里的问题在于，当 s1 和 s2 都超出了作用域，它们都会尝试释放相同的内存。这是前面提到的两次释放同一块内存的安全漏洞。

为了保证内存安全，在 `let s2 = s1;` 这一行后面，Rust 会将变量 s1 置为无效，所以当 s1 超出作用域时，Rust 不用自动释放任何内存。你可以事实在 `let s2 = s1;` 之后再使用变量 s1，你会发现会报错：

```rust
let s1 = String::from("hello");
let s2 = s1;
println!("{}, world!", s1);
```

运行代码就会报错，因为使用了无效的变量。

```rust
$ cargo run
   Compiling ownership v0.1.0 (file:///projects/ownership)
error[E0382]: borrow of moved value: `s1`
 --> src/main.rs:5:28
  |
2 |     let s1 = String::from("hello");
  |         -- move occurs because `s1` has type `String`, which does not implement the `Copy` trait
3 |     let s2 = s1;
  |              -- value moved here
4 |
5 |     println!("{}, world!", s1);
  |                            ^^ value borrowed here after move
  |
  = note: this error originates in the macro `$crate::format_args_nl` which comes from the expansion of the macro `println` (in Nightly builds, run with -Z macro-backtrace for more info)
help: consider cloning the value if the performance cost is acceptable
  |
3 |     let s2 = s1.clone();
  |                ++++++++

For more information about this error, try `rustc --explain E0382`.
error: could not compile `ownership` due to previous error
```

如果你在使用其他编程语言时，通过深拷贝和浅拷贝这两个词，那可能会觉得 Rust 这种只复制指针、length、capacity 的形式是浅拷贝。但是因为 Rust 将第一个变量作废了，所以不叫作浅拷贝，这种操作在 rust 中叫做移动（原文：move）。在这个例子中，我们认为是变量 s1 移动到了变量 s2，所以，内存中真正发生的操作应是像图 4-4 这样的：

![图 4-4：当 s1 失效后的内存示意图](http://image.hanelalo.cn/image/202306122325694.png)

图 4-4：当 s1 失效后的内存示意图



这里终于解开了我们的疑惑，当 s2 超出作用域时，将只有它会释放这部分内存。

此外，这也是 Rust 在设计上作出的一个选择：Rust 从来不会主动深拷贝数据。因此，可以认为运行时任何的拷贝都是不会耗费很多性能的。

### 变量和数据的交互：克隆

> 译者注：这里的“克隆”对应原文单词是“clone”。

如果我们想要在 rust 中使用深拷贝，而不是仅仅只复制栈上的数据，可以使用 `clone` 方法。我们会在第 5 章讨论方法的语法，但因为方法在所有编程语言中是一个比较通用的概念，可能你在此之前就已经了解它了。

> 译者注：本文中“方法”对应的英文原文是“method”，“函数”对应的英文原文是 “function”。

下面是一个使用 `clone` 方法的例子：

```rust
let s1 = String::from("hello");
let s2 = s1.clone();

println!("s1 = {}, s2 = {}", s1, s2);
```

它能像图 4-3 描述的一样正常工作：会将堆上的数据也复制一份。

当看见调用 `clone` 方法时，应该意识到这里执行的代码是很耗费性能的。

### 栈上数据：复制

下面的代码清单 4-3 使用了 integer，奇妙的是，它能正常运行。

```rust
let x = 5;
let y = x;

println!("x = {}, y = {}", x, y);
```

这段代码按理说是有问题的，因为它并没有调用 clone 方法，但是 x、y 竟然依然都是有效的变量，换句话说，x 似乎并没有移动（move）到 y 上。

这是因为这里使用的 integer 类型的的长度在编译时就是已知且固定的，会保存在栈上，所以复制它的值就变得十分容易，这意味着当声明了变量 y 之后我们没必要让变量 x 失效。换句话说，这里没有深拷贝、浅拷贝之分，就算调用了 clone 方法，其实也和浅拷贝没区别。

Rust 有一个叫 Copy 的 trait（译者注：trait 可参见https://doc.rust-lang.org/book/ch10-02-traits.html），使用了这个 Copy 注解的类型，就会放在栈上面，就像 Integer 一样。如果一个类型使用了 Copy 的注解，它的变量将不会被 move，而是会被复制，当它的值被赋值给其他变量后，原来的变量依然还是有效的。

如果一个类型已经实现了 Drop trait，Rust 将不会允许该类型或者该类型的某一部分使用 Copy 注解，否则将会产生编译错误。要学习如何在你的自定义类型上使用 Copy 注解，可以看附录 C 的“Derivable Traits”一节。

你可以查看官方文档以确定哪些类型使用了 Copy trait，但是通常情况下，任何的标量值类型都可以使用 Copy trait。如果是需要分配资源，或者本身就是某种形式的资源的类型，肯定不能使用 Copy trait。

下面是一些使用了 Copy trait 的类型：

* 所有的 integer 类型，比如 u32。
* boolean 类型。
* 所有的浮点型，比如 f64。
* 字符类型。
* tuple 类型，如果包含的类型都是实现了 Copy，则可以使用 Copy trait。比如 `(i32, i32)` 可以，而 `(i32, String)` 不行。

## 所有权和函数

传一个参数值到函数中，和给一个变量赋值相似。函数传参会 move 或者复制，就像变量赋值一样。代码清单 4-3 中的例子通过注释解释了变量什么时候进入作用域，什么时候出作用域。

```rust
fn main() {
  let s  = String::from("hello"); // s 进入作用域
  takes_ownership(s); // s 的值 move 到这个函数中
  										// 所以 s 在这里无效了
  let x = 5;					// x 进入作用域
  makes_copy(x);			// x 会 move 进这个函数中
  										// 但是因为 x 是 i32 类型的，所以到这里依然可以使用 x
} // 到这里，x 超出作用域，然后是 s，但是 s 的值已经 move 了，所以这里不会针对 s 发生任何事情

fn takes_ownership(some_string: String) { // some_string 进入作用域
  println!("{}", some_string);
} // 到这里，some_string 超出作用域，自动调用 drop，切内存被释放。

fn makes_copy(some_integer: i32) { // some_integer 进入作用域
  println!("{}", some_integer);
} // 到这里 some_integer 超出作用域，无事发生
```

代码清单 4-3：函数和所有权



如果我们在调用了 `takes_ownership` 后再尝试使用变量 `s`，Rust 会抛出一个编译一场，这个静态检查会防止写出 bug。尝试在 main 函数中添加代码来使用 `s` 和 `x`，看看你可以在哪里使用它们，而 Rust 又会在哪里阻止你使用它们。

## 返回值和作用域

返回一个值，也会发生所有权的转换。代码清单 4-4 的代码和代码清单 4-3 一样使用注释解释了有返回值的函数：

```rust
fn main() {
    let s1 = gives_ownership();         // gives_ownership 将它的返回值 move 到 s1

    let s2 = String::from("hello");     // s2 进入作用域

    let s3 = takes_and_gives_back(s2);  // s2 move 到 takes_and_gives_back 中，这里也会将它的返回值 move 到 s3
                                        
} // s3、s1 超出作用域并释放内存，s2 已经 move 了，所以无事发生

fn gives_ownership() -> String {             // gives_ownership 会将返回值 move 到调用它的地方

    let some_string = String::from("yours"); // some_string 进入作用域

    some_string                              // some_string 被返回，并 move 到调用的地方
}

fn takes_and_gives_back(a_string: String) -> String { // a_string 进入作用域
                                                      

    a_string  // a_string 被返回并 move 到调用的地方
}
```

代码清单 4-4：返回值的所有权转让



所有权会已知遵循相同的模式：赋值给另一个变量，将会将它 move 走。当一个在堆上有数据的变量超出作用域时，如果它的值没有 move 到其他变量上，那么堆上的数据将会被清除。

虽然这种方式可以运行，但是在每个函数中获取所有权并返回所有权的过程有点繁琐。如果我们只想让函数使用一个值而不获取所有权，那会变得相当麻烦。除了我们可能希望返回函数体中生成的数据之外，我们还需要将传入的值再次传递回去，以便之后继续使用。

Rust 允许我们的函数返回一个 tuple，比如下面的代码清单 4-5：

```rust
fn main() {
    let s1 = String::from("hello");

    let (s2, len) = calculate_length(s1);

    println!("The length of '{}' is {}.", s2, len);
}

fn calculate_length(s: String) -> (String, usize) {
    let length = s.len(); // len() returns the length of a String

    (s, length)
}
```

代码清单 4-5：返回参数的所有权



但是对一个本就应该支持的常规操作来讲，这操作太复杂，幸运的是，Rust 有一个特性可以支持使用值，而不用转让所有权，这个特性就是引用。
