---
title: "Io"
author: "huochai"
date: 2023-01-30T16:25:10+08:00
draft: false

categories: [io ]
---

# 定义

在linux 中一切都是文件，文件就是由一串字节组成的对象。

在信息的交换过程中，我们都是对这些文件进行数据收发操作，简称为I/O操作。

# 分类

IO 操作可以分为内存IO、磁盘IO、网络IO

磁盘IO: 就是应用程序从磁盘中获取数据的过程

<img src="https://gcore.jsdelivr.net/gh/Footman56/imageBeds/202301301152416.png" alt="image-20230130115225009" style="zoom:50%;" />

用户空间是程序可以操作的，内核空间是操作系统来控制。

为了保证内核的安全，用户进程不能直接操作内核。

网络IO:就是应用程序从网络中其他机器中获取数据的过程

<img src="https://gcore.jsdelivr.net/gh/Footman56/imageBeds/202301301627280.png" alt="image-20230130115241216" style="zoom:50%;" />

根据IO 执行时CPU 的状态，可以分为阻塞IO、非阻塞IO.

阻塞IO就是在进程读写文件的时候CPU要等待完成之后再做其他的事情 ，当进程进入阻塞状态，是不占用CPU资源的。

非阻塞IO就是读写文件的时候CPU可以挂起当前线程，之后CPU去做其他的事情

## 同步阻塞

用户进程等待内核进程获取数据，在获取数据的期间，不做任何事。

<img src="https://gcore.jsdelivr.net/gh/Footman56/imageBeds/202301301627367.png" alt="image-20230130141321114" style="zoom:50%;" />

这种适合读写文件不频繁的情况，不需要快速反应请求。

## 同步不阻塞

非阻塞的系统调用调用之后，进程并没有被阻塞，内核马上返回给进程，如果数据还没准备好，此时会返回一个error。

与同步阻塞的区别是，通过轮询来确定是否准备好数据，在轮询的时候，CPU可以做其他的事情，当系统调用成功时候，还是要阻塞等待数据拷贝的

<img src="https://gcore.jsdelivr.net/gh/Footman56/imageBeds/202301301627025.png" alt="image-20230130152059343" style="zoom:50%;" />

## IO多路复用

可以实现一个线程监视多个文件句柄；一旦某个文件句柄就绪，就能够通知到对应应用程序进行相应的读写操作；没有文件句柄就绪时就会阻塞应用程序，从而释放出CPU资源。

通过一个或者多个线程来监听，能够减少监听线程的数据，否则就是一个线程检查一个读写操作。这才是复用

<img src="https://gcore.jsdelivr.net/gh/Footman56/imageBeds/202301301627182.png" alt="image-20230130152928530" style="zoom:50%;" />

在客户端操作服务器时，会创建三种文件描述符，简称FD。分别是writefds（写描述符）、readfds（读描述符）和 exceptfds（异常描述符）。

### select模型

select会阻塞监视这三种文件描述符，等有数据、可读、可写、出异常或超时都会返回；返回之后就会遍历FD 集合，去获取那些文件是可以进行操作的

每次调用select()方法，都需要把FD集合从用户态拷贝到内核态，并进行遍历。而**操作系统对单个进程打开的FD数量是有限制的，一般默认是1024个**。

优点是跨平台支持性好，几乎在所有的平台上支持。

缺点：随着FD数量增多而导致性能下降。

###  poll模型

poll 模型的原理与select模型基本一致，也是采用轮询加遍历，唯一的区别就是 poll 采用链表的方式来存储FD。就没有最大FD 数量限制

###  epoll模型

采用回调的形式，等有可以操作的文件之后，通过回调函数来通知

epoll模型是采用时间通知机制来触发相关的IO操作。它没有FD个数限制，而且从用户态拷贝到内核态只需要一次。它主要通过系统底层的函数来注册、激活FD，从而触发相关的 IO 操作，这样大大提高了性能。主要是通过调用以下三个系统函数：

