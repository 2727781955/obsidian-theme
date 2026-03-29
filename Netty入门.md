## NIO的概念

为什么需要 NIO？（打破“一连接一线程”）
传统的 **BIO (Blocking I/O)** 采用的是同步阻塞模型。每当有一个客户端连接，服务端就必须开启一个专门的线程来处理。
当并发量达到万级甚至更高时，大量的线程会瞬间榨干服务器内存，且频繁的线程上下文切换会导致性能急剧下降。
线程在等待数据读取（read）或写入（write）时会被挂起，白白浪费 CPU 资源。
    

### NIO 的三大核心组件

NIO 引入了“多路复用”的概念，主要由以下三个核心组成：
1. Buffer（缓冲区）：在 NIO 中，所有数据的读写不是直接操作流（Stream），而是通过 Buffer。它本质上是一块可以读写的内存块，既可以从中读数据，也可以向其中写数据。
    
2. Channel（通道）：Channel类似于 BIO 中的 Stream，但它是双向的。数据可以从 Channel 读到 Buffer 中，也可以从 Buffer 写入 Channel。TCP中分为ServerSocketChannel 和 SocketChannel。ServeSocketChannel可以理解为主干道，负责建立连接；而SocketChannel就是在主干道上的分干道，负责建立连接后的消息读写。主干道只有一个而副干道可以有多个。
    
3. Selector（选择器）；Selector 允许单线程处理多个 Channel。Channel 会向 Selector 注册自己感兴趣的事件（如：连接就绪、读就绪）。Selector 会不断轮询这些 Channel，只有当事件真正发生时，才会交给线程去处理。仅需极少的线程就能管理成千上万个连接，极大地提升了并发处理能力。
    



## NIO 的编程逻辑流程


### 服务端
我们用代码来演示，先是服务端：
```java
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.*;
import java.util.Iterator;

public class NioServer {
    public static void main(String[] args) throws IOException {
        // 1. 创建并配置 ServerSocketChannel
        ServerSocketChannel serverChannel = ServerSocketChannel.open();
        serverChannel.bind(new InetSocketAddress(8080));
        serverChannel.configureBlocking(false); // 必须是非阻塞才能注册到 Selector

        // 2. 创建 Selector
        Selector selector = Selector.open();

        // 3. 将服务端通道注册到 Selector，监听“连接请求”
        serverChannel.register(selector, SelectionKey.OP_ACCEPT);
        System.out.println("NIO 服务端启动，端口：8080");

        while (true) {
            // 4. 等待事件发生（阻塞）
            selector.select();

            // 5. 获取所有就绪事件并迭代
            Iterator<SelectionKey> it = selector.selectedKeys().iterator();
            while (it.hasNext()) {
                SelectionKey key = it.next();
                it.remove(); // 必须手动移除，否则下次循环会处理废弃的 key

                // 6. 分发处理
                if (key.isAcceptable()) {
                    // 接受新客户端连接
                    SocketChannel clientChannel = serverChannel.accept();
                    clientChannel.configureBlocking(false);
                    // 注册读事件，等待客户端发消息
                    clientChannel.register(selector, SelectionKey.OP_READ);
                    System.out.println("新客户端接入：" + clientChannel.getRemoteAddress());
                } 
                else if (key.isReadable()) {
                    // 读取数据
                    handleRead(key);
                }
            }
        }
    }

    private static void handleRead(SelectionKey key) throws IOException {
        SocketChannel channel = (SocketChannel) key.channel();
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        try {
            int len = channel.read(buffer);
            if (len > 0) {
                System.out.println("收到消息: " + new String(buffer.array(), 0, len));
                // 回应客户端
                channel.write(ByteBuffer.wrap("Server Echo".getBytes()));
            } else if (len == -1) {
                System.out.println("客户端断开");
                channel.close();
            }
        } catch (IOException e) {
            key.cancel();
            channel.close();
        }
    }
}               
```

具体的流程如下：首先，我们需要注册一个ServeSocketChannel，并将其设置为非阻塞模式，接着我们创建一个选择器，连接到ServeSocketChannel上，监听`OP_ACCEPT`，也就是接受连接的事件。
我们设置一个死循环，调用 `selector.select()`，该方法会阻塞直到至少有一个事件发生。

如果有事件发生了，`selector.selectedKeys()` 会返回所有发生了事件的 Channel 句柄。紧接着，遍历 `selectedKeys`，根据事件类型（Accept, Read, Write）执行相应的业务逻辑。

### 客户端

