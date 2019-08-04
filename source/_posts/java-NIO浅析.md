---
title: java NIO浅析
date: 2019-05-25 09:32:51
cover: /img/Random-img/55.jpg
categories: 
- java
tags:
- IO
- NIO
---

> NIO（Non-blocking I/O，在Java领域，也称为New I/O），是一种同步非阻塞的I/O模型，也是I/O多路复用的基础，已经被越来越多地应用到大型应用服务器，成为解决高并发与大量连接、I/O处理问题的有效方式。

NIO主要有三大核心部分：**Channel(通道)**，**Buffer(缓冲区)**，**Selector(选择器)**。传统IO基于字节流和字符流进行操作，而NIO基于Channel和Buffer进行操作，数据总是从通道读取到缓冲区中，或者从缓冲区写入到通道中。Selector用于监听多个通道的事件（比如：连接打开，数据到达）。因此，**单个线程可以监听多个数据通道**。

## NIO与IO的主要区别
- **IO是面向流的，NIO是面向缓冲区的**。Java IO面向流意味着每次只能从流中读取一个或多个字节，直到读取完所有字节，它们没有被缓存在任何地方。此外，它不能前后移动流中的数据。如果需要前后移动从流中读取的数据，需要先将它缓存到一个缓冲区。NIO的缓冲区导向方法略有不同。数据读取到一个它稍后处理的缓冲区，需要时可在缓冲区中前后移动，而且，需确保当更多的数据读入缓冲区时，不能覆盖掉缓冲区尚未处理的数据。
- **IO是阻塞的，NIO是同步非阻塞的**。IO的各种流是阻塞的，这意味着，当一个线程调用read()或write()方法时，该线程被阻塞，直到有一些数据被读取，或数据完全写入。该线程在此期间不能再干任何事了，NIO的非阻塞模式，使一个线程从某通道发送请求读取数据，但是它仅能得到目前可用的数据，如果目前没有数据可用时，就什么都不会获取，而是保持线程阻塞，所以直至数据变到可以读取之前，该线程可以继续做其他事情。非阻塞写也是如此，一个线程请求写入一些数据到某通道，但不需要等待它完全写入，这个线程同时可以去做别的事情。线程通常将非阻塞IO的空闲时间用于在其他通道上执行IO操作，所以**一个单独线程可以管理多个输入输出通道（channel）**。

## NIO
> - &emsp;NIO(New IO)是一个可以替代标准Java IO API的IO API（从Java 1.4开始)，Java NIO提供了与标准IO不同的IO工作方式。
> - &emsp;Java NIO可以让你非阻塞的使用IO，例如：当线程从通道读取数据到缓冲区时，线程还是可以进行其他事情。当数据被写入到缓冲区时，线程可以继续处理它。从缓冲区写入通道也类似。
> - &emsp;Java NIO引入了选择器的概念，选择器用于监听多个通道的事件（比如：连接打开，数据到达）。因此，单个的线程可以监听多个数据通道。
		
## 缓冲区(Buffer)：
为什么说NIO是基于缓冲区的IO方式呢？因为，当一个连接建立完成后，IO的数据未必会马上到达，为了当数据到达时能够正确完成IO操作，在BIO（阻塞IO）中，等待IO的线程必须被阻塞，以全天候地执行IO操作。为了解决这种IO方式低效的问题，引入了缓冲区的概念，**当数据到达时，可以预先被写入缓冲区，再由缓冲区交给线程，因此线程无需阻塞地等待IO**。

缓冲区分类：
- **ByteBuffer：字节缓冲区** 
- MappedByteBuffer：直接字节缓冲区，其内容是文件的内存映射区域
- CharBuffer：字符缓冲区
- DoubleBuffer：double缓冲区
- FloatBuffer：float缓冲区
- IntBuffer：int缓冲区
- LongBuffer：long缓冲区
- ShortBuffer：short缓冲区

常用属性：

|  序号   |   属性描述  |
| --- | --- |
|  1 |  capacity<br>缓冲区大小，无论是读模式还是写模式，此属性值不会变   |
|   2  |  position<br>写数据时，position表示当前写的位置，每写一个数据，会向下移动一个数据单元，初始为0；最大为capacity - 1切换到读模式时，position会被置为0，表示当前读的位置   |
|   3  |  limit<br>写模式下，limit 相当于capacity 表示最多可以写多少数据，切换到读模式时，limit 等于原先的position，表示最多可以读多少数据。   |

常用方法：

|   序号  |  方法描述   |
| --- | --- |
| 1   |   public static ByteBuffer allocate(int capacity)<br>分配一个新的字节缓冲区  |
| 2   | public ByteBuffer get(byte[]dst)<br>此方法将此缓冲区的字节传输到给定的目标数组中。                                             |
| 3   | public ByteBuffer put(byte[]dst)<br>此方法将给定的源 byte 数组的所有内容传输到此缓冲区中。                                     |
| 4   | public final Buffer flip();<br>将缓冲区从写模式切换到读模式                                                                    |
| 5   | public Buffer clear();<br>从读模式切换到写模式，不会清空数据，但后续写数据会覆盖原来的数据，即使有部分数据没有读，也会被遗忘。 |

