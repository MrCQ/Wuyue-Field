# I/O模型与Reactor编程模式

## 同步&异步，阻塞&非阻塞

同步与异步，阻塞与非阻塞，概念极易混淆，难以理清

* 同步与异步： 针对的是消息的通信机制
* 阻塞与非阻塞： 针对的是发出消息的线程所处状态

在此基础上，可将I/O模型分为三类模式：

* 阻塞IO
* 同步非阻塞IO
* 异步非阻塞IO

一般情况下，一个IO操作分为两个阶段

1. 等待数据准备好
2. 数据准备好后，由内核向进程复制数据

在此基础上，可对同步&异步，阻塞与非阻塞状态加以区分

是否同步： 看两个阶段是否存在阻塞状态
是否阻塞： 看阶段（1）是否有阻塞状态

## Unix网络IO模型

主要为以下五类IO模型：

* 阻塞式 I/O
* 非阻塞式 I/O
* I/O 复用（select，poll，epoll...）
* 信号驱动式 I/O（SIGIO）
* 异步 I/O（POSIX 的 aio_系列函数）

### 阻塞式IO

阻塞式IO是标准的线性处理模式，简单易于理解，请求端发起请求之后一致等待知道数据返回。同步阻塞模式

![](https://lc-mhke0kuv.cn-n1.lcfile.com/97ddb24bef7e8f445e16.png)


### 非阻塞式IO

非阻塞IO，在发起请求之后，询问数据是否准备好，如果数据准备好则读取数据，否则轮训（先干点别的等会再来问一下），直到数据准备好，所以在数据准备阶段调用方并不是出于阻塞状态。但是，在数据读取阶段，调用方还是处于阻塞状态，因此该模型完整看应该是**同步非阻塞**

![](https://lc-mhke0kuv.cn-n1.lcfile.com/1d806ad741f74c2833f3.png)

### IO复用

IO多路复用方式包括有：select, poll, epoll， I/O 多路复用可以同时监听多个 fd，如此就减少了为每个需要监听的 fd 开启线程的开销。

select, poll与epoll之间的区别在于：
![](https://oscimg.oschina.net/oscnet/56233af10105512cba59a1be13ffffa9c5d.jpg)

select是内核级别的，可以同时等待监听多个socket，socket发起方处于等待状态，当某个socket数据状态准备好时，返回数据可读信息，进入数据读取阶段。因此可认为该模式为**同步阻塞模式**。

![](https://lc-mhke0kuv.cn-n1.lcfile.com/0dded096fb8290ccf980.png)


### 信号驱动IO

发起系统调用之后，直接返回，把结果处理过程交给信号处理函数完成，调用进程继续完成自己的后续工作，在数据准备完成之后，进程会受到信号并由信号处理函数进行相应处理，开启数据接收流程。

![](https://lc-mhke0kuv.cn-n1.lcfile.com/ff79e80c581163292594.png)

### 异步IO

用户程序使用操作系统提供的异步 I/O 的系统调用aio_read，这个调用会即时返回，当整个 I/O 操作完成后它会通知用户进程。

用户进程调用aio_read完成系统调用后，不论数据是否准备妥当，都会直接返回，数据准备完成之后，内核会直接将数据复制到进程。

![](https://lc-mhke0kuv.cn-n1.lcfile.com/554280a61c6141e2bbc8.png)

等待数据准备过程以及数据复制过程都是非阻塞状态，因此该模式下可认为是**异步非阻塞模式**

### 总结

一张图对比集中IO模型的区别：

![](https://raw.githubusercontent.com/waylau/essential-java/master/images/net/1-12%20Comparison%20of%20the%20five%20IO%20models.png)


## Reactor模式

编程模式对应IO模型思想，Java中的NIO对应到同步非阻塞IO模式，Reactor编程模式可以作为IO多路复用的实现载体

Reactor：

> The reactor design pattern is an event handling pattern for handling service requests delivered concurrently to a service handler by one or more inputs. The service handler then demultiplexes the incoming requests and dispatches them synchronously to the associated request handlers.

其特点在于：

* 事件驱动
* 多输入源同时处理
* 每个事件对应到handler处理器

![](https://oscimg.oschina.net/oscnet/1f5c2e595e41559ac0c829a96a918c15f4e.jpg)

Reactor处理模式包含三种角色类型：

* Reactor：将I/O事件分派到对应的Handler
* Acceptor：处理新连接
* Handler：还行非阻塞读写任务

如何来理解这三类角色的相互作用关系，以及Reactor的工作原理呢？ 

类比应用场景：长途客车在路途上，有人上车有人下车，但是乘客总是希望能够在客车上得到休息。

传统做法：每隔一段时间（或每一个站），司机或售票员对每一个乘客询问是否下车。

Reactor 做法：汽车是乘客访问的主体（Reactor），乘客上车后，到售票员（acceptor）处登记，之后乘客便可以休息睡觉去了，当到达乘客所要到达的目的地后，售票员将其唤醒即可。


### 单Reactor单线程

这种模式下，Reactor线程负责多路Socket的分离，新连接触发connect事件之后会由Reactor交给Acceptor进行处理，有IO读写事件之后会交给handler处理。

* Acceptor的任务是构建handler，将相关socket信息绑定到对应的handler
* 有读写事件时，动过Reactor的分发，handler就能处理了

![](https://oscimg.oschina.net/oscnet/e9f813b5b08ac68021039ae5141c03f3cfc.jpg)

但是这种模式下，读写事件都是通过Reactor主线程来完成的，效率相对比较低

### 单Reactor多线程

该模式相对于上一种模式，主体上是一致的，区别在于将IO读写操作都放到了线程池进行，不再局限于主线程空间，能够降低主线程Reactor的性能开销，使其可以专心于事件分发工作，以此通过多线程来提升整个应用的吞吐。

![](https://oscimg.oschina.net/oscnet/1828d992e8821f9f093b6bf12c58732bb13.jpg)

### 多Reactor

该模式相对于第二种模式，其将Reactor分为两部分：

* mainReactor 负责监听 server socket，用于处理新连接的建立，将建立的 socketChannel 指定注册给 subReactor
* subReactor 维护自己的selector，完成 mainReactor 指派的 socketChannel 的IO事件监听工作，并将对数据的处理过程交给线程池进行

![](https://oscimg.oschina.net/oscnet/7ea7f4beb7b3c1d1c87d7b9e3bab8b6afb4.jpg)

这种模式下，每个模块的工作更加专一，耦合度更低，性能和稳定性也能得到大幅提升，可支撑的客户端数量也能有很大提升。

参考：

[从 I/O 模型到 Netty](https://juejin.im/post/58bbaee6ac502e006b02f607)

[NIO系列—Reactor模型](https://juejin.im/post/5b4570cce51d451984695a9b)