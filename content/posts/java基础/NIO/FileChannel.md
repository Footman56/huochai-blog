---
title: "FileChannel"
date: 2022-12-05T00:44:36+08:00
draft: false

categories: ["java基础","NIO"]
---

# 一、FileChannel

```java
/**
     * FileChannel读取数据到buffer
     */
    public static void main(String[] args) throws Exception {
        // 通过文件创建channel
        RandomAccessFile randomAccessFile = new RandomAccessFile("/Users/peilizhi/file/one.txt", "rw");
        final FileChannel channel = randomAccessFile.getChannel();
        // 分配一个1024大小到缓冲区
        ByteBuffer allocate = ByteBuffer.allocate(1024);
        int len;
        // 从通道读取到缓冲区 -1表示结束
        while ((len = channel.read(allocate)) > 0) {
            // 输出缓冲区
            System.out.println("len = " + len);
            allocate.flip();
            while (allocate.hasRemaining()) {
                System.out.println("allocate = " + (char) allocate.get());
            }
            allocate.clear();
        }
      //文件关闭的时候，管道同样关闭
        randomAccessFile.close();
    }
```



## 1.1 获取FileChannel

```
可以通过 FileInputStream 、FileoutputStream获取FileChannel
或者通过 RandomAccessFile 与一个文件关联，之后再获取数据
```

## 1.2 读取数据到Buffer

```
ByteBuffer allocate = ByteBuffer.allocate(1024);
channel.read(allocate)   // 返回读到多少字节，-1表示结束
```

## 1.3 写数据到Buffer

```java
   通过文件创建channel
        RandomAccessFile randomAccessFile = new RandomAccessFile("/Users/peilizhi/file/one.txt", "rw");
        final FileChannel channel = randomAccessFile.getChannel();

        // 写入的内容
        String context = "peilizhi";
        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
        byteBuffer.clear();
        // 先放入缓冲区
        byteBuffer.put(context.getBytes(StandardCharsets.UTF_8));
        // 转换模式,之前是读;现在是写
        byteBuffer.flip();


        // 如果还有byte
        while (byteBuffer.hasRemaining()) {
            channel.write(byteBuffer);
        }
        randomAccessFile.close();
```

```
position() 获取通道中当前的位置
position(long newPosition) 设置通道中的位置
 	将位置设置为大于文件当前大小的值是合法的，但不会更改文件的大小。 稍后尝试在这样的位置读取字节将立即返回文件结束指示。 稍后尝试在这	样的位置写入字节将导致文件增长以容纳新字
truncate（）：此频道的文件截断为给定的大小。
		如果给定的大小小于文件的当前大小，则文件将被截断，丢弃超出文件新末尾的任何字节。 如果给定的大小大于或等于文件的当前大小，则不会修		 改文件
force（）：强制将管道数据写到磁盘上

transferTo（）、transferFrom（）管道之间数据传递
```