客户端的操作相对简单，需要通过buffer接收和发放数据
```java
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SocketChannel;
import java.util.Scanner;

public class NioClient {
    public static void main(String[] args) throws IOException {
        // 1. 打开通道并连接服务端
        SocketChannel socketChannel = SocketChannel.open();
        socketChannel.configureBlocking(false); // 设置为非阻塞
        
        // 发起连接
        if (!socketChannel.connect(new InetSocketAddress("127.0.0.1", 8080))) {
            // 非阻塞模式下，connect 会立即返回，需要轮询是否连接完成
            while (!socketChannel.finishConnect()) {
                System.out.println("正在尝试连接...");
            }
        }
        System.out.println("连接成功！请输入要发送的消息：");

        // 2. 交互逻辑
        Scanner scanner = new Scanner(System.in);
        while (scanner.hasNextLine()) {
            String msg = scanner.nextLine();
            
            // 写入数据到 Buffer 并发送
            ByteBuffer buffer = ByteBuffer.wrap(msg.getBytes());
            socketChannel.write(buffer);

            // 读取服务端回传（简单处理）
            ByteBuffer readBuffer = ByteBuffer.allocate(1024);
            int len = socketChannel.read(readBuffer);
            if (len > 0) {
                System.out.println("服务端回应: " + new String(readBuffer.array(), 0, len));
            }
        }
    }
}
```

流程相对服务端更加简单：首先注册一个ServeSocket并设置为非阻塞模式，紧接着调用`connect`方法进行连接；并对连接结果进行判断，如果返回false，就一直轮询尝试建立连接。连接成功后，客户端就可以把数据放在buffer里面和服务端进行交互了。

### Netty 对原生 NIO的优化

虽然原生 NIO 很强大，但直接使用时还是会有许多问题：
- **复杂度高：** 需要手动处理半包、粘包问题。
- **断连重连：** 需要处理复杂的链路异常和网络闪断。
- **Epoll 空轮询 Bug 修复**：JDK 的 NIO 在 Linux 环境下可能会触发 `epoll` 的空轮询，导致 `Selector` 在没有事件发生时也被唤醒，从而使 CPU 占用率瞬间飙升至 100%。
    


下面开始真正接触Netty。





## Netty基本概念
相比NIO里面的Selector监听事件，Netty中引入了EventLoop的概念，这是一个事件循环，里面封装了NIO中的Selector。它会监听Channel上的事件，一个EventLoop对应着一个线程，它会不断检查是否有新的事件发生。

Netty将事件监听线程分为两组：
1. Boss线程组：用于监听服务端的accept事件，负责客户端和服务端之间建立连接；它就像一个大管家，只负责接待来客，而细致的活就交给worker线程组。
2. Worker线程组：用于监听客户端的write、read事件，负责读写数据；它就像小伙计，在Boss线程组确认建立连接后，具体的事物就交给它来处理。


结合前面的EventLoop，我们将这样的线程组分别称为：bossEventLoopGroup和workerEventLoop Group，而boss线程和worker线程称为bossEventLoop和workerEventLoop。

一般情况下我们只有一个服务端，所以bossEventLoopGroup里只有一个EventLoop监听accept事件；而客户端有多个，所以workerEventLoopGroup内有多个EventLoop来监听读写事件。

在Netty服务启动后，会先创建一个NioServeSocketChannel,它负责承载连接的消息。当建立连接成功后，boss线程组会为这个新链接创建一个NioSocketChannel，然后在worker线程组内挑选一个具体的EventLoop，以后这个NioSocketChannel上的任何读写操作都由这个EventLoop负责。一个 EventLoop 负责管理多个 Channel，但一个 Channel 只能绑定到一个固定的 EventLoop。

![](Pasted%20image%2020260329003307.png)

建立读写连接通道后，数据在服务端和客户端之间大概是这样的一个流程：
首先，客户端会先生成数据，经过一系列处理后将数据进行编码，再将编码好的数据发送给服务端；服务端将客户端传递过来的数据先解码，然后进行处理、最后将数据反馈给客户端，这一系列的处理都是在Channel上进行的。

而这一系列对消息的操作，我们统称为消息处理器，即Handler。相同类型的Handler之间相互进行传递；根据出站和入站的消息处理器类型，我们又把Handler 分为入站处理器：ChannelInboundHandler和出站处理器：ChannelOutBoundHandler。

我们将这些消息处理器维护在一个双向链表里，这个结构叫做PipeLine，它是一个消息处理流水线，当消息从Channel进行处理时，就会按照流水线上消息处理器的顺序对消息进行处理。当然每一个channel都拥有自己的pipeline。


Netty内部架构示意图：
![](Pasted%20image%2020260328035852.png)
上面提到的所有组件，在Netty中，我们使用bootStrap来进行组装。可以看到，Netty的组件还是很多的：ServeSocketChannel、SocketChannel、eventLoopGroup、Handler、Pipeline、BootStrap，现在请你回想一下，将这些概念串联起来，我们再继续。


