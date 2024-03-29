---
title: IO：同步、异步、阻塞、非阻塞、多路复用
date: 2022-06-12 23:04:00
tags: IO
description: 看了很多关于同步、异步 IO 的文章，但总是似懂非懂，闲来无事，查阅各方资料，总算是明白了一些。
---

<img src='https://image.hanelalo.cn/image/202206121200343.png'/>

<!--more-->

## 从线程状态说起

在操作系统层面，线程状态分为 5 种。

1. 初始状态
2. 可运行状态
3. 休眠状态
4. 运行状态
5. 终止状态

状态流转图如下：

![操作系统线程状态流转](http://image.hanelalo.cn/image/202206121336730.png)

而对于 Java 来讲，线程在整个生命周期中有如下 6 种状态：

* New

  新创建的线程，还未被执行。

* Runnable

  线程内的逻辑正在被执行，或等待调度，所以可以细分为两种状态：

  * Ready

    线程可执行，只不过当前 CPU 时间片并未被调度（CPU 在多线程环境下，并非一个线程执行到底，而是分时间片执行，当前时间片在执行线程 A，可能下一个时间片就执行线程 B），线程被挂起。

  * Running

    当前线程正在被调度执行。

  总之，在 Runnable 状态下，线程要么被调度，使用 CPU，要么等待 CPU 调度，CPU 正在被其他线程使用。简而言之，CPU 没有空闲着浪费资源。

* Blocked

  正在等待获取监视器锁，以进入或重新进入同步代码块。此时不会分配 CPU 时间片。

* Waiting

  无时间限制地等待其他线程执行唤醒操作。此时不会分配 CPU 时间片。
  
* TIMED_WAITING

  有时间限制地等待其他线程执行唤醒操作。此时不会分配 CPU 时间片。

* TERMINATED

  线程执行完毕。

状态流转图如下：

![Java线程状态流转](http://image.hanelalo.cn/image/202206121349663.png)

Java 中，将操作系统的运行中、可运行状态合并为 Runnable 状态，而操作系统的休眠状态，在 Java 中又根据调用不同的 api 分为 WAITING、TIMED_WAITING、BLOCKED 3 种状态。

除此之外，像下面的代码，线程 t 的状态是 Runnable。

```java
  public static void main(String[] args) {

    Thread t =
        new Thread(
            () -> {
              try {
                System.in.read();
              } catch (Exception e) {
                e.printStackTrace();
              }
            });
    t.start();
    try {
      Thread.sleep(1000);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
    System.out.println(t.getState());
  }
```

```
RUNNABLE
```

按理说，这里调用 t 调用了 System.in.read() 这种会阻塞线程的 api，线程应是阻塞的，但是实际的 state 却是 RUNNABLE。

从操作系统的层面看，`System.in.read()` 确实会阻塞线程，因为需要等待外部设备输入并将数据拷贝到内核，再拷贝到用户态，而对于 Java 来讲，等待内核的数据到达和等待 CPU 调度没区别，一样都是等待。

所以，调用阻塞式 api （比如 IO）时，因为“视角”不同，从操作系统层面看，线程是处于休眠状态，不分配 CPU 时间片，而对于 Java 来讲，虽然此时本质上也没分配 CPU 时间片，但同样都是等待，所以归为 RUNNABLE 状态，形成了一种在等待调度 CPU 时间片的假象。

那么，同步阻塞 IO、同步非阻塞 IO 中的阻塞是指什么呢？

## 阻塞到底是什么？

前文讲到，调用 IO 时，从操作系统视角看，线程实质上是处于休眠状态，Java 的视角看，线程处于等待 CPU 资源调度的状态，这是本质，从本质反推到现象则是，调用了 IO 的 api 之后，线程“卡”住了，换言之，在当前线程里，CPU 不继续执行指令。一直到 IO 操作结束，数据已经从内核空间复制到用户空间，线程才恢复运行，又有了被分配 CPU 时间片的可能，等到被调度的时候，线程内的逻辑又可以继续往下执行。

> 关于内核空间和用户空间，简而言之就是为了操作系统的安全，应用程序并不是什么函数都能调用，有些函数的调用是需要权限的，调用普通函数时，处于用户态，而当需要调用这种特权函数时，就需要进入到内核态。



以网络 IO 为例，用户线程调用 recv() 函数，陷入内核态，但是内核中没数据。针对内核没数据的情况，有 2 种解决方案：

1. 调用 io 函数后，线程置为休眠状态。当数据从物理链路到达网卡之后，由 DMA 将数据从网卡拷贝到内核缓冲区，然后再由 CPU 从内核缓冲区将数据复制到用户空间，这里有 2 次复制，在第二次复制完成之前，用户线程都是处于休眠状态的。
2. 调用 io 函数后，不管内核有没有数据，都立即返回。用户线程或通过轮训的方式一直查询内核数据是否到达，或者当内核数据到达后发出信号，然后 CPU 复制数据到用户空间。

> 没深入了解操作系统的 api，所以具体做 IO 的 api 是什么，我也不知道，等哪天把书看完了应该就知道了。

而阻塞，指的就是这 2 种处理方案。第一种，一直等着数据到达内核再返回，这个过程中不会占用 CPU 时间片，线程处于休眠状态，就是阻塞；第二种，不管内核有没有数据都直接返回，线程不会一直处于休眠状态，就是非阻塞。

换言之，**阻塞**是指用户线程是否一直等着复制数据，如果是，那线程就一直处于休眠状态，即为阻塞，如果不是，那线程就不是一直处于休眠状态，就是非阻塞。

## 同步和异步 IO

IO 模型主要分为：

* 同步阻塞 IO。
* 同步非阻塞 IO。
* IO 多路复用。
* 事件驱动 IO。
* 异步 IO。

![IO模型](http://image.hanelalo.cn/image/202206121720984.png)

将整个 IO 过程分为两阶段，一阶段在用户态，二阶段在内核态。

* 同步阻塞，线程从用户态到内核态，两阶段都在阻塞。
* 同步非阻塞，用户态不会阻塞，一阶段一直在轮训检查数据是否到达，当数据到达内核后进入二阶段，复制数据到用户态，发生在内核空间，这个过程是阻塞的。
* IO 多路复用，可以简单理解为一个线程负责监听多个 IO，当有一个 IO 完成后，通知用户线程处理 IO 结束后的数据，从内核拷贝数据到用户态的过程，用户线程依然是阻塞（详见后文 Reactor 线程模型）。
* 信号驱动 IO，告知内核，当某个信号到达时，通知用户线程。
* 异步 IO，发起 IO 请求，等 IO 数据到达并复制到用户空间后，这个过程和用户线程没关系，复制完后，要么通知，要么回调的方式通知用户线程处理数据。

> 信号驱动 IO 和异步 IO 的区别在于，前者是通知用户进程可以复制数据，后者是通知用户进程数据复制完成。

同步和异步的区别在于，二阶段从内核空间复制数据到用户空间时，用户线程是否阻塞，阻塞就是同步，不阻塞就是异步。

## 同步、阻塞小结

介绍完了阻塞和同\异步概念，结合前文 5 中 IO 模型再总结一下：

**同步**，关注第二阶段，是指用户线程是否一定要等到数据到达内核，然后复制到用户空间，这个过程，用户线程是否阻塞。

**阻塞**，关注第一阶段，是指用户线程是否一直等着复制数据，如果是，那线程就一直处于休眠状态，即为阻塞，如果不是，那线程就不是一直处于休眠状态，就是非阻塞。

理解这两个概念之后，关于同步阻塞 IO、同步非阻塞 IO、异步 IO 是否就清晰了呢？

> 所以，我感觉其实相对同步阻塞 IO，同步非阻塞 IO 本身有点不明显，因为对于用户线程来说，等内核数据到达，跟一直循环询问内核数据是否到达，没什么区别，只不过同步阻塞少用了 CPU，同步非阻塞 提高了 CPU 利用率，用来轮询。
>
> 而 IO 多路复用，则是基于同步非阻塞 IO 的一次大改进，如果有一天我发现我理解错了再改。
>
> 我发现理解了阻塞和同步的概念之后，我依然不是很理解 IO 多路复用的概念。

## IO 多路复用

提到 IO 多路复用，网上一大片文章就是什么 select、poll、epoll 来了，IO 多路复用到底是什么？我也没见几篇文章回归到这个问题上。

在前文给到 5 种 IO 模型对比图中，看见 IO 多路复用在用户态依然会阻塞，在内核态也会阻塞，似乎和同步阻塞 IO 没啥区别，不过 IO 多路复用又比同步非阻塞 IO 多了一个“就绪”通知，具体的区别又是什么呢？

在同步阻塞 IO 中，每一个 IO 都会阻塞一个线程，这样十分耗费线程资源，极端情况下甚至可能出现应用开了一大堆线程，大部分线程都因为 IO 处于休眠状态，这本身是一种资源浪费。

所以，就想，不然就让专门的一个线程来负责 IO 事件的监听，当某个 IO 请求从内核返回后，立马找一个线程来处理数据，这样的话，阻塞的线程就只有 1 个，至于其他线程，则可以处理其他事。

> 以服务器网络 IO 的场景举例，当客户端发起一个网络连接时，服务端处理这个连接肯定需要开一个线程，如果同时 10000 个连接请求到来，那岂不是需要开 10000 个线程，Java 的线程一般需要 512K 到 1M 内存空间，10000 个线程，那就是接近 10G。所以这种一个连接一个线程的处理方式，不可控性太强了，服务器容易炸。
>
> 那么，如果有一种技术，可以同时监听多个网络 IO 请求呢？那么就只需要一个线程来实现这种技术，进行监听，然后再创建一个线程池，用来处理业务逻辑、建立连接的逻辑，线程池本身是有可控性的，来 10000 个请求，我也不会开 10000 个线程，可以扔线程池里面慢慢处理。

所以 IO 多路复用，其实本身是 select、poll、epoll 这些技术的底层原理，它的关注点很单纯的就是我不能让每次 IO 都“卡”死一个线程。

> select、poll、epoll 的原理和区别见 6 号参考资料。

那么 IO 多路复用的实际使用场景是怎样的？

### Reactor 线程模型

Reactor 线程模型便是基于 IO 多路复用来实现的。

Reactor 的核心组件有 3 个：

* Reactor，负责监听请求事件，并分发事件，如果是连接事件，则分发给 Acceptor，
* Acceptor，获取网络连接。
* Handler，业务处理器。

可以分为 3 种 Reactor 模型：

* 单 Reactor 单线程模型

  只有一个线程，可以理解为，只是在代码层级上将组件做了区分，本质上接受事件、分发事件、处理事件其实就在一个线程中，本身的资源开销自然不大。

  1. 客户端请求建立连接；
  2. Reactor 监听到连接事件，交由 Acceptor 处理；
  3. Acceptor 对象调用 accept() 方法，建立连接，并创建 Handler，由于响应后续的请求事件；
  4. 上一步建立的连接中发来请求，Reactor 监听到请求事件，发现不是连接事件，就交由该连接对应的 Handler 处理；
  5. Handler 通过 read() 读取数据，执行业务逻辑之后，通过 send() 方法发送响应。

  问题在于因为只有一个线程，所以当 Handler 还没处理完请求，新来的请求只能先等着，不适用于请求量比较大的场景。

  单 Reactor 单线程模型适用业务处理很快的场景，比如 Redis 就是使用的这种模型。

  ![单Reactor单线程](http://image.hanelalo.cn/image/202206122045576.png)

* 单 Reactor 多线程模型

  1. 客户端请求建立连接；
  2. Reactor 监听到连接事件，分发给 Acceptor 处理；
  3. Acceptor 调用 accept() 方法，建立连接，并创建一个 Handler 用户响应连接上的后续请求。
  4. 客户端发来请求，Reactor 发现不是连接事件，交由连接对应的 Handler 处理；
  5. Handler 调用 read() 方法读取数据，然后交由线程池里处理；
  6. 线程池处理完成后，将响应数据返回给 Handler；
  7. Handler 接收到响应数据，调用 send() 方法发送响应数据。

  在这种模型下，Handler 就不负责业务逻辑了，而只负责数据接受和发送，业务逻辑交由线程池处理，提高了资源利用率。也提高了并发能力。 

  但是，因为只有一个 Reactor 来监听事件，所以对于瞬间的高并发场景，会存在性能瓶颈。

  ![单Reactor多线程](http://image.hanelalo.cn/image/202206122051222.png)

* 多 Reactor 多线程模型

  多 Reactor 多线程模型下，定义了 MainReactor 和 SubReactor 的概念，MainReactor 负责监听连接事件，然后交由 Acceptor 处理，Acceptor 调用 accept 方法获取连接，然后交由 SubReactor，SubReactor 将连接注册到 select 进行监听，在这个连接上后续的请求就不需要 MainReactor 处理，而是有 SubReactor 处理。

  1. 客户端请求建立连接。
  2. MainReactor 监听到连接事件，交由 Acceptor 处理；
  3. Acceptor 调用 accept() 方法建立连接，并交由 SubReactor；
  4. SubReactor 拿到连接后，注册到 select 中继续监听，并创建一个 Handler 用于响应该连接后续的请求；
  5. 客户端发来请求，SubReactor 监听到请求事件，分发到该连接的 Handler 中进行处理；
  6. Handler 调用 read() 方法读取数据，然后扔到线程池中进行业务逻辑的处理；
  7. 线程池处理完之后，将响应数据数据返回给 Handler；
  8. Handler 接收到响应数据后，调用 send() 方法发送数据；

  这样的好处在于 MainReactor 和 SubReactor 分工合作，能更好的的提高并发能力和资源，且相对于单 Reactor 多线程模型，因为采用多 Reactor，MainReactor 和 SubReactor 各司其职，对于瞬间的高并发也有了足够的应对能力。

  > 值得一提的是，Java 中大名鼎鼎的网络框架 Netty，就是使用的多 Reactor 多线程模型，这从初始化时需要区分 bossGroup 和 workGroup 就可以看出来。

  ![多Reactor多线程](http://image.hanelalo.cn/image/202206122239809.png)

## 参考资料

1. [Life Cycle of a Thread in Java](https://www.baeldung.com/java-thread-lifecycle)
2. [极客时间《Java 并发编程实战》专栏：《09 | Java线程（上）：Java线程的生命周期》](https://time.geekbang.org/column/article/86366)
3. [DMA和零拷贝](https://mp.weixin.qq.com/s/2_-ot3t9Yaws7bOqEhv78Q)
4. [三分钟短文快速了解信号驱动式IO，似乎没那么完美](https://www.itzhai.com/articles/it-seems-not-so-perfect-signal-driven-io.html)
5. [信号驱动I/O](https://static.kancloud.cn/luoyoub/network-programming/2234088)
6. [LINUX – IO MULTIPLEXING – SELECT VS POLL VS EPOLL](https://devarea.com/linux-io-multiplexing-select-vs-poll-vs-epoll/)
7. [彻底搞懂Reactor模型和Proactor模型](https://cloud.tencent.com/developer/article/1488120)
8. [如何深刻理解Reactor和Proactor？](https://www.zhihu.com/question/26943938/answer/1856426252)



