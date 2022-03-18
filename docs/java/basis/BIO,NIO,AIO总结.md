熟练掌握 BIO,NIO,AIO 的基本概念以及一些常见问题是你准备面试的过程中不可或缺的一部分，另外这些知识点也是你学习 Netty 的基础。

<!-- MarkdownTOC -->

- [BIO,NIO,AIO 总结](#bionioaio-总结)
  - [1. BIO \(Blocking I/O\)](#1-bio-blocking-io)
    - [1.1 传统 BIO](#11-传统-bio)
    - [1.2 伪异步 IO](#12-伪异步-io)
    - [1.3 代码示例](#13-代码示例)
    - [1.4 总结](#14-总结)
  - [2. NIO \(New I/O\)](#2-nio-new-io)
    - [2.1 NIO 简介](#21-nio-简介)
    - [2.2 NIO的特性/NIO与IO区别](#22-nio的特性nio与io区别)
      - [1)Non-blocking IO（非阻塞IO）](#1non-blocking-io非阻塞io)
      - [2)Buffer\(缓冲区\)](#2buffer缓冲区)
      - [3)Channel \(通道\)](#3channel-通道)
      - [4)Selectors\(选择器\)](#4selector-选择器)
    - [2.3  NIO 读数据和写数据方式](#23-nio-读数据和写数据方式)
    - [2.4 NIO核心组件简单介绍](#24-nio核心组件简单介绍)
    - [2.5 代码示例](#25-代码示例)
  - [3. AIO  \(Asynchronous I/O\)](#3-aio-asynchronous-io)
  - [参考](#参考)

<!-- /MarkdownTOC -->


# BIO,NIO,AIO 总结

 Java 中的 BIO、NIO和 AIO 理解为是 Java 语言对操作系统的各种 IO 模型的封装。程序员在使用这些 API 的时候，不需要关心操作系统层面的知识，也不需要根据不同操作系统编写不同的代码。只需要使用Java的API就可以了。

在讲 BIO,NIO,AIO 之前先来回顾一下这样几个概念：同步与异步，阻塞与非阻塞。

关于同步和异步的概念解读困扰着很多程序员，大部分的解读都会带有自己的一点偏见。参考了 [Stackoverflow](https://stackoverflow.com/questions/748175/asynchronous-vs-synchronous-execution-what-does-it-really-mean)相关问题后对原有答案进行了进一步完善：

> When you execute something synchronously, you wait for it to finish before moving on to another task. When you execute something asynchronously, you can move on to another task before it finishes.
>
> 当你同步执行某项任务时，你需要等待其完成才能继续执行其他任务。当你异步执行某些操作时，你可以在完成另一个任务之前继续进行。

- **同步** ：两个同步任务相互依赖，并且一个任务必须以依赖于另一任务的某种方式执行。 比如在`A->B`事件模型中，你需要先完成 A 才能执行B。 再换句话说，同步调用中被调用者未处理完请求之前，调用不返回，==调用者会一直等待结果的返回==。
- **异步**： 两个异步的任务是完全独立的，一方的执行不需要等待另外一方的执行。再换句话说，异步调用中一调用就返回结果不需要等待结果返回，当结果返回的时候通过==回调函数==或者其他方式拿着结果再做相关事情，

**阻塞和非阻塞**

- **阻塞：** 阻塞就是发起一个请求，调用者==一直等待请求结果返回==，也就是当前线程==会被挂起==，无法从事其他任务，只有当条件就绪才能继续。
- **非阻塞：** 非阻塞就是发起一个请求，调用者不用一直等着结果返回，可以先去干其他事情。

**如何区分 “同步/异步 ”和 “阻塞/非阻塞” 呢？**

同步/异步是从行为角度描述事物的，而阻塞和非阻塞描述的==当前事物的状态==（等待调用结果时的状态）。

## 1. BIO (Blocking I/O)

同步阻塞I/O模式，数据的读取写入必须阻塞在一个线程内等待其完成。

### 1.1 传统 BIO

BIO通信（一请求一应答）模型图如下(图源网络，原出处不明)：

![传统BIO通信模型图](https://my-blog-to-use.oss-cn-beijing.aliyuncs.com/2.png)

采用 **BIO 通信模型** 的服务端，通常由一个独立的 ==Acceptor 线程==负责==监听客户端的连接==。我们一般通过在`while(true)` 循环中服务端会调用 ==`accept()`== 方法==等待接收==客户端的连接的方式监听==请求==，请求一旦接收到一个==连接请求==，就可以建立通信套接字在==这个通信套接字上进行读写操作==，此时不能再接收其他客户端连接请求，只能等待同当前连接的客户端的操作执行完成， 不过可以通过多线程来支持多个客户端的连接，如上图所示。

如果要让 **BIO 通信模型** 能够同时处理多个客户端请求，就必须使用多线程（主要原因是`socket.accept()`、`socket.read()`、`socket.write()` 涉及的==三个主要函数都是同步阻塞==的），也就是说它在==接收到客户端连接请求之后==为==每个客户端创建一个新的线程==进行链路处理，处理完成之后，通过输出流返回应答给客户端，线程销毁。这就是典型的 **一请求一应答通信模型** 。我们可以设想一下如果这个连接不做任何事情的话就会造成不必要的线程开销，不过可以通过 **线程池机制** 改善，线程池还可以让线程的创建和回收成本相对较低。使用==`FixedThreadPool`== 可以有效的控制了线程的最大数量，保证了系统有限的资源的控制，实现了==N(客户端请求数量):M(处理客户端请求的线程数量==)的==伪异步I/O==模型（N 可以远远大于 M），下面一节"伪异步 BIO"中会详细介绍到。



**我们再设想一下当客户端并发访问量增加后这种模型会出现什么问题？**

在 Java 虚拟机中，线程是宝贵的资源，线程的创建和销毁成本很高，除此之外，线程的切换成本也是很高的。尤其在 Linux 这样的操作系统中，线程本质上就是一个进程，==创建和销毁线程都是重量级的系统函数==。如果并发访问量增加会导致线程数急剧膨胀可能会导致==线程堆栈溢出==、==创建新线程失败==等问题，最终导致进程宕机或者僵死，不能对外提供服务。

### 1.2 伪异步 IO

为了解决同步阻塞I/O面临的一个链路需要一个线程处理的问题，后来有人对它的线程模型进行了优化一一一后端通过一个线程池来处理多个客户端的请求接入，形成客户端个数M：线程池最大线程数N的比例关系，其中M可以远远大于N.通过线程池可以灵活地调配线程资源，设置线程的最大值，防止由于海量并发接入导致线程耗尽。

伪异步IO模型图(图源网络，原出处不明)：

![伪异步IO模型图](https://my-blog-to-use.oss-cn-beijing.aliyuncs.com/3.png)

采用线程池和任务队列可以实现一种叫做伪异步的 I/O 通信框架，它的模型图如上图所示。当有新的客户端接入时，将客户端的 ==Socket 封装成一个Task==（该任务实现java.lang.Runnable接口）==投递到后端的线程池==中进行处理，JDK 的==线程池维护一个消息队列和 N 个活跃线程==，==对消息队列中的任务==进行处理。由于线程池可以设置消息队列的大小和最大线程数，因此，它的资源占用是可控的，无论多少个客户端并发访问，都不会导致资源的耗尽和宕机。

伪异步I/O通信框架采用了线程池实现，因此避免了为每个请求都创建一个独立线程造成的线程资源耗尽问题。不过因为它的底层仍然是同步阻塞的BIO模型，因此无法从根本上解决问题。

### 1.3 代码示例

下面代码中演示了BIO通信（一请求一应答）模型。我们会在客户端创建多个线程依次连接服务端并向其发送"当前时间+:hello world"，服务端会为每个客户端线程创建一个线程来处理。代码示例出自闪电侠的博客，原地址如下：        

[https://www.jianshu.com/p/a4e03835921a](https://www.jianshu.com/p/a4e03835921a)

**客户端**

```java
/**
 * 
 * @author 闪电侠
 * @date 2018年10月14日
 * @Description:客户端
 */
public class IOClient {

  public static void main(String[] args) {
    // TODO 创建多个线程，模拟多个客户端连接服务端
    new Thread(() -> {
      try {
        Socket socket = new Socket("127.0.0.1", 3333);
        while (true) {
          try {
            socket.getOutputStream().write((new Date() + ": hello world").getBytes());
            Thread.sleep(2000);
          } catch (Exception e) {
          }
        }
      } catch (IOException e) {
      }
    }).start();

  }

}

```

**服务端**

```java
/**
 * @author 闪电侠
 * @date 2018年10月14日
 * @Description: 服务端
 */
public class IOServer {

  public static void main(String[] args) throws IOException {
    // TODO 服务端处理客户端连接请求
    ServerSocket serverSocket = new ServerSocket(3333);

    // 接收到客户端连接请求之后为每个客户端创建一个新的线程进行链路处理
    new Thread(() -> {
      while (true) {
        try {
          // 阻塞方法获取新的连接
          Socket socket = serverSocket.accept();

          // 每一个新的连接都创建一个线程，负责读取数据
          new Thread(() -> {
            try {
              int len;
              byte[] data = new byte[1024];
              InputStream inputStream = socket.getInputStream();
              // 按字节流方式读取数据
              while ((len = inputStream.read(data)) != -1) {
                System.out.println(new String(data, 0, len));
              }
            } catch (IOException e) {
            }
          }).start();

        } catch (IOException e) {
        }

      }
    }).start();

  }

}
```

### 1.4 总结

在活动==连接数不是特别高==（小于单机1000）的情况下，这种模型是比较不错的，可以让每一个连接专注于自己的 I/O 并且编程模型简单，也==不用过多考虑系统的过载、限流==等问题。线程池本身就是一个天然的漏斗，可以缓冲一些系统处理不了的连接或请求。但是，当面对十万甚至百万级连接的时候，传统的 BIO 模型是无能为力的。因此，我们需要一种更高效的 I/O 处理模型来应对更高的并发量。

## 2. NIO (New I/O)

### 2.1 NIO 简介

 NIO是一种==同步非阻塞的I/O==模型，在Java 1.4 中引入了 NIO 框架，对应 java.nio 包，提供了 Channel , Selector，Buffer等抽象。

NIO中的N可以理解为Non-blocking，不单纯是New。它支持面向缓冲的，基于通道的I/O操作方法。 NIO提供了与传统BIO模型中的 `Socket` 和 `ServerSocket` 相对应的 `SocketChannel` 和 `ServerSocketChannel` 两种不同的套接字通道实现,两种通道都支持阻塞和非阻塞两种模式。阻塞模式使用就像传统中的支持一样，比较简单，但是性能和可靠性都不好；非阻塞模式正好与之相反。对于低负载、低并发的应用程序，可以使用同步阻塞I/O来提升开发速率和更好的维护性；对于==高负载、高并发==的（网络）应用，应使用 ==NIO 的非阻塞==模式来开发。

### 2.2 NIO的特性/NIO与IO区别

如果是在面试中回答这个问题，我觉得首先肯定要从 NIO 流是非阻塞 IO 而 IO 流是阻塞 IO 说起。然后，可以从 NIO 的3个核心组件/特性为 NIO 带来的一些改进来分析。如果，你把这些都回答上了我觉得你对于 NIO 就有了更为深入一点的认识，面试官问到你这个问题，你也能很轻松的回答上来了。

#### 1)Non-blocking IO（非阻塞IO）

**IO流是阻塞的，NIO流是不阻塞的。**

Java NIO使我们可以进行非阻塞IO操作。比如说，单线程中从通道读取数据到buffer，同时可以继续做别的事情，当数据读取到buffer中后，线程再继续处理数据。写数据也是一样的。另外，非阻塞写也是如此。一个线程请求写入一些数据到某通道，但==不需要等待它完全写入==，这个线程同时可以去做别的事情。

Java IO的各种流是阻塞的。这意味着，当一个线程调用 ==`read()` 或  `write()` 时==，该线程==被阻塞==，直到有一些==数据被读取，或数据完全写入==。该线程在==此期间不能==再干任何事情了

#### 2)Buffer(缓冲区)

**IO 面向流(Stream oriented)，而 NIO 面向缓冲区(Buffer oriented)。**

Buffer是一个对象，它包含一些要写入或者要读出的数据。在==NIO类库==中加入==Buffer对象==，体现了新库与原I/O的一个重要区别。在面向流的I/O中·可以将数据直接写入或者将数据直接读到 Stream 对象中。虽然 Stream 中也有 Buffer 开头的扩展类，但只是流的包装类，还是从流读到缓冲区，而 ==NIO 却是直接读到 Buffer 中==进行操作。

在NIO厍中，所有数据都是用缓冲区处理的。在读取数据时，它是直接读到缓冲区中的; 在写入数据时，写入到缓冲区中。任何时候访问NIO中的数据，都是通过缓冲区进行操作。

最常用的缓冲区是 ByteBuffer,一个 ByteBuffer 提供了一组功能用于操作 byte 数组。除了ByteBuffer,还有其他的一些缓冲区，事实上，每一种Java基本类型（除了Boolean类型）都对应有一种缓冲区。

#### 3)Channel (通道)

NIO 通过==Channel（通道） 进行读写==。

通道是双向的，可读也可写，而流的读写是单向的。无论读写，通道只能和Buffer交互。因为 Buffer，通道可以异步地读写。

#### 4)Selector (选择器)

NIO有选择器，而IO没有。

选择器用于使用==单个线程处理多个通道==。因此，它需要较少的线程来处理这些通道。线程之间的切换对于操作系统来说是昂贵的。 因此，为了提高系统效率选择器是有用的。

![一个单线程中Selector维护3个Channel的示意图](https://my-blog-to-use.oss-cn-beijing.aliyuncs.com/2019-2/Slector.png)

### 2.3  NIO 读数据和写数据方式
通常来说NIO中的==所有IO都是从 Channel（通道） 开始==的。

- 从通道进行数据读取 ：创建一个缓冲区，然后==请求通道读取==数据。
- 从通道进行数据写入 ：创建一个缓冲区，填充数据，并要求==通道写入==数据。

数据读取和写入操作图示：

![NIO读写数据的方式](https://my-blog-to-use.oss-cn-beijing.aliyuncs.com/2019-2/NIO读写数据的方式.png)


### 2.4 NIO核心组件简单介绍

NIO 包含下面几个核心的组件：

- Channel(通道)
- Buffer(缓冲区)
- Selector(选择器)

整个NIO体系包含的类远远不止这三个，只能说这三个是NIO体系的“核心API”。我们上面已经对这三个概念进行了基本的阐述，这里就不多做解释了。

### 2.5 代码示例

代码示例出自闪电侠的博客，原地址如下：        

[https://www.jianshu.com/p/a4e03835921a](https://www.jianshu.com/p/a4e03835921a)

客户端 IOClient.java 的代码不变，我们对服务端使用 NIO 进行改造。以下代码较多而且逻辑比较复杂，大家看看就好。

```java
/**
 * 
 * @author 闪电侠
 * @date 2019年2月21日
 * @Description: NIO 改造后的服务端
 */
public class NIOServer {
  public static void main(String[] args) throws IOException {
    // 1. serverSelector负责轮询是否有新的连接，服务端监测到新的连接之后，不再创建一个新的线程，
    // 而是直接将新连接绑定到clientSelector上，这样就不用 IO 模型中 1w 个 while 循环在死等
    Selector serverSelector = Selector.open();
    // 2. clientSelector负责轮询连接是否有数据可读
    Selector clientSelector = Selector.open();

    new Thread(() -> {
      try {
        // 对应IO编程中服务端启动
        ServerSocketChannel listenerChannel = ServerSocketChannel.open();
        listenerChannel.socket().bind(new InetSocketAddress(3333));
        listenerChannel.configureBlocking(false);
        listenerChannel.register(serverSelector, SelectionKey.OP_ACCEPT);

        while (true) {
          // 监测是否有新的连接，这里的1指的是阻塞的时间为 1ms
          if (serverSelector.select(1) > 0) {
            Set<SelectionKey> set = serverSelector.selectedKeys();
            Iterator<SelectionKey> keyIterator = set.iterator();

            while (keyIterator.hasNext()) {
              SelectionKey key = keyIterator.next();

              if (key.isAcceptable()) {
                try {
                  // (1) 每来一个新连接，不需要创建一个线程，而是直接注册到clientSelector
                  SocketChannel clientChannel = ((ServerSocketChannel) key.channel()).accept();
                  clientChannel.configureBlocking(false);
                  clientChannel.register(clientSelector, SelectionKey.OP_READ);
                } finally {
                  keyIterator.remove();
                }
              }

            }
          }
        }
      } catch (IOException ignored) {
      }
    }).start();
    new Thread(() -> {
      try {
        while (true) {
          // (2) 批量轮询是否有哪些连接有数据可读，这里的1指的是阻塞的时间为 1ms
          if (clientSelector.select(1) > 0) {
            Set<SelectionKey> set = clientSelector.selectedKeys();
            Iterator<SelectionKey> keyIterator = set.iterator();

            while (keyIterator.hasNext()) {
              SelectionKey key = keyIterator.next();

              if (key.isReadable()) {
                try {
                  SocketChannel clientChannel = (SocketChannel) key.channel();
                  ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
                  // (3) 面向 Buffer
                  clientChannel.read(byteBuffer);
                  byteBuffer.flip();
                  System.out.println(
                      Charset.defaultCharset().newDecoder().decode(byteBuffer).toString());
                } finally {
                  keyIterator.remove();
                  key.interestOps(SelectionKey.OP_READ);
                }
              }

            }
          }
        }
      } catch (IOException ignored) {
      }
    }).start();

  }
}
```

为什么大家都不愿意用 JDK 原生 NIO 进行开发呢？从上面的代码中大家都可以看出来，是真的难用！除了编程复杂、编程模型难之外，它还有以下让人诟病的问题：

- JDK 的 ==NIO 底层由 epoll 实现==，该实现饱受诟病的空轮询 bug 会导致 ==cpu 飙升 100%==
- 项目庞大之后，自行实现的 NIO ==很容易出现各类 bug==，维护成本较高，上面这一坨代码我都不能保证没有 bug

Netty 的出现很大程度上改善了 JDK 原生 NIO 所存在的一些让人难以忍受的问题。

### 3. AIO (Asynchronous I/O)

AIO 也就是 NIO 2。在 Java 7 中引入了 NIO 的改进版 NIO 2,它是==异步非阻塞的IO模型==。异步 IO 是基于事件和回调机制实现的，也就是应用操作之后会直接返回，不会堵塞在那里，当后台处理完成，操作系统会通知相应的线程进行后续的操作。

AIO 是异步IO的缩写，虽然 NIO 在网络操作中，提供了==非阻塞的方法==，但是 ==NIO 的 IO 行为还是同步的==。对于 NIO 来说，我们的业务线程是在 IO 操作准备好时，得到通知，接着就由这个线程自行进行 IO 操作，IO操作本身是同步的。（除了 AIO 其他的 IO 类型都是同步的，这一点可以从底层IO线程模型解释，推荐一篇文章：[《漫话：如何给女朋友解释什么是Linux的五种IO模型？》](https://mp.weixin.qq.com/s?__biz=Mzg3MjA4MTExMw==&mid=2247484746&amp;idx=1&amp;sn=c0a7f9129d780786cabfcac0a8aa6bb7&source=41#wechat_redirect) ）

查阅网上相关资料，我发现就目前来说 AIO 的应用还不是很广泛，Netty 之前也尝试使用过 AIO，不过又放弃了。

## 参考

- 《Netty 权威指南》第二版
- https://zhuanlan.zhihu.com/p/23488863 (美团技术团队)



# 附录

# Java的BIO和NIO很难懂？用代码实践给你看，再不懂我转行！

https://cloud.tencent.com/developer/article/1545422

# 1、引言

这段时间自己在看一些Java中BIO和NIO之类的东西，也看了很多博客，发现各种关于NIO的理论概念说的天花乱坠头头是道，可以说是非常的完整，但是整个看下来之后，发现自己对NIO还是一知半解、一脸蒙逼的状态（请原谅我太笨）。

![img](https://ask.qcloudimg.com/http-save/6708446/zre7stazrc.jpeg?imageView2/2/w/1620)

基于以上原因，就有了写本文的想法。本文不会提到很多Java NIO和Java BIO的理论概念（需要的话请参见本文的“相关文章”一节），而是站在编码实践的角度，通过代码实例，总结了我自己对于Java NIO的见解。有了代码实践的过程后再重新回头看理论概念，会有一个不一样的理解视角，希望能助你吃透它们！

**术语约定：**本文所说的BIO即Java程序员常说的经典阻塞式IO，NIO是指Java 1.4版加入的NIO（即异步IO）。

# 2、关于作者

本文作者：Object

个人博客：[http://blog.objectspace.cn/](https://links.jianshu.com/go?to=http%3A%2F%2Fblog.objectspace.cn%2F)

# 3、相关文章

本文为了避免过多的阐述Java NIO、BIO的概念性内容，因而尽量少的提及相关理论知识，如果你对Java NIO、BIO的理论知识本来就了解不多，建议还是先读一读即时通讯网整理一下文章，将有助于你更好地理解本文。

# 4、先用经典的BIO来实现一个简易的单线程网络通信程序

要讲明白BIO和NIO，首先我们应该自己实现一个简易的服务器，不用太复杂，单线程即可。

#### 4.1 为什么使用单线程作为演示

因为在单线程环境下可以很好地对比出BIO和NIO的一个区别，当然我也会演示在实际环境中BIO的所谓一个请求对应一个线程的状况。

#### 4.2 服务端代码

>  public class Server {         public static void main(String[] args) {                 byte[] buffer = new byte[1024];                 try{                         ServerSocket serverSocket = newServerSocket(8080);                         System.out.println("服务器已启动并监听8080端口");                         while(true) {                                 System.out.println();                                 System.out.println("服务器正在等待连接...");                                 Socket socket = serverSocket.accept();                                 System.out.println("服务器已接收到连接请求...");                                 System.out.println();                                 System.out.println("服务器正在等待数据...");                                 socket.getInputStream().read(buffer);                                 System.out.println("服务器已经接收到数据");                                 System.out.println();                                 String content = newString(buffer);                                 System.out.println("接收到的数据:"+ content);                         }                 } catch(IOException e) {                         // TODO Auto-generated catch block                         e.printStackTrace();                 }         } } 

#### 4.3 客户端代码

>  public class Consumer {         public static void main(String[] args) {                 try{                         Socket socket = newSocket("127.0.0.1",8080);                         socket.getOutputStream().write("向服务器发数据".getBytes());                         socket.close();                 } catch(IOException e) {                         // TODO Auto-generated catch block                         e.printStackTrace();                 }         } } 

#### 4.4 代码解析

我们首先创建了一个服务端类，在类中实现实例化了一个SocketServer并绑定了8080端口。之后调用accept方法来接收连接请求，并且调用read方法来接收客户端发送的数据。最后将接收到的数据打印。

完成了服务端的设计后，我们来实现一个客户端，首先实例化Socket对象，并且绑定ip为127.0.0.1（本机），端口号为8080，调用write方法向服务器发送数据。

![img](https://ask.qcloudimg.com/http-save/6708446/6no5cqkavv.jpeg?imageView2/2/w/1620)

#### 4.5 运行结果

当我们启动服务器，但客户端还没有向服务器发起连接时，控制台结果如下：

![img](https://ask.qcloudimg.com/http-save/6708446/u907kuy3z6.png?imageView2/2/w/1620)

当客户端启动并向服务器发送数据后，控制台结果如下：

![img](https://ask.qcloudimg.com/http-save/6708446/uixx8yr2tn.png?imageView2/2/w/1620)

#### 4.6 结论

从上面的运行结果，首先我们至少可以看到，在服务器启动后，客户端还没有连接服务器时，服务器由于调用了accept方法，将一直阻塞，直到有客户端请求连接服务器。

# 5、对客户端功能进行扩展

在上节中，我们实现的客户端的逻辑主要是：建立Socket –> 连接服务器 –> 发送数据，我们的数据是在连接服务器之后就立即发送的，现在我们来对客户端进行一次扩展，当我们连接服务器后，不立即发送数据，而是等待控制台手动输入数据后，再发送给服务端。（注意：本节中，服务端代码保持不变）

#### 5.1 改进后的代码

>  public class Consumer {         public static void main(String[] args) {                 try{                         Socket socket = newSocket("127.0.0.1",8080);                         String message = null;                         Scanner sc = newScanner(System.in);                         message = sc.next();                         socket.getOutputStream().write(message.getBytes());                         socket.close();                         sc.close();                 } catch(IOException e) {                         // TODO Auto-generated catch block                         e.printStackTrace();                 }         } } 

#### 5.2 测试

当服务端启动，客户端还没有请求连接服务器时，控制台结果如下：

![img](https://ask.qcloudimg.com/http-save/6708446/es0bymnf7o.png?imageView2/2/w/1620)

当服务端启动，客户端连接服务端，但没有发送数据时，控制台结果如下：

![img](https://ask.qcloudimg.com/http-save/6708446/ihj9veq2ly.png?imageView2/2/w/1620)

当服务端启动，客户端连接服务端，并且发送数据时，控制台结果如下：

![img](https://ask.qcloudimg.com/http-save/6708446/n5pat77kxb.png?imageView2/2/w/1620)

#### 5.3 结论

**从上面的运行结果中我们可以看到，服务器端在启动后：**

>  1）首先需要等待客户端的连接请求（第一次阻塞）； 2）如果没有客户端连接，服务端将一直阻塞等待； 3）然后当客户端连接后，服务器会等待客户端发送数据（第二次阻塞）； 4）如果客户端没有发送数据，那么服务端将会一直阻塞等待客户端发送数据。 

**服务端从启动到收到客户端数据的这个过程，将会有两次阻塞的过程：**

>  1）第一次在等待连接时阻塞； 2）第二次在等待数据时阻塞。 

BIO会产生两次阻塞，这就是BIO的非常重要的一个特点。

# 6、BIO

#### 6.1 在单线程条件下BIO的弱点

在上两节中，我们用经典的Java BIO实现了一个简易的网络通信程序，这个简易的程序是以单线程运行的。

**其实我们不难看出：**当我们的服务器接收到一个连接后，并且没有接收到客户端发送的数据时，是会阻塞在read()方法中的，那么此时如果再来一个客户端的请求，服务端是无法进行响应的。**换言之：**在不考虑多线程的情况下，BIO是无法处理多个客户端请求的。

#### 6.2 BIO如何处理并发

在上面的服务器实现中，我们实现的是单线程版的BIO服务器，不难看出，单线程版的BIO并不能处理多个客户端的请求，那么如何能使BIO处理多个客户端请求呢。

**其实不难想到：**我们只需要在每一个连接请求到来时，创建一个线程去执行这个连接请求，就可以在BIO中处理多个客户端请求了，这也就是为什么BIO的其中一条概念是服务器实现模式为一个连接一个线程，即客户端有连接请求时服务器端就需要启动一个线程进行处理。

#### 6.3 多线程BIO服务器简易实现

>  public class Server {         public static void main(String[] args) {                 byte[] buffer = newbyte[1024];                 try{                         ServerSocket serverSocket = newServerSocket(8080);                         System.out.println("服务器已启动并监听8080端口");                         while(true) {                                 System.out.println();                                 System.out.println("服务器正在等待连接...");                                 Socket socket = serverSocket.accept();                                 newThread(newRunnable() {                                         @Override                                         publicvoidrun() {                                                 System.out.println("服务器已接收到连接请求...");                                                 System.out.println();                                                 System.out.println("服务器正在等待数据...");                                                 try{                                                         socket.getInputStream().read(buffer);                                                 } catch(IOException e) {                                                         // TODO Auto-generated catch block                                                         e.printStackTrace();                                                 }                                                 System.out.println("服务器已经接收到数据");                                                 System.out.println();                                                 String content = newString(buffer);                                                 System.out.println("接收到的数据:"+ content);                                         }                                 }).start();                           }                 } catch(IOException e) {                         // TODO Auto-generated catch block                         e.printStackTrace();                 }         } } 

#### 6.4 运行结果

![img](https://ask.qcloudimg.com/http-save/6708446/a6zx8hnl5k.png?imageView2/2/w/1620)

![img](https://ask.qcloudimg.com/http-save/6708446/cuemb8mryu.png?imageView2/2/w/1620)

很明显，现在我们的服务器的状态就是一个线程对应一个请求，换言之，服务器为每一个连接请求都创建了一个线程来处理。

#### 6.5 多线程BIO服务器的弊端

**多线程BIO服务器虽然解决了单线程BIO无法处理并发的弱点，但是也带来一个问题：**如果有大量的请求连接到我们的服务器上，但是却不发送消息，那么我们的服务器也会为这些不发送消息的请求创建一个单独的线程，那么如果连接数少还好，连接数一多就会对服务端造成极大的压力。

**所以：**如果这种不活跃的线程比较多，我们应该采取单线程的一个解决方案，但是单线程又无法处理并发，这就陷入了一种很矛盾的状态，于是就有了NIO。

# 7、NIO

#### 7.1 NIO的引入

我们先来看看单线程模式下BIO服务器的代码，其实NIO需要解决的最根本的问题就是存在于BIO中的两个阻塞，分别是等待连接时的阻塞和等待数据时的阻塞。

>  public class Server {         public static void main(String[] args) {                 byte[] buffer = new byte[1024];                 try{                         ServerSocket serverSocket = newServerSocket(8080);                         System.out.println("服务器已启动并监听8080端口");                         while(true) {                                 System.out.println();                                 System.out.println("服务器正在等待连接...");                                 //阻塞1：等待连接时阻塞                                 Socket socket = serverSocket.accept();                                 System.out.println("服务器已接收到连接请求...");                                 System.out.println();                                 System.out.println("服务器正在等待数据...");                                 //阻塞2：等待数据时阻塞                                 socket.getInputStream().read(buffer);                                 System.out.println("服务器已经接收到数据");                                 System.out.println();                                 String content = new String(buffer);                                 System.out.println("接收到的数据:"+ content);                         }                 } catch(IOException e) {                         // TODO Auto-generated catch block                         e.printStackTrace();                 }         } } 

我们需要再老调重谈的一点是，如果单线程服务器在等待数据时阻塞，那么第二个连接请求到来时，服务器是无法响应的。如果是多线程服务器，那么又会有为大量空闲请求产生新线程从而造成线程占用系统资源，线程浪费的情况。

那么我们的问题就转移到，如何让单线程服务器在等待客户端数据到来时，依旧可以接收新的客户端连接请求。

#### 7.2 模拟NIO解决方案

如果要解决上文中提到的单线程服务器接收数据时阻塞，而无法接收新请求的问题，那么其实可以让服务器在等待数据时不进入阻塞状态，问题不就迎刃而解了吗？

***【第一种解决方案（等待连接时和等待数据时不阻塞）】：\***

>  public class Server {         public static void main(String[] args) throws InterruptedException {                 ByteBuffer byteBuffer = ByteBuffer.allocate(1024);                 try{                         //Java为非阻塞设置的类                         ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();                         serverSocketChannel.bind(newInetSocketAddress(8080));                         //设置为非阻塞                         serverSocketChannel.configureBlocking(false);                         while(true) {                                 SocketChannel socketChannel = serverSocketChannel.accept();                                 if(socketChannel==null) {                                         //表示没人连接                                         System.out.println("正在等待客户端请求连接...");                                         Thread.sleep(5000);                                 }else{                                         System.out.println("当前接收到客户端请求连接...");                                 }                                 if(socketChannel!=null) {                     //设置为非阻塞                                         socketChannel.configureBlocking(false);                                         byteBuffer.flip();//切换模式  写-->读                                         int effective = socketChannel.read(byteBuffer);                                         if(effective!=0) {                                                 String content = Charset.forName("utf-8").decode(byteBuffer).toString();                                                 System.out.println(content);                                         }else{                                                 System.out.println("当前未收到客户端消息");                                         }                                 }                         }                 } catch(IOException e) {                         // TODO Auto-generated catch block                         e.printStackTrace();                 }         } } 

**运行结果：**

![img](https://ask.qcloudimg.com/http-save/6708446/16lubjnni1.png?imageView2/2/w/1620)

**代码解析：**

不难看出，在这种解决方案下，虽然在接收客户端消息时不会阻塞，但是又开始重新接收服务器请求，用户根本来不及输入消息，服务器就转向接收别的客户端请求了，换言之，服务器弄丢了当前客户端的请求。

***【解决方案二（缓存Socket，轮询数据是否准备好）】：\***

>  public class Server {         public static void main(String[] args) throws InterruptedException {                 ByteBuffer byteBuffer = ByteBuffer.allocate(1024);                   List<SocketChannel> socketList = newArrayList<SocketChannel>();                 try{                         //Java为非阻塞设置的类                         ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();                         serverSocketChannel.bind(newInetSocketAddress(8080));                         //设置为非阻塞                         serverSocketChannel.configureBlocking(false);                         while(true) {                                 SocketChannel socketChannel = serverSocketChannel.accept();                                 if(socketChannel==null) {                                         //表示没人连接                                         System.out.println("正在等待客户端请求连接...");                                         Thread.sleep(5000);                                 }else{                                         System.out.println("当前接收到客户端请求连接...");                                         socketList.add(socketChannel);                                 }                                 for(SocketChannel socket:socketList) {                                         socket.configureBlocking(false);                                         int effective = socket.read(byteBuffer);                                         if(effective!=0) {                                                 byteBuffer.flip();//切换模式  写-->读                                                 String content = Charset.forName("UTF-8").decode(byteBuffer).toString();                                                 System.out.println("接收到消息:"+content);                                                 byteBuffer.clear();                                         }else{                                                 System.out.println("当前未收到客户端消息");                                         }                                 }                         }                 } catch(IOException e) {                         // TODO Auto-generated catch block                         e.printStackTrace();                 }         } } 

**运行结果：**

![img](https://ask.qcloudimg.com/http-save/6708446/34k2xo5qkz.png?imageView2/2/w/1620)

![img](https://ask.qcloudimg.com/http-save/6708446/ojjikot0q1.png?imageView2/2/w/1620)

**代码解析：**

在解决方案一中，我们采用了非阻塞方式，但是发现一旦非阻塞，等待客户端发送消息时就不会再阻塞了，而是直接重新去获取新客户端的连接请求，这就会造成客户端连接丢失。

而在解决方案二中，我们将连接存储在一个list集合中，每次等待客户端消息时都去轮询，看看消息是否准备好，如果准备好则直接打印消息。

可以看到，从头到尾我们一直没有开启第二个线程，而是一直采用单线程来处理多个客户端的连接，这样的一个模式可以很完美地解决BIO在单线程模式下无法处理多客户端请求的问题，并且解决了非阻塞状态下连接丢失的问题。

#### 7.3 存在的问题（解决方案二）

从刚才的运行结果中其实可以看出，消息没有丢失，程序也没有阻塞。

但是，在接收消息的方式上可能有些许不妥，我们采用了一个轮询的方式来接收消息，每次都轮询所有的连接，看消息是否准备好，测试用例中只是三个连接，所以看不出什么问题来，但是我们假设有1000万连接，甚至更多，采用这种轮询的方式效率是极低的。

另外，1000万连接中，我们可能只会有100万会有消息，剩下的900万并不会发送任何消息，那么这些连接程序依旧要每次都去轮询，这显然是不合适的。

#### 7.4 真实NIO中如何解决

在真实NIO中，并不会在Java层上来进行一个轮询，而是将轮询的这个步骤交给我们的操作系统来进行，他将轮询的那部分代码改为操作系统级别的系统调用（select函数，在linux环境中为epoll），在操作系统级别上调用select函数，主动地去感知有数据的socket。

# 8、关于使用select/epoll和直接在应用层做轮询的区别

我们在之前实现了一个使用Java做多个客户端连接轮询的逻辑，但是在真正的NIO源码中其实并不是这么实现的，NIO使用了操作系统底层的轮询系统调用 select/epoll(windows:select,linux:epoll)，那么为什么不直接实现而要去调用系统来做轮询呢？

#### 8.1 select底层逻辑

![img](https://ask.qcloudimg.com/http-save/6708446/zqzqo22fue.jpeg?imageView2/2/w/1620)

假设有A、B、C、D、E五个连接同时连接服务器，那么根据我们上文中的设计，程序将会遍历这五个连接，轮询每个连接，获取各自数据准备情况，那么和我们自己写的程序有什么区别呢？

**首先：**我们写的Java程序其本质在轮询每个Socket的时候也需要去调用系统函数，那么轮询一次调用一次，会造成不必要的上下文切换开销。

**而：**Select会将五个请求从用户态空间全量复制一份到内核态空间，在内核态空间来判断每个请求是否准备好数据，完全避免频繁的上下文切换。所以效率是比我们直接在应用层写轮询要高的。

**如果：**select没有查询到到有数据的请求，那么将会一直阻塞（是的，select是一个阻塞函数）。如果有一个或者多个请求已经准备好数据了，那么select将会先将有数据的文件描述符置位，然后select返回。返回后通过遍历查看哪个请求有数据。

**select的缺点：**

>  1）底层存储依赖bitmap，处理的请求是有上限的，为1024； 2）文件描述符是会置位的，所以如果当被置位的文件描述符需要重新使用时，是需要重新赋空值的； 3）fd（文件描述符）从用户态拷贝到内核态仍然有一笔开销； 4）select返回后还要再次遍历，来获知是哪一个请求有数据。 

#### 8.2 poll函数底层逻辑

poll的工作原理和select很像，先来看一段poll内部使用的一个结构体。

>  struct pollfd{     int fd;     short events;     short revents; } 

poll同样会将所有的请求拷贝到内核态，和select一样，poll同样是一个阻塞函数，当一个或多个请求有数据的时候，也同样会进行置位，但是它置位的是结构体pollfd中的events或者revents置位，而不是对fd本身进行置位，所以在下一次使用的时候不需要再进行重新赋空值的操作。poll内部存储不依赖bitmap，而是使用pollfd数组的这样一个数据结构，数组的大小肯定是大于1024的。解决了select 1、2两点的缺点。

#### 8.3 epoll函数底层逻辑

epoll是最新的一种多路IO复用的函数。这里只说说它的特点。

epoll和上述两个函数最大的不同是，它的fd是共享在用户态和内核态之间的，所以可以不必进行从用户态到内核态的一个拷贝，这样可以节约系统资源。

另外，在select和poll中，如果某个请求的数据已经准备好，它们会将所有的请求都返回，供程序去遍历查看哪个请求存在数据，但是epoll只会返回存在数据的请求，这是因为epoll在发现某个请求存在数据时，首先会进行一个重排操作，将所有有数据的fd放到最前面的位置，然后返回（返回值为存在数据请求的个数N），那么我们的上层程序就可以不必将所有请求都轮询，而是直接遍历epoll返回的前N个请求，这些请求都是有数据的请求。



# 9、Java中BIO和NIO的概念总结

通常一些文章都是在开头放上概念，但是我这次选择将概念放在结尾，因为通过上面的实操，相信大家对Java中BIO和NIO都有了自己的一些理解，这时候再来看[概念](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fguanghuichenshao%2Farticle%2Fdetails%2F79375967)应该会更好理解一些了。

**先来个例子理解一下概念，以银行取款为例：**

1）同步 ： 自己亲自出马持银行卡到银行取钱（使用同步IO时，Java自己处理IO读写）；

3）异步 ： 委托一小弟拿银行卡到银行取钱，然后给你（使用异步IO时，Java将IO读写委托给OS处理，需要将数据缓冲区地址和大小传给OS(银行卡和密码)，OS需要支持异步IO操作API）；

3）阻塞 ： ATM排队取款，你只能等待（使用阻塞IO时，Java调用会一直阻塞到读写完成才返回）；

4）非阻塞 ： 柜台取款，取个号，然后坐在椅子上做其它事，等号广播会通知你办理，没到号你就不能去，你可以不断问大堂经理排到了没有，大堂经理如果说还没到你就不能去（使用非阻塞IO时，如果不能读写Java调用会马上返回，当IO事件分发器会通知可读写时再继续进行读写，不断循环直到读写完成）。

**Java对BIO、NIO的支持：**

1）Java BIO (blocking I/O)：同步并阻塞，服务器实现模式为一个连接一个线程，即客户端有连接请求时服务器端就需要启动一个线程进行处理，如果这个连接不做任何事情会造成不必要的线程开销，当然可以通过线程池机制改善；

2）Java NIO (non-blocking I/O)： 同步非阻塞，服务器实现模式为一个请求一个线程，即客户端发送的连接请求都会注册到多路复用器上，多路复用器轮询到连接有I/O请求时才启动一个线程进行处理。

**BIO、NIO适用场景分析:**

1）BIO方式： 适用于连接数目比较小且固定的架构，这种方式对服务器资源要求比较高，并发局限于应用中，JDK1.4以前的唯一选择，但程序直观简单易理解；

2）NIO方式： 适用于连接数目多且连接比较短（轻操作）的架构，比如聊天服务器，并发局限于应用中，编程比较复杂，JDK1.4开始支持。

# 10、本文小结

本文介绍了一些关于JavaBIO和NIO从自己实操的角度上的一些理解，我个人认为这样去理解BIO和NIO会比光看概念会有更深的理解，也希望各位同学可以自己去敲一遍，通过程序的运行结果得出自己对JavaBIO和NIO的理解。