## 构建Netty
现在我们来使用代码进行演示：

首先，我们引入依赖
```xml
<dependencies>
    <dependency>
        <groupId>io.netty</groupId>
        <artifactId>netty-all</artifactId>
        <version>4.1.100.Final</version> 
    </dependency>
</dependencies>
```

### 服务端
接着我们写服务端的bootStrap

```Java
import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.*;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.codec.string.StringEncoder;
import io.netty.handler.logging.LogLevel;
import io.netty.handler.logging.LoggingHandler;

public class NettyServer {
    public static void main(String[] args) throws InterruptedException {
        // 1. 创建两个线程组
        // BossGroup: 负责监听 ServerSocketChannel 的 Accept 事件 (1个线程就够了)
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        // WorkerGroup: 负责处理所有 SocketChannel 的 Read/Write 事件 (默认 CPU*2)
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        try {
            // 2. 服务端启动辅助对象
            ServerBootstrap b = new ServerBootstrap();
            
            b.group(bossGroup, workerGroup)
             // 3. 指定使用的 NIO 传输通道类型
             .channel(NioServerSocketChannel.class)
             
             // 4. 设置针对 ServerSocketChannel 的处理器 (比如日志记录)
             .handler(new LoggingHandler(LogLevel.INFO))
             
             // 5. 设置针对新产生的 SocketChannel 的初始化逻辑 (核心点)
             .childHandler(new ChannelInitializer<SocketChannel>() {
                 @Override
                 protected void initChannel(SocketChannel ch) {
                     // 这里的 ch 是新接生的“孩子”，每个孩子都有独立的 Pipeline
                     ChannelPipeline pipeline = ch.pipeline();
                     
                     // 依次添加 Handler (入站按顺序，出站按逆序)
                     pipeline.addLast(new StringDecoder()); // 入站：ByteBuf -> String
                     pipeline.addLast(new StringEncoder()); // 出站：String -> ByteBuf
                     
                     // 添加自定义的业务处理器
                     pipeline.addLast(new MyServerHandler());
                 }
             })
             
             // 6. TCP 参数配置
             .option(ChannelOption.SO_BACKLOG, 128)          // 服务端接受连接的队列长度
             .childOption(ChannelOption.SO_KEEPALIVE, true); // 保持长连接

            // 7. 绑定端口并同步等待成功 (启动服务器)
            System.out.println("服务器启动中...");
            ChannelFuture f = b.bind(8888).sync();

            // 8. 等待服务端监听端口关闭
            f.channel().closeFuture().sync();
            
        } finally {
            // 9. 优雅释放资源
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}

```

可以看到，流程是这样的：我们先定义两个NioEventLoopGroup，也就是线程组，分别为boss和worker，boss线程组来监听客户端的连接，然后将读写的监听交给worker线程。

接着，我们实例化一个bootStrap对象，指定它的boss线程组和worker线程组，同时我们也要指定其监听的Channel类型：`NioServeSocketChannel`，然后再配置服务端的Handler，我们在`initHandler`方法中，通过pipeline的`addLast`方法依次配置handler。

配置完Handler后，我们就需要将这个bootStrap绑定在端口上，来监听服务端。

### 客户端
下面是客户端的配置，客户端就更简单了，因为它只管发送消息
```java
import io.netty.bootstrap.Bootstrap;
import io.netty.channel.*;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.codec.string.StringEncoder;

/**
 * Netty 客户端配置示例
 */
public class NettyClient {

    public static void main(String[] args) throws InterruptedException {
        // 1. 客户端只需要一个线程组 (相当于服务端的 WorkerGroup)
        // 用于处理网络 I/O 读写
        EventLoopGroup group = new NioEventLoopGroup();

        try {
            // 2. 客户端启动辅助对象 (注意是 Bootstrap，不是 ServerBootstrap)
            Bootstrap bootstrap = new Bootstrap();

            bootstrap.group(group)
                    // 3. 指定客户端使用的通道类型为 NioSocketChannel
                    .channel(NioSocketChannel.class)
                    // 4. 客户端直接使用 .handler()，因为它只有一个唯一的 SocketChannel
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) {
                            ChannelPipeline pipeline = ch.pipeline();
                            
                            // 添加编解码器 (必须与服务端对应)
                            pipeline.addLast(new StringEncoder()); // 发送时：String -> ByteBuf
                            pipeline.addLast(new StringDecoder()); // 接收时：ByteBuf -> String
                            
                            // 添加客户端业务处理器
                            pipeline.addLast(new MyClientHandler());
                        }
                    });

            // 5. 连接服务器 (IP 和 端口)
            System.out.println("正在连接服务器...");
            ChannelFuture future = bootstrap.connect("127.0.0.1", 8888).sync();

            // 6. 连接成功后，获取 Channel 并发送一条消息
            future.channel().writeAndFlush("Hello, I am Yuanyifan!");

            // 7. 等待客户端通道关闭 (比如在 Handler 中调用了 ctx.close())
            future.channel().closeFuture().sync();

        } finally {
            // 8. 优雅退出，释放 NIO 线程组
            group.shutdownGracefully();
            System.out.println("客户端已关闭");
        }
    }

}
```

