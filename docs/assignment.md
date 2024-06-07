# 《互联网络程序设计》课程作业

## 基于iouring的文件读写

在muduo框架下，设计并实现基于iouring的读写channel。

## iouring

io_uring是Linux内核中的一种系统调用接口，用于处理存储设备的异步I/O操作。它解决了类似于read()、write()或aio_read()、aio_write()等函数提供的接口在处理通过文件描述符访问的数据时的性能问题。

### 接口

io_uring通过创建两个循环缓冲区，称为“队列环”，来存储I/O请求的提交和完成。对于存储设备，这些缓冲区分别称为提交队列（SQ）和完成队列（CQ）**。将这些缓冲区在内核和应用程序之间共享有助于提高I/O性能，避免了在两者之间复制这些缓冲区所需的昂贵系统调用134。

### 历史

Linux内核自2.5版本起就支持异步I/O，但使用起来被认为很困难且效率低下。旧的API仅支持某些特定的用例，尤其是只有在使用O_DIRECT标志并访问已分配的文件时才启用异步操作。这会阻止使用页面缓存，同时还会暴露应用程序于复杂的O_DIRECT语义。此外，Linux AIO不支持套接字，因此无法用于多路复用网络和磁盘I/O 。

## muduo

Muduo是一个基于非阻塞 I/O 和事件驱动的C++ 高并发 TCP 网络库，其核心是一个名为 EventLoop 的事件循环，用于响应计时器和 I/O 事件.

### 设计模式

muduo 基于 Reactor 模式，通过创建多线程来处理并发连接。每个线程拥有自己的 EventLoop，负责处理 I/O 事件，而主线程则负责接受新连接并将其分配给其他线程1.

### 多线程模型

muduo 使用的线程模型是 one loop per thread，即每个线程只有一个 EventLoop。这种模型有效地利用了多核 CPU，使得网络应用程序能够高效地处理大量并发连接2.

## channel

muduo 中的 channel 是一个关键组件，用于事件分发和管理。Channel 是 muduo 中的事件分发器，它负责处理文件描述符（fd）的 I/O 事件。
每个 channel 对象只属于一个 EventLoop，因此每个 channel 对象都只负责某一个文件描述符的 I/O 事件分发，但它并不拥有这个 fd。
muduo 用户通常不直接使用 Channel，而是使用更高层次的封装，例如 TcpConnection。

### channel 的生命周期和回调函数

channel 的生命周期由其 owner class（通常是 TcpConnection）负责管理。
channel 的成员函数都只能在 I/O 线程中调用，因此更新数据成员时不必加锁。
channel 中保存着 I/O 事件的类型以及对应的回调函数，包括读、写、关闭和错误回调。

### Tie 方法

Tie 方法用于防止 channel 在执行期间被提前释放，从而避免异常问题。
它将 channel 绑定到由 shared_ptr 管理的 owner 对象，确保在 handleEvent 中 owner 对象不会被销毁。

### channel 的工作流程

当有事件产生时，EventLoop 首先调用 channel 的回调函数。
channel 通过两个循环缓冲区（SQ 和 CQ）存储 I/O 请求的提交和完成。
channel 的更新由 update 方法负责。

## liburing

liburing 是一个基于 iouring 接口的用户态库，旨在简化应用程序对 iouring 实例的设置和拆除，并提供了一个更简单的接口，适用于不需要（或不想）处理完整内核实现的应用程序123.

### 接口和目的

liburing 提供了帮助设置和拆除 iouring 实例的功能。
它还为那些不需要或不想处理完整内核实现的应用程序提供了一个简化的接口。

## 要求

1. 指定在fedora最新平台上开发；

2. 指定使用C++语言开发；

3. 指定采用cmake管理项目；

4. 指定使用google test作为单元测试框架，测试用例自行编写，在cmake中编译；

5. iouring可以借助于liburing操纵，使用yum安装各种依赖；

6. iouring提供的异步io服务很多，本项目仅限制于普通文件的读写。对于目录的操作可以考虑直接使用C++17提供的filesystem库。

7. 扩展EventLoop类，包含一个iouring和eventfd。设计并实现File类，包含Channel对象。提供文件注册与读写功能，并在gtest框架上测试。

8. 作业完成后，撰写标准实验报告。将报告+源码打包发送到邮箱 niexiaowen@uestc.edu.cn。打包文件名为：unp24-学号-姓名.zip。

9. 最后截止提交时间：2024.06.16，第16周周日，晚12:00