+ epoll_create()函数，在系统启动时，会在Linux内核里面申请一个B+树结构的文件系统，然后，返回epoll对象，也是一个FD。
+ epoll_ctl()函数，每新建一个连接的时候，会同步更新epoll对象中的FD，并且绑定一个 callback回调函数。
+ epoll_wait()函数，轮询所有的callback集合，并触发对应的 IO 操作

## 信号驱动

- 开启套接字信号驱动IO功能
- 系统调用Sigaction执行信号处理函数（非阻塞，立刻返回）
- 数据就绪，生成Sigio信号，通过信号回调通知应用来读取数据

此种IO方式存在的一个很大的问题：Linux中信号队列是有限制的，如果超过这个数字问题就无法读取数据

<img src="https://gcore.jsdelivr.net/gh/Footman56/imageBeds/202301301627400.png" alt="image-20230130154904718" style="zoom:50%;" />

## 异步非阻塞

- 当用户线程调用了`aio_read`系统调用，立刻就可以开始去做其它的事，用户线程不阻塞
- 内核就开始了IO的第一个阶段：准备数据。当内核一直等到数据准备好了，它就会将数据从内核内核缓冲区，拷贝到用户缓冲区
- 内核会给用户线程发送一个信号，或者回调用户线程注册的回调接口，告诉用户线程Read操作完成了
- 用户线程读取用户缓冲区的数据，完成后续的业务操作
- 使用`aio_read`或者`aio_write`发起异步IO操作，使用`aio_error`检查正在运行的IO操作的状态。

<img src="https://gcore.jsdelivr.net/gh/Footman56/imageBeds/202301301627701.png" alt="image-20230130155302811" style="zoom:50%;" />



# java 实现

## BIO

就是同步阻塞模型，采用Socket API 

##  NIO

JDK1.4开始引入了NIO类库，主要是使用Selector多路复用器来实现。Selector在Linux等主流操作系统上是通过IO复用Epoll实现的。

## AIO

 AIO框架中，由于应用程序不是**轮询**方式，而是订阅-通知方式，所以不再需要Selector（选择器）了，改由Channel通道直接到操作系统注册监听 。









# 流读写

## 分类

可以从不同的角度对流进行分类：

1. 处理的数据单位不同，可分为：字符流，字节流

2. 数据流方向不同，可分为：输入流，输出流

3. 功能不同，可分为：节点流，处理流

**节点流**：节点流从一个特定的数据源读写数据。即节点流是直接操作文件，网络等的流，例如FileInputStream和FileOutputStream，他们直接从文件中读取或往文件中写入字节流。

![image-20230130162204993](https://gcore.jsdelivr.net/gh/Footman56/imageBeds/202301301628888.png)

**处理流**：“连接”在已存在的流（节点流或处理流）之上通过对数据的处理为程序提供更为强大的读写功能。过滤流是使用一个已经存在的输入流或输出流连接创建的，**过滤流就是对节点流进行一系列的包装**。例如BufferedInputStream和BufferedOutputStream，使用已经存在的节点流来构造，提供带缓冲的读写，提高了读写的效率，以及DataInputStream和DataOutputStream，使用已经存在的节点流来构造，提供了读写Java中的基本数据类型的功能。他们都属于过滤流。

![image-20230130162231415](https://gcore.jsdelivr.net/gh/Footman56/imageBeds/202301301628374.png)

## 结构：

Java所有的流类位于java.io包中，都分别继承字以下四种抽象流类型。

|        | 字节流       | 字符流 |
| ------ | ------------ | ------ |
| 输入流 | InputStream  | Reader |
| 输出流 | OutputStream | Writer |

1.继承自InputStream/OutputStream的流都是用于向程序中输入/输出数据，且数据的单位都是字节(byte=8bit)，如图，深色的为节点流，浅色的为处理流。

![img](https://gcore.jsdelivr.net/gh/Footman56/imageBeds/202301301628417.png)

2.继承自Reader/Writer的流都是用于向程序中输入/输出数据，且数据的单位都是字符(2byte=16bit)，如图，深色的为节点流，浅色的为处理流。

![img](https://gcore.jsdelivr.net/gh/Footman56/imageBeds/202301301628618.png)