流程是一样的：首先，我们定义一个NioEventLoopGroup，这个线程组是workerEventLoopGroup，接着实例化一个bootStrap，指定其线程组和通道类型，紧接着配置客户端的handler，我们在`initHandler`方法中，实例化一个pipeline，然后通过其`addLast`方法配置Handler，最后绑定端口，给服务端发消息。



## Netty细节讲解

### Handler顺序
我们在对handler的编排顺序是有一定要求的，我们要求“进站顺序，出站逆序”的顺序来进行编排。
ChannelPipeLine**本质上是一个双向链表**，它会基于当先事件的类型选择方向：
1. 如果是入站类型，就会按照从头到位的方向（head->tail）
2. 如果是出站类型，就会按照从尾到头的方向（tail->head）

假设我们在pipeline中设置了五个handler
```java
pipeline.addLast("In1", new InboundHandlerA());   // 1
pipeline.addLast("Out1", new OutboundHandlerB()); // 2
pipeline.addLast("Out2", new OutboundHandlerC()); // 3
pipeline.addLast("In2", new InboundHandlerD());   // 4
pipeline.addLast("In3", new InboundHandlerE());   // 5
```
根据事件的不同，它的顺序如下：
入站流程（读取数据时）
数据从 `Head` 节点开始，只寻找 `Inbound` 类型的处理器，跳过所有 Outbound。
执行顺序：**Head -> In1 -> In2 -> In3 -> Tail** （注意：尽管 Out1 和 Out2 插在中间，但在入站时它们会被直接无视）

出站流程（写出数据时）
当你在代码里调用 `ctx.writeAndFlush()` 时，消息是会从当前 Handler 开始向从往 Head 方向寻找出站处理器。

![](Pasted%20image%2020260329003540.png)


### Handler对Channel进行监听
我们可以通过事件回调机制让Handler对Channle的各种状态变化进行感知，主要有两种实现方法，一种是在handler中使用内部匿名类进行监听

```java
pipeline.addLast(new ChannelInboundHandlerAdapter() {
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        // 监听：连接激活
        System.out.println("检测到新连接: " + ctx.channel().id());
        // 必须向后传递，否则后面的 Handler 听不到这个事件
        super.channelActive(ctx); 
    }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
        // 监听：连接断开
        System.out.println("连接已关闭，清理用户 Session...");
        super.channelInactive(ctx);
    }
});
```

或者是通过继承`ChannelInboundHandlerAdapter`来重写各种监听方法
```java
public class MyChannelListener extends ChannelInboundHandlerAdapter {

    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        // 监听到新成员加入会议（连接激活）
        System.out.println("新连接已建立: " + ctx.channel().remoteAddress());
    }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) {
        // 监听到成员退出或网络掉线
        System.out.println("连接已断开，正在清理资源...");
    }

    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        // 特殊监听：配合 IdleStateHandler 监听心跳超时
        if (evt instanceof IdleStateEvent) {
            // 发现客户端很久没说话了，主动踢掉
            ctx.close();
        }
    }
}
```

一下是常见的监听状态的方法：

| 方法名                 | 监听的事件       | 触发时机                                       |
| ------------------- | ----------- | ------------------------------------------ |
| `channelRegistered` | 注册成功        | Channel 注册到 EventLoop 时。                   |
| `channelActive`     | **连接建立/激活** | TCP 三次握手完成，Channel 准备好读写时（最常用，常用于心跳开始、鉴权）。 |
| `channelInactive`   | **连接断开**    | TCP 连接关闭或掉线。常用于清理资源、统计在线人数。                |
| `channelRead`       | 数据到达        | 有新数据从对端发来，进入 Pipeline。                     |
| `exceptionCaught`   | 异常发生        | 读写过程中出现错误（如连接重置）。                          |

### Bytebuf
我们知道,`SocketChannel`是数据读写的通道，而`ByteBuffer`是数据的基本载体，数据的读写都是基于ByteBuffer进行的。在NIO中，必须通过`flip()`方法进行读写模式的切换，读写不能进行分离。

而Netty的`Bytebuf`对此进行了优化，它在内部维护了两个指针：
1. writeIndex：指示下一次写操作的位置
2. readIndex：指示下一次读操作的位置




### 解决 NIO 的 Epoll Bug


>