---
title: "ServerSocketChannel"
date: 2022-12-04T23:59:47+08:00
draft: false
categories: ["java基础","NIO"]
---

```
所有Socket通道类（DatagramChannel、SocketChannel、ServerSocketChannel）继承AbstractSelectableChannel
 可以通过Selector来执行Socket通道的就绪选择
 
 
 ServerSocketChannel 不具备读和写的功能


Socket 不能被重复使用
Socket通道类 可以被重复使用
```



# 一、ServerSocketChannel

```
ServerSocketChannel 基于通道的socket 监听器
没有bind（） ，需要建立对等的socket 并使用它来监听端口
accept() 获取访问这个端口的socket，可以选择阻塞或者非阻塞的方式。
```

```java
ByteBuffer byteBuffer = ByteBuffer.wrap("hello lizhi".getBytes(StandardCharsets.UTF_8));

        // 绑定的端口
        int port = 8888;
        // 创建一个ServerSocketChannel
        final ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();

        // 用于监听8888端口
        serverSocketChannel.socket().bind(new InetSocketAddress(port));

        // 设置非阻塞模式
        serverSocketChannel.configureBlocking(false);

        while (true) {
          // 如果是阻塞式的，会一直等待连接
            final SocketChannel socketChannel = serverSocketChannel.accept();
            if (null == socketChannel) {
                System.out.println("不回答");
                Thread.sleep(1000);
            } else {
                System.out.printf("%s已经收到消息\n", socketChannel.socket().getRemoteSocketAddress());
                // 指针回到起点
                byteBuffer.rewind();

                // 向通道内写数据
                socketChannel.write(byteBuffer);
                socketChannel.close();
            }
        }
```

