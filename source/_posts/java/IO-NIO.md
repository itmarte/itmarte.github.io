---
title: Java IO/NIO 对比
date: 2020-05-20 16:19:27
tags:
    - java
    - io
    - nio
categories:
    - java
    - IO/NIO
---
# Java NIO Buffer, Channel 及 Selector

## Java IO VS NIO

- JDK 1.4 之前，java.io 包，

  面向流的I/O系统

  （字节流或者字符流）

  - 系统一次处理一个字节
  - 速度慢

- JDK 1.4 提供，java.nio 包，

  面向块的I/O系统

  - 系统一次处理一个块
  - 速度快

  ​

  NIO主要有三大核心部分：Channel(通道)，Buffer(缓冲区),Selector。

  ​       传统IO基于字节流和字符流进行操作，而NIO基于Channel和Buffer(缓冲区)进行操作，数据总是从通道读取到缓冲区中，或者从缓冲区写入到通道中。Selector(选择区)用于监听多个通道的事件（比如：连接打开，数据到达）。因此，单个线程可以监听多个数据通道。

  NIO和传统IO（一下简称IO）之间第一个最大的区别是，IO是面向流的，NIO是面向缓冲区的。

## Buffer 缓冲区

缓冲区实际上是一个容器对象，更直接的说，其实就是一个数组。
在 NIO 库中，所有数据都是用缓冲区处理的：

- 在读取数据时，它是直接读到缓冲区中的；
- 在写入数据时，它也是写入到缓冲区中的；

在 NIO 中，所有的缓冲区类型都继承于抽象类 Buffer。常见的缓冲区 Buffer 包括：

- ByteBuffer 存储了字节数组 `final byte[] hb;`