## 通道(Channel)：
> 类似于流，但是可以异步读写数据（流只能同步读写），通道是双向的（流是单向的），通道的数据总是要先读到一个buffer 或者 从一个buffer写入，即通道与buffer进行数据交互。

**注意：读的时候不能写，写的时候不能读，如果需要必须切换状态**

- FileChannel：从文件中读写数据。非异步，阻塞
- DatagramChannel：能通过UDP读写网络中的数据。
- SocketChannel：能通过TCP读写网络中的数据。
>创建一个SocketChannel：`SocketChannel channel = SocketChannel.open();`
请求服务器：`channel.connect(IP,Port);`
- ServerSocketChannel：可以监听新进来的TCP连接，像Web服务器那样。对每一个新进来的连接都会创建一个SocketChannel。
> 创建一个ServerSocketChannel：`ServerSocketChannel ssc = ServerSocketChannel.open();`
> 绑定指定端口号9999：`ssc.socket().bind(new InetSocketAddress(9999));`
关闭ServerSocket：`Channelssc.close();`
非阻塞模式false(阻塞模式true)：`ssc.configureBlocking(false);`
监听新进来的channel(客户端的channel)：`SocketChannel channel = ssc.accept();`
注意：**一般都是死循环方式，一直监听，如果channel非null，则获取到客户端的channel，否则继续监听**

**注意：FileChannel比较特殊，它可以与通道进行数据交互， 不能切换到非阻塞模式，套接字通道可以切换到非阻塞模式；**

## 通道和缓冲区：
> 基本上，所有的IO在NIO中都从一个Channel开始。Channel有点像流。 数据可以从Channel读到Buffer中，也可以从Buffer 写到Channel中。

## 选择器：
> Selector允许单线程处理多个 Channel。如果你的应用打开了多个连接（通道），但每个连接的流量都很低，使用Selector就会很方便。例如，在一个聊天服务器中。

这是在一个单线程中使用一个Selector处理3个Channel的图示：
![selector图示](/img/post-img/19-5-25-1.png)

要使用Selector，得向Selector注册Channel，然后调用它的select()方法。这个方法会一直阻塞到某个注册的通道有事件就绪。一旦这个方法返回，线程就可以处理这些事件，事件的例子有如新连接进来，数据接收等。

**选择器相当于一个观察者，用来监听通道感兴趣的事件，一个选择器可以绑定多个通道**。

``` java
// 1. 选择器的创建
Selector selector=Selector.open();
// 2. 向选择器注册通道
channel.configureBlocking(false);
SelectionKey key = channel.register(selector,SelectionKey.事件);
// 一共有四种事件
// SelectionKey.OP_CONNECT  ：请求连接
// SelectionKey.OP_ACCEPT  ：接收请求
// SelectionKey.OP_READ  ：读取
// SelectionKey.OP_WRITE  ：写入
// 注意：使用选择器时，通道必须是非阻塞的
selector.select(); //代表注册的事件发生是可以执行它下边的代码，否则阻塞
// 3. SelectionKey
ServerSocketChannel ssc = (强转)key.channel();
SocketChannel sc = (强转)key.channel();
// 4. 通过Selector选择通道
//一旦向Selector注册了一或多个通道，就可以调用几个重载的select()方法。这些方法返回你所感兴趣的事件（如连接、接受、读或写）已经准备就绪的那些通道。换句话说，如果你对“读就绪”的通道感兴趣，select()方法会返回读事件已经就绪的那些通道。下面是select()方法：
int select(); // 阻塞到至少有一个通道在你注册的事件上就绪了。
int select(long timeout); // 和select()一样，除了最长会阻塞timeout毫秒(参数)
int selectNow(); // 不会阻塞，不管什么通道就绪都立刻返回.（注：此方法执行非阻塞的选择操作。如果自从前一次选择操作后，没有通道变成可选择的，则此方法直接返回零）。
/**select()方法返回的int值表示有多少通道已经就绪。亦即，自上次调用select()方法后有多少通道变成就绪状态。如果调用select()方法，因为有一个通道变成就绪状态，返回了1，若再次调用select()方法，如果另一个通道就绪了，它会再次返回1。如果对第一个就绪的channel没有做任何操作，现在就有两个就绪的通道，但在每次select()方法调用之间，只有一个通道就绪了。*/

// 5. 关闭选择器
// 用完Selector后调用其close()方法会关闭该Selector，且使注册到该Selector上的所有SelectionKey实例无效。通道本身并不会关闭。
```


相关链接：[美团技术团队-java NIO浅析](https://tech.meituan.com/nio.html)