- CharBuffer 

  ```java
  final char[] hb;
  ```

  - **ByteBuffer 与 CharBuffer 之间的转换需要使用字符集 Charset**
  - Charset 具体使用，参见 [Java Charset 字符集](https://www.jianshu.com/p/1c61e001b609)

- ShortBuffer `final short[] hb;`

- IntBuffer `final int[] hb;`

- LongBuffer `final long[] hb;`

- FloatBuffer `final float[] hb;`

- DoubleBuffer `final double[] hb;`

Buffer 类的属性：

- `private int mark = -1;` 记录一个标记位置
- `private int position = 0;`

> A buffer's <i>position</i> is the index of the next element to be read or written.  A buffer's position is never negative and is never greater than its limit.
> 当前操作的位置

- `private int limit;`

> A buffer's <i>limit</i> is the index of the first element that should not be read or written.  A buffer's limit is never negative and is never greater than its capacity.
> 可以存放的元素的个数

- `private int capacity;`

> A buffer's <i>capacity</i> is the number of elements it contains.  The capacity of a buffer is never negative and never changes.
> 数组容量

- 大小关系：**mark <= position <= limit <= capacity**

Buffer 类的方法：

- `allocate(int capacity)` 分配一个缓冲区，默认 limit = capacity
- `put()` 在当前位置添加元素
- `get()` 得到当前位置的元素
- `clear()` 将 Buffer 从 读模式 切换到 写模式 （该方法实际不会清空原 Buffer 的内容）

```java
public final Buffer clear() {
    position = 0;
    limit = capacity;
    mark = -1;
    return this;
}
```

- `flip()`  将 Buffer 从 写模式 切换到 读模式

```java
public final Buffer flip() {
    limit = position;
    position = 0;
    mark = -1;
    return this;
}
```

** `clear()` VS `flip()`**：

- 在写模式下，Buffer 的 limit 表示你最多能往 Buffer 里写多少数据。
  - 因此写之前，调用 `clear()`，使得 `limit = capacity;`
- 在读模式时，Buffer 的 limit 表示你最多能从 Buffer 里读多少数据。
  - 因此读之前，调用 `flip()`，使得 `limit = position;`

IntBuffer 的使用：

```java
public static void main(String[] args) throws Exception {
    // 创建 int 缓冲区 capacity 为 4
    // 默认 limit = capacity
    IntBuffer buffer = IntBuffer.allocate(4);
    System.out.println("Capacity & Limit: " + buffer.capacity() + " " + buffer.limit());

    // 往 Buffer 中写数据
    buffer.put(11);
    buffer.put(22);
    buffer.put(33);
    buffer.put(44);

    System.out.println("Position: " + buffer.position());

    // 在从 Buffer 中读数据之前，调用 flip()
    buffer.flip();

    while (buffer.hasRemaining()) {
        System.out.print(buffer.get() + "  ");
    }
}
```

输出：

> Capacity & Limit: 4 4
> Position: 4
> 11  22  33  44

## Channel 通道

- Java NIO 的核心概念，表示的是对支持 I/O 操作的实体的一个连接
- 通过它可以读取和写入数据（并不是直接操作，而是通过 Buffer 来处理）
- 双向的

常用的 Channel 包括：

- FileChannel 从文件中读写数据
- DatagramChannel 从 UDP 中读写数据
- SocketChannel 从 TCP 中读写数据
- ServerSocketChannel 监听新进来的 TCP 连接，每一个新进来的连接都会创建一个 SocketChannel。

### FileChannel 连接到文件的通道

**FileChannel 无法设置为非阻塞模式，只能运行在阻塞模式下**
常用方法：

- `int read(ByteBuffer dst)` 从 Channel 中读取数据，写入 Buffer
- `int write(ByteBuffer src)` 从 Buffer 中读取数据，写入 Channel
- `long size()` 得到 Channel 中文件的大小
- `long position()` 得到 Channel 中文件的当前操作位置
- `FileChannel position(long newPosition)` 设置 Channel 中文件的当前操作位置

使用 FileChannel 来复制文件的例子：

```java
public static void main(String[] args) throws Exception {
    // 通过 InputStream 或者 OutputStream 来构造 FileChannel
    FileChannel in = new FileInputStream("a.txt").getChannel();
    FileChannel out = new FileOutputStream("b.txt").getChannel();

    ByteBuffer buffer = ByteBuffer.allocate(1024);

    // 调用 channel 的 read 方法往 Buffer 中写数据
    while(in.read(buffer) != -1) {
        // 在从 Buffer 中读数据之前，调用 flip()
        buffer.flip();
        // 从 Buffer 中读数据，写入到 channel
        out.write(buffer);
        // 在往 Buffer 中写数据之前，调用 clear()
        buffer.clear();
    }

    // 或者使用如下代码
    // out.transferFrom(in, 0, in.size());
}
```

### SocketChannel 连接到 TCP 套接字的通道

**SocketChannel 可以设置为阻塞模式或非阻塞模式**
使用 SocketChannel 来建立 TCP 连接，发送并接收数据，默认使用 **阻塞模式**：

```java
public static void main(String[] args) throws Exception {
    // 打开 SocketChannel
    SocketChannel channel = SocketChannel.open();
    // connect 方法会阻塞，直至连接建立成功
    channel.connect(new InetSocketAddress("127.0.0.1", 8080));

    ByteBuffer buffer = ByteBuffer.allocate(1024);

    // 发送数据
    String msg = "This is client.";
    // 在往 Buffer 中写数据之前，调用 clear()
    buffer.clear();
    buffer.put(msg.getBytes());

    // 在从 Buffer 中读数据之前，调用 flip()
    buffer.flip();
    channel.write(buffer);

    // 接收数据
    // 在往 Buffer 中写数据之前，调用 clear()
    buffer.clear();

    // 调用 channel 的 read 方法往 Buffer 中写数据
    channel.read(buffer);

    // 在从 Buffer 中读数据之前，调用 flip()
    buffer.flip();

    // 从 Buffer 中读数据
    while (buffer.hasRemaining()) {
        System.out.print(buffer.get());
    }
}
```

使用 SocketChannel 的 **非阻塞模式** 来建立 TCP 连接，发送并接收数据：

```java
public static void main(String[] args) throws Exception {
    // 打开 SocketChannel
    SocketChannel channel = SocketChannel.open();

    channel.configureBlocking(false);
    channel.connect(new InetSocketAddress("127.0.0.1", 8080));

    ByteBuffer buffer = ByteBuffer.allocate(1024);

    while (!channel.finishConnect()) {
        // 发送数据
        String msg = "This is client.";
        // 在往 Buffer 中写数据之前，调用 clear()
        buffer.clear();
        buffer.put(msg.getBytes());

        // 在从 Buffer 中读数据之前，调用 flip()
        buffer.flip();
        channel.write(buffer);

        // 接收数据
        // 在往 Buffer 中写数据之前，调用 clear()
        buffer.clear();

        // 调用 channel 的 read 方法往 Buffer 中写数据
        channel.read(buffer);

        // 在从 Buffer 中读数据之前，调用 flip()
        buffer.flip();

        // 从 Buffer 中读数据
        while (buffer.hasRemaining()) {
            System.out.print(buffer.get());
        }
    }
}
```

### ServerSocketChannel 监听 TCP 连接的通道

**ServerSocketChannel 可以设置为阻塞模式或非阻塞模式**
使用 ServerSocketChannel 来监听 TCP 连接，默认使用 **阻塞模式**：

```java
public static void main(String[] args) throws Exception {
    // 打开 SocketChannel
    ServerSocketChannel channel = ServerSocketChannel.open();
    // 绑定端口
    channel.socket().bind(new InetSocketAddress(8080));

    ByteBuffer buffer = ByteBuffer.allocate(1024);

    while (true) {
        // accept 方法会阻塞，直至监听到 TCP 连接
        SocketChannel socketChannel = channel.accept();
        System.out.println("A new connection...");

        // 接收数据
        // 在往 Buffer 中写数据之前，调用 clear()
        buffer.clear();

        // 调用 channel 的 read 方法往 Buffer 中写数据
        socketChannel.read(buffer);

        // 在从 Buffer 中读数据之前，调用 flip()
        buffer.flip();

        // 从 Buffer 中读数据
        while (buffer.hasRemaining()) {
            System.out.print(buffer.get());
        }

        // 在往 Buffer 中写数据之前，调用 clear()
        // 发送数据
        String msg = "This is server.";
        // 在往 Buffer 中写数据之前，调用 clear()
        buffer.clear();
        buffer.put(msg.getBytes());

        // 在从 Buffer 中读数据之前，调用 flip()
        buffer.flip();
        socketChannel.write(buffer);
    }
}
```

## Selector 选择器

**Selector 允许单个进程可以同时处理多个网络连接的 IO，即监听多个端口的 Channel**。

**关于 IO 模式，参见 Linux IO 模型 中对多路复用 IO Multiplexing IO 的说明。**

引用：

------

## 多路复用 IO Multiplexing IO

- **单个进程可以同时处理多个网络连接的 IO，即监听多个端口的 IO**
- 适用于连接数很高的情况
- 实现方式：select，poll，epoll 系统调用
  - 注册多个端口的监听 Socket，比如 8080，8081
  - 当用户进程调用 select 方法后，整个用户进程被阻塞，OS 内核会监听所有注册的 Socket
  - 当任何一个端口的 Socket 中的数据准备好了（ 8080 或者 8081），select 方法就会返回
  - 随后用户进程再调用 read 操作，将数据从 OS 内核缓存区拷贝到应用程序的地址空间。
- 多路复用 IO 类似于 多线程结合阻塞 IO
  - 要实现监听多个端口的 IO，还可以通过多线程的方式，每一个线程负责监听一个端口的 IO
  - 如果处理的连接数不是很高的话，使用 多路复用 IO 不一定比使用 **多线程结合阻塞 IO** 的服务器性能更好，可能延迟还更大
  - 多路复用 IO 的优势并不是对于单个连接能处理得更快，而是在于能处理更多的连接

------

**Selector 使用步骤：**

- **创建 Selector**

- **创建 Channel**，可以创建多个 Channel，即监听多个端口，比如 8080，8081

- 将 Channel 注册到 Selector 中

  - 如果一个 Channel 要注册到 Selector 中, 那么这个 Channel 必须是非阻塞的, 即 `channel.configureBlocking(false);`

  - 因此 FileChannel 是不能够使用 Selector 的, 因为 FileChannel 都是阻塞的

  - 注册时，需要指定了对 Channel 的什么事件感兴趣，包括：

    - SelectionKey.OP_CONNECT：TCP 连接 `static final int OP_CONNECT = 1 << 3;`
    - SelectionKey.OP_ACCEPT：确认 `static final int OP_ACCEPT = 1 << 4;`
    - SelectionKey.OP_READ：读 `static final int OP_READ = 1 << 0;`
    - SelectionKey.OP_WRITE：写 `static final int OP_WRITE = 1 << 2;`
    - 可以使用或运算 **|** 来组合，例如 `SelectionKey.OP_READ | SelectionKey.OP_WRITE`

  - register 方法返回一个 SelectionKey 对象，包括：

    - `int interestOps()`：调用 register 注册 channel 时所设置的 interest set.

    - ```java
      int readyOps()
      ```

      ：Channel 所准备好了的操作

      - `selectionKey.isAcceptable();`
      - `selectionKey.isConnectable();`
      - `selectionKey.isReadable();`
      - `selectionKey.isWritable();`

    - `public abstract SelectableChannel channel();`： 得到 Channel

    - `public abstract Selector selector();`：得到 Selector

    - `public final Object attachment`：得到附加对象

- 不断重复：

  - 调用 Selector 对象的 select() 方法，**该方法会阻塞，直至注册的事件发生**
  - **事件发生**，调用 Selector 对象的 selectedKeys() 方法获取 selected keys
  - 遍历每个 selected key:
    - 从 selected key 中获取对应的 Channel 并处理
    - 在 OP_ACCEPT 事件中, 从 key.channel() 返回的是 ServerSocketChannel
    - 在 OP_WRITE 和 OP_READ 事件中, 从 key.channel() 返回的是 SocketChannel

- **关闭 Selector**

示例：

```java
public static void main(String args[]) throws Exception {
    // 创建 Selector
    Selector selector = Selector.open();

    // 创建 Server Socket，监听端口 8080
    ServerSocketChannel serverChannel1 = ServerSocketChannel.open();
    serverChannel1.socket().bind(new InetSocketAddress(8080));
    // 如果一个 Channel 要注册到 Selector 中, 那么这个 Channel 必须是非阻塞的
    serverChannel1.configureBlocking(false);

    // 创建 Server Socket，监听端口 8081
    ServerSocketChannel serverChannel2 = ServerSocketChannel.open();
    serverChannel2.socket().bind(new InetSocketAddress(8081));
    // 如果一个 Channel 要注册到 Selector 中, 那么这个 Channel 必须是非阻塞的
    serverChannel2.configureBlocking(false);

    // 将 Channel 注册到 Selector 中
    serverChannel1.register(selector, SelectionKey.OP_ACCEPT);
    serverChannel2.register(selector, SelectionKey.OP_ACCEPT);

    // 不断重复
    while (true) {
        // 调用 Selector 对象的 select() 方法，该方法会阻塞，直至注册的事件发生
        selector.select();

        // 事件发生，调用 Selector 对象的 selectedKeys() 方法获取 selected keys
        Iterator<SelectionKey> it = selector.selectedKeys().iterator();

        // 遍历每个 selected key:
        while (it.hasNext()) {
            SelectionKey key = it.next();

            if (key.isAcceptable()) {
                // 在 OP_ACCEPT 事件中, 从 key.channel() 返回的是 ServerSocketChannel
                ServerSocketChannel serverChannel = (ServerSocketChannel) key.channel();

                // 调用 accept 方法获取 TCP 连接 SocketChanne
                SocketChannel clientChannel = serverChannel.accept();
                clientChannel.configureBlocking(false);

                // 注册 SocketChannel
                clientChannel.register(key.selector(), SelectionKey.OP_READ | SelectionKey.OP_WRITE);

                System.out.println("Accept event");
            }

            if (key.isReadable()) {
                // 在 OP_WRITE 和 OP_READ 事件中, 从 key.channel() 返回的是 SocketChannel
                SocketChannel clientChannel = (SocketChannel) key.channel();
                System.out.println("Read event");
                // 可以从 clientChannel 中读数据，通过 ByteBuffer
                // TO DO
            }

            if (key.isWritable()) {
                // 在 OP_WRITE 和 OP_READ 事件中, 从 key.channel() 返回的是 SocketChannel
                SocketChannel clientChannel = (SocketChannel) key.channel();
                System.out.println("Write event");
                // 可以向 clientChannel 中写数据，通过 ByteBuffer
                // TO DO
            }
        }
    }
}
```