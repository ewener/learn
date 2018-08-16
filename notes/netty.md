



## 一。linux网络IO模型

> 前期知识储备：linux内核将所有外部设备都看作一个文件夹操作，对一个文件的读写操作会调用内核提供的系统命令，返回一个file descriptor(fd 文件描述符)，描述符就是一个数字，它指向内核的一个结构体（文件路径，数据区等一些属性）。

### 1. IO阻塞模型

​            ![](https://github.com/XwDai/learn/raw/master/notes/image/%E9%98%BB%E5%A1%9E%E6%A8%A1%E5%9E%8B.png)

### 2. 非阻塞IO模型

![](https://github.com/XwDai/learn/raw/master/notes/image/%E9%9D%9E%E9%98%BB%E5%A1%9E%E6%A8%A1%E5%9E%8B.png)

### 3. IO复用模型

> 1. select/poll，阻塞在select操作上，select/poll可以侦测多个fd(文件描述)是否处于就绪状态，它是顺序扫描fd是否就绪，而且支持fd的数量有限（默认1024，由FD_SETSIZE设置，可以选择修改这个宏然后重新编译内核，不过这会带来网络效率的下降），因此它的使用受到了制约。
> 2. epoll，它基于事件驱动的方式代替顺序扫描，因此性能更高，当 fd就绪时，立即回调函数rollback。

![](https://github.com/XwDai/learn/raw/master/notes/image/io%E5%A4%8D%E7%94%A8%E6%A8%A1%E5%9E%8B.png)

### 4. 信号驱动IO模型

![](https://github.com/XwDai/learn/raw/master/notes/image/%E4%BF%A1%E5%8F%B7%E9%A9%B1%E5%8A%A8%E6%A8%A1%E5%9E%8B.png)

### 5. 异步IO

告知内核启动某个操作，并让内核在整个操作完成之后（包括将数据从内核复制到用户自己的缓冲区）通知我们。

> 这种模型和信号驱动模型区别：
>
> 1. 信号驱动：由内核通知我们何时开始一个IO操作。
> 2. 异步IO：由内核通知我们IO操作何时已完成。

![](https://github.com/XwDai/learn/raw/master/notes/image/%E5%BC%82%E6%AD%A5%E6%A8%A1%E5%9E%8B.png)

## 二。IO多路复用技术

> 原理：通过把多个IO的阻塞复用到同一个select的阻塞上，从而使得系统在单线程的情况下可以同时处理多个客户端请求。

> 优点：系统开销小，降低维护工作量，节省资源。  

> 运用场景：
>
> 1. 服务器同时处理多个处于监听状态或多个连接状态的套接字。
> 2. 服务器需要处理多个协议的网络套接字。

> 目前支持IO多路复用的系统调用：select、pselect、poll、epoll。

>  epoll相对于select的重大改进：
>
> 1. 支持一个进程打开的socket描述符不受限（仅受限于操作系统的最大文件句柄数）
> 2. IO效率不会随着FD数目的增加而线性下降
> 3. 使用mmap加速内核与用户空间的消息传递（内核和用户空间mmap同一块内存）
> 4. epoll的API更简单

## 三。NIO

### 1. 缓冲区Buffer

> 在NIO库中，所有数据的读写都是用缓冲区处理。它实质上是一个数组，通常是一个字节数组（ByteBuffer），也可以用其他种类的数组。缓冲区不仅仅是一个数组，它也提供了对数据的结构化访问以及维护读写位置等信息。

缓冲区种类：

- ByteBuffer：字节缓冲区
- CharBuffer：字符缓冲区
- ShortBuffer：短整型缓冲区
- IntBuffer：整型缓冲区
- LongBuffer：长整型缓冲区
- FloatBuffer：浮点型缓冲区
- DoubleBuffer：双精度浮点型缓冲区

​                              

![](https://github.com/XwDai/learn/raw/master/notes/image/buffer%E7%BB%A7%E6%89%BF%E5%85%B3%E7%B3%BB%E5%9B%BE.png)

### 2. 通道Channel

> 可以通过它读取和写入数据，它与流不同之处在于通道是双向的，流只能在一个方向移动（一个流必须是InputStream或OutputStream的子类），而且通道可以用于读、写、或同时读写。因为它是全双工的，所以比流更好的映射底层操作系统API。特别是在UNIX网络编程模型中，底层操作系统的通道都是全双工的，同时支持读写操作。

![](https://github.com/XwDai/learn/raw/master/notes/image/Channel%E7%BB%A7%E6%89%BF%E5%85%B3%E7%B3%BB%E5%9B%BE.png)

### 3. 多路复用器Selector

​          Selector会不断的轮询注册在其上的Channel，如果某个Channel上面有新的TCP连接接入、读和写事件，这个Channel就处于就绪状态，会被Selector轮询出来，然后通过SelectionKey可以获得就绪Channel的集合，进行后续的IO操作。一个多路复用器Selector可以同时轮询多个Channel，由于JDK使用epoll代替了select实现，所以他没有最大连接句柄1024/2048的限制。

## 四。NIO服务端序列图

​       ![](https://github.com/XwDai/learn/raw/master/notes/image/NIO%E6%9C%8D%E5%8A%A1%E7%AB%AF%E9%80%9A%E4%BF%A1%E5%BA%8F%E5%88%97%E5%9B%BE.png)



## 五。NIO客户端序列图

​      

![](https://github.com/XwDai/learn/raw/master/notes/image/NIO%E5%AE%A2%E6%88%B7%E7%AB%AF%E5%BA%8F%E5%88%97%E5%9B%BE.png)

## 六。API

​      flip：将缓冲区当前的limit设置为position，position设置为0。

​      hasRemaining：判断消息是否发送完成。

## 七。AIO

## 八。IO模型 对比

![](https://github.com/XwDai/learn/raw/master/notes/image/io%E6%A8%A1%E5%9E%8B%E5%AF%B9%E6%AF%94.png)

## 九。netty

### 1. tcp粘包、拆包问题。

> 产生原因：

1. 应用程序write写入的字节大小大于套接口发送缓冲区大小。
2. 进行MSS大小的TCP分段。
3. 以太网帧的playload大于MTU进行ip分片。

![](https://github.com/XwDai/learn/raw/master/notes/image/tcp%E7%B2%98%E5%8C%85%E6%8B%86%E5%8C%85%E9%97%AE%E9%A2%98%E5%8E%9F%E5%9B%A0.png)

> 解决策略：

1. 消息定长，不够补空格。

2. 在包尾加回车换行符进行分割，例如，ftp协议。

3. 将消息分为消息头和消息体，消息头里包含消息总长度或消息体长度的字段，通常设计思路为消息头的第一个字段使用int32来表示消息的总长度。

4. 更复杂的应用层协议。

5. netty解决方法：

   * 利用LineBasedFrameDecoder、StringDecoder。LineBasedFrameDecoder原理：遍历ByteBuf的可读字节，判断是否有“\n”或“\r\n”，有就以此为结束位置，从可读索引到 结束位置区间就组成了一行。支持携带结束符或者不携带两种方式，同时支持配置单行的最大长度，如果读取到最大长度仍然没有发现换行符，就抛出异常，同时忽略掉之前读到的异常码流。

   * 分隔符和定长解码器。

     DelimiterBasedFrameDecoder：以分隔符作为结束标志

     FixedLengthFrameDecoder：定长消息

### 2.编解码

 #### 1. java序列化

   java序列化缺点：

   1. 无法跨语言
   2. 序列化的码流太大
   3. 序列化性能低

 #### 2. 业界主流编解码框架

   * protobuf

   特点：

   1. 数据结构以.proto文件描述，通过工具生成对应数据结构的pojo和protobuf相关属性和方法。

   ​       优点：1.文本化数据结构与语言和平台无关。2.通过标识字段顺序，可以实现协议的向前兼容。

   ​                   3.自动代码生成。4.相比代码，结构化文档更容易维护。

   1. 结构化数据存储格式（xml，json等）。
   2. 高效编解码性能。
   3. 跨平台，跨语言，扩展性好。
   4. 官方支持java、c++、python三中语言。
   5. 使用二进制编码，空间和性能都有优势。

   为何不使用xml：

   1. 优点：可读性和扩展性好。适合描述数据结构。
   2. 缺点：解析时间开销和空间开销非常大。这一点不适合做高性能的通信协议。

   

   * thrift

   组成：

   1. 语言系统及IDL编译器。
   2. TProtocol：rpc协议层。也就是编解码框架。可以作为类库单独使用。
   3. TTransport：rpc传输层。
   4. TProcessor：协议层和用户提供服务实现之间的纽带。
   5. TServer：聚合上面的对象。

   特点：

   1. IDL描述接口和数据结构。支持java8种基本类型、map、set、list。
   2. 支持可选和必选定义。
   3. 可以定义数据结构中的字段顺序，所以支持协议向前兼容。

   三种典型的编解码：

   1. 二进制编解码。
   2. 压缩二进制编解码。
   3. 优化的可选字段压缩编解码。

   因为有压缩功能，所以性能比protobuf略好。

   

   * JBoss Marshalling

   特点：

   1. 修正了jdk自带的序列化包很多问题，保持跟jdk序列化接口兼容，增加了可调的参数和附加特性，可通过工厂类配置。
   2. 可插拔的类解析器。更便捷的类加载制定策略，通过一个接口即可实现定制。
   3. 可插拔的对象替换技术。不需要继承。
   4. 可插拔的预定义类缓存表。可减小序列化字节数组长度，提升常用类型对象的序列化性能。
   5. 无需实现序列化接口。
   6. 通过缓存技术提升性能。

   #### 3. netty使用java序列化

   * ObjectEncoder\ObjectDecoder
   * client：

   ```java
   import io.netty.bootstrap.Bootstrap;
   import io.netty.channel.ChannelFuture;
   import io.netty.channel.ChannelInitializer;
   import io.netty.channel.ChannelOption;
   import io.netty.channel.EventLoopGroup;
   import io.netty.channel.nio.NioEventLoopGroup;
   import io.netty.channel.socket.SocketChannel;
   import io.netty.channel.socket.nio.NioSocketChannel;
   import io.netty.handler.codec.serialization.ClassResolvers;
   import io.netty.handler.codec.serialization.ObjectDecoder;
   import io.netty.handler.codec.serialization.ObjectEncoder;
   
   /**
    * @author lilinfeng
    * @date 2014年2月14日
    * @version 1.0
    */
   public class SubReqClient {
   
       public void connect(int port, String host) throws Exception {
   	// 配置客户端NIO线程组
   	EventLoopGroup group = new NioEventLoopGroup();
   	try {
   	    Bootstrap b = new Bootstrap();
   	    b.group(group).channel(NioSocketChannel.class)
   		    .option(ChannelOption.TCP_NODELAY, true)
   		    .handler(new ChannelInitializer<SocketChannel>() {
   			@Override
   			public void initChannel(SocketChannel ch)
   				throws Exception {
   			    ch.pipeline().addLast(
   				    new ObjectDecoder(1024, ClassResolvers
   					    .cacheDisabled(this.getClass()
   						    .getClassLoader())));
   			    ch.pipeline().addLast(new ObjectEncoder());
   			    ch.pipeline().addLast(new SubReqClientHandler());
   			}
   		    });
   
   	    // 发起异步连接操作
   	    ChannelFuture f = b.connect(host, port).sync();
   
   	    // 当代客户端链路关闭
   	    f.channel().closeFuture().sync();
   	} finally {
   	    // 优雅退出，释放NIO线程组
   	    group.shutdownGracefully();
   	}
       }
   
       /**
        * @param args
        * @throws Exception
        */
       public static void main(String[] args) throws Exception {
   	int port = 8080;
   	if (args != null && args.length > 0) {
   	    try {
   		port = Integer.valueOf(args[0]);
   	    } catch (NumberFormatException e) {
   		// 采用默认值
   	    }
   	}
   	new SubReqClient().connect(port, "127.0.0.1");
       }
   ```

   * client handler

     ```java
     import io.netty.channel.ChannelHandlerAdapter;
     import io.netty.channel.ChannelHandlerContext;
     
     import com.phei.netty.codec.pojo.SubscribeReq;
     
     /**
      * @author lilinfeng
      * @date 2014年2月14日
      * @version 1.0
      */
     public class SubReqClientHandler extends ChannelHandlerAdapter {
     
         /**
          * Creates a client-side handler.
          */
         public SubReqClientHandler() {
         }
     
         @Override
         public void channelActive(ChannelHandlerContext ctx) {
     	for (int i = 0; i < 10; i++) {
     	    ctx.write(subReq(i));
     	}
     	ctx.flush();
         }
     
         private SubscribeReq subReq(int i) {
     	SubscribeReq req = new SubscribeReq();
     	req.setAddress("南京市雨花台区软件大道101号华为基地");
     	req.setPhoneNumber("138xxxxxxxxx");
     	req.setProductName("Netty 最佳实践和原理分析");
     	req.setSubReqID(i);
     	req.setUserName("Lilinfeng");
     	return req;
         }
     
         @Override
         public void channelRead(ChannelHandlerContext ctx, Object msg)
     	    throws Exception {
     	System.out.println("Receive server response : [" + msg + "]");
         }
     
         @Override
         public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
     	ctx.flush();
         }
     
         @Override
         public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
     	cause.printStackTrace();
     	ctx.close();
         }
     ```

     

   * server：

     ```java
     import io.netty.bootstrap.ServerBootstrap;
     import io.netty.channel.ChannelFuture;
     import io.netty.channel.ChannelInitializer;
     import io.netty.channel.ChannelOption;
     import io.netty.channel.EventLoopGroup;
     import io.netty.channel.nio.NioEventLoopGroup;
     import io.netty.channel.socket.SocketChannel;
     import io.netty.channel.socket.nio.NioServerSocketChannel;
     import io.netty.handler.codec.serialization.ClassResolvers;
     import io.netty.handler.codec.serialization.ObjectDecoder;
     import io.netty.handler.codec.serialization.ObjectEncoder;
     import io.netty.handler.logging.LogLevel;
     import io.netty.handler.logging.LoggingHandler;
     
     /**
      * @author lilinfeng
      * @date 2014年2月14日
      * @version 1.0
      */
     public class SubReqServer {
         public void bind(int port) throws Exception {
     	// 配置服务端的NIO线程组
     	EventLoopGroup bossGroup = new NioEventLoopGroup();
     	EventLoopGroup workerGroup = new NioEventLoopGroup();
     	try {
     	    ServerBootstrap b = new ServerBootstrap();
     	    b.group(bossGroup, workerGroup)
     		    .channel(NioServerSocketChannel.class)
     		    .option(ChannelOption.SO_BACKLOG, 100)
     		    .handler(new LoggingHandler(LogLevel.INFO))
     		    .childHandler(new ChannelInitializer<SocketChannel>() {
     			@Override
     			public void initChannel(SocketChannel ch) {
     			    ch.pipeline()
     				    .addLast(
     					    new ObjectDecoder(
     						    1024 * 1024,
     						    ClassResolvers
     							    .weakCachingConcurrentResolver(this
     								    .getClass()
     								    .getClassLoader())));
     			    ch.pipeline().addLast(new ObjectEncoder());
     			    ch.pipeline().addLast(new SubReqServerHandler());
     			}
     		    });
     
     	    // 绑定端口，同步等待成功
     	    ChannelFuture f = b.bind(port).sync();
     
     	    // 等待服务端监听端口关闭
     	    f.channel().closeFuture().sync();
     	} finally {
     	    // 优雅退出，释放线程池资源
     	    bossGroup.shutdownGracefully();
     	    workerGroup.shutdownGracefully();
     	}
         }
     
         public static void main(String[] args) throws Exception {
     	int port = 8080;
     	if (args != null && args.length > 0) {
     	    try {
     		port = Integer.valueOf(args[0]);
     	    } catch (NumberFormatException e) {
     		// 采用默认值
     	    }
     	}
     	new SubReqServer().bind(port);
         }
     ```

   * server handler：

     ```java
     import io.netty.channel.ChannelHandler.Sharable;
     import io.netty.channel.ChannelHandlerAdapter;
     import io.netty.channel.ChannelHandlerContext;
     
     import com.phei.netty.codec.pojo.SubscribeReq;
     import com.phei.netty.codec.pojo.SubscribeResp;
     
     /**
      * @author lilinfeng
      * @date 2014年2月14日
      * @version 1.0
      */
     @Sharable
     public class SubReqServerHandler extends ChannelHandlerAdapter {
     
         @Override
         public void channelRead(ChannelHandlerContext ctx, Object msg)
     	    throws Exception {
     	SubscribeReq req = (SubscribeReq) msg;
     	if ("Lilinfeng".equalsIgnoreCase(req.getUserName())) {
     	    System.out.println("Service accept client subscrib req : ["
     		    + req.toString() + "]");
     	    ctx.writeAndFlush(resp(req.getSubReqID()));
     	}
         }
     
         private SubscribeResp resp(int subReqID) {
     	SubscribeResp resp = new SubscribeResp();
     	resp.setSubReqID(subReqID);
     	resp.setRespCode(0);
     	resp.setDesc("Netty book order succeed, 3 days later, sent to the designated address");
     	return resp;
         }
     
         @Override
         public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
     	cause.printStackTrace();
     	ctx.close();// 发生异常，关闭链路
         }
     ```
#### 4.protobuf编解码

   * ProtobufDecoder/ProtobufEncoder

     > 注意事项：ProtobufDecoder只负责解码，它不支持读半包，在之前，一定要有能够处理读半包的解码器，没有的话，程序会报错。有三种方式可以选择：
     >
     > 1. 使用netty的ProtobufVarint32FrameDecoder，可以处理半包消息。
     > 2. 继承netty提供的半包解码器LengthFieldBasedFrameDecoder。
     > 3. 继承ByteToMessageDecoder，自己处理半包消息。

####      5.jboss marshalling编解码（只在jboss中有用到）



### 3.http协议开发应用

> http缺点：
>
> 1. 半双工协议。数据可以在客户端和服务的两个方向传输，但不能同时传输，同一时刻只有只有一个方向上的数据传输。
> 2. http消息冗长。有消息头、消息体、换行符等，相比二进制通信协议，冗长繁琐。
> 3. 针对服务器推送的黑客攻击。例如长时间轮询。

* 代码示例：文件服务器，可浏览目录，下载文件

```java
import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.http.HttpObjectAggregator;
import io.netty.handler.codec.http.HttpRequestDecoder;
import io.netty.handler.codec.http.HttpResponseEncoder;
import io.netty.handler.stream.ChunkedWriteHandler;

/**
 * @author lilinfeng
 * @date 2014年2月14日
 * @version 1.0
 */
public class HttpFileServer {

    private static final String DEFAULT_URL = "/test";

    private static final String IP = "localhost";

    public void run(final int port, final String url) throws Exception {
	EventLoopGroup bossGroup = new NioEventLoopGroup();
	EventLoopGroup workerGroup = new NioEventLoopGroup();
	try {
	    ServerBootstrap b = new ServerBootstrap();
	    b.group(bossGroup, workerGroup)
		    .channel(NioServerSocketChannel.class)
		    .childHandler(new ChannelInitializer<SocketChannel>() {
			@Override
			protected void initChannel(SocketChannel ch)
				throws Exception {
			    ch.pipeline().addLast("http-decoder",
				    new HttpRequestDecoder()); // 请求消息解码器
			    ch.pipeline().addLast("http-aggregator",
				    new HttpObjectAggregator(65536));// 目的是将多个消息转换为单一的request或者response对象
			    ch.pipeline().addLast("http-encoder",
				    new HttpResponseEncoder());//响应解码器
			    ch.pipeline().addLast("http-chunked",
				    new ChunkedWriteHandler());//目的是支持异步大文件传输（）
			    ch.pipeline().addLast("fileServerHandler",
				    new HttpFileServerHandler(url));// 业务逻辑
			}
		    });
	    ChannelFuture future = b.bind(IP, port).sync();
	    System.out.println("HTTP文件目录服务器启动，网址是 : " + "http://"+IP+":"
		    + port + url);
	    future.channel().closeFuture().sync();
	} finally {
	    bossGroup.shutdownGracefully();
	    workerGroup.shutdownGracefully();
	}
    }

    public static void main(String[] args) throws Exception {
	int port = 8081;
	if (args.length > 0) {
	    try {
		port = Integer.parseInt(args[0]);
	    } catch (NumberFormatException e) {
		e.printStackTrace();
	    }
	}
	String url = DEFAULT_URL;
	if (args.length > 1)
	    url = args[1];
	new HttpFileServer().run(port, url);
    }
```
> handler
```java

import static io.netty.handler.codec.http.HttpHeaders.isKeepAlive;
import static io.netty.handler.codec.http.HttpHeaders.setContentLength;
import static io.netty.handler.codec.http.HttpHeaders.Names.CONNECTION;
import static io.netty.handler.codec.http.HttpHeaders.Names.CONTENT_TYPE;
import static io.netty.handler.codec.http.HttpHeaders.Names.LOCATION;
import static io.netty.handler.codec.http.HttpMethod.GET;
import static io.netty.handler.codec.http.HttpResponseStatus.BAD_REQUEST;
import static io.netty.handler.codec.http.HttpResponseStatus.FORBIDDEN;
import static io.netty.handler.codec.http.HttpResponseStatus.FOUND;
import static io.netty.handler.codec.http.HttpResponseStatus.INTERNAL_SERVER_ERROR;
import static io.netty.handler.codec.http.HttpResponseStatus.METHOD_NOT_ALLOWED;
import static io.netty.handler.codec.http.HttpResponseStatus.NOT_FOUND;
import static io.netty.handler.codec.http.HttpResponseStatus.OK;
import static io.netty.handler.codec.http.HttpVersion.HTTP_1_1;
import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelFutureListener;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelProgressiveFuture;
import io.netty.channel.ChannelProgressiveFutureListener;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.handler.codec.http.DefaultFullHttpResponse;
import io.netty.handler.codec.http.DefaultHttpResponse;
import io.netty.handler.codec.http.FullHttpRequest;
import io.netty.handler.codec.http.FullHttpResponse;
import io.netty.handler.codec.http.HttpHeaders;
import io.netty.handler.codec.http.HttpResponse;
import io.netty.handler.codec.http.HttpResponseStatus;
import io.netty.handler.codec.http.LastHttpContent;
import io.netty.handler.stream.ChunkedFile;
import io.netty.util.CharsetUtil;

import java.io.File;
import java.io.FileNotFoundException;
import java.io.RandomAccessFile;
import java.io.UnsupportedEncodingException;
import java.net.URLDecoder;
import java.util.regex.Pattern;

import javax.activation.MimetypesFileTypeMap;

/**
 * @author lilinfeng
 * @date 2014年2月14日
 * @version 1.0
 */
public class HttpFileServerHandler extends
	SimpleChannelInboundHandler<FullHttpRequest> {
    private final String url;

    public HttpFileServerHandler(String url) {
	this.url = url;
    }

    @Override
    public void messageReceived(ChannelHandlerContext ctx,
	    FullHttpRequest request) throws Exception {
	if (!request.getDecoderResult().isSuccess()) {
	    sendError(ctx, BAD_REQUEST);
	    return;
	}
	if (request.getMethod() != GET) {
	    sendError(ctx, METHOD_NOT_ALLOWED);
	    return;
	}
	final String uri = request.getUri();
	final String path = sanitizeUri(uri);
	if (path == null) {
	    sendError(ctx, FORBIDDEN);
	    return;
	}
	File file = new File(path);
	if (file.isHidden() || !file.exists()) {
	    sendError(ctx, NOT_FOUND);
	    return;
	}
	if (file.isDirectory()) {
	    if (uri.endsWith("/")) {
		sendListing(ctx, file);
	    } else {
		sendRedirect(ctx, uri + '/');
	    }
	    return;
	}
	if (!file.isFile()) {
	    sendError(ctx, FORBIDDEN);
	    return;
	}
	RandomAccessFile randomAccessFile = null;
	try {
	    randomAccessFile = new RandomAccessFile(file, "r");// 以只读的方式打开文件
	} catch (FileNotFoundException fnfe) {
	    sendError(ctx, NOT_FOUND);
	    return;
	}
	long fileLength = randomAccessFile.length();
	HttpResponse response = new DefaultHttpResponse(HTTP_1_1, OK);
	setContentLength(response, fileLength);
	setContentTypeHeader(response, file);
	if (isKeepAlive(request)) {
	    response.headers().set(CONNECTION, HttpHeaders.Values.KEEP_ALIVE);
	}
	ctx.write(response);
	ChannelFuture sendFileFuture;
	sendFileFuture = ctx.write(new ChunkedFile(randomAccessFile, 0,
		fileLength, 8192), ctx.newProgressivePromise());
	sendFileFuture.addListener(new ChannelProgressiveFutureListener() {
	    @Override
	    public void operationProgressed(ChannelProgressiveFuture future,
		    long progress, long total) {
		if (total < 0) { // total unknown
		    System.err.println("Transfer progress: " + progress);
		} else {
		    System.err.println("Transfer progress: " + progress + " / "
			    + total);
		}
	    }

	    @Override
	    public void operationComplete(ChannelProgressiveFuture future)
		    throws Exception {
		System.out.println("Transfer complete.");
	    }
	});
	ChannelFuture lastContentFuture = ctx
		.writeAndFlush(LastHttpContent.EMPTY_LAST_CONTENT);
	if (!isKeepAlive(request)) {
	    lastContentFuture.addListener(ChannelFutureListener.CLOSE);
	}
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)
	    throws Exception {
	cause.printStackTrace();
	if (ctx.channel().isActive()) {
	    sendError(ctx, INTERNAL_SERVER_ERROR);
	}
    }

    private static final Pattern INSECURE_URI = Pattern.compile(".*[<>&\"].*");

    private String sanitizeUri(String uri) {
	try {
	    uri = URLDecoder.decode(uri, "UTF-8");
	} catch (UnsupportedEncodingException e) {
	    try {
		uri = URLDecoder.decode(uri, "ISO-8859-1");
	    } catch (UnsupportedEncodingException e1) {
		throw new Error();
	    }
	}
	if (!uri.startsWith(url)) {
	    return null;
	}
	if (!uri.startsWith("/")) {
	    return null;
	}
	uri = uri.replace('/', File.separatorChar);
	if (uri.contains(File.separator + '.')
		|| uri.contains('.' + File.separator) || uri.startsWith(".")
		|| uri.endsWith(".") || INSECURE_URI.matcher(uri).matches()) {
	    return null;
	}
	return System.getProperty("user.dir") + File.separator + uri;
    }

    private static final Pattern ALLOWED_FILE_NAME = Pattern
	    .compile("[A-Za-z0-9][-_A-Za-z0-9\\.]*");

    private static void sendListing(ChannelHandlerContext ctx, File dir) {
	FullHttpResponse response = new DefaultFullHttpResponse(HTTP_1_1, OK);
	response.headers().set(CONTENT_TYPE, "text/html; charset=UTF-8");
	StringBuilder buf = new StringBuilder();
	String dirPath = dir.getPath();
	buf.append("<!DOCTYPE html>\r\n");
	buf.append("<html><head><title>");
	buf.append(dirPath);
	buf.append(" 目录：");
	buf.append("</title></head><body>\r\n");
	buf.append("<h3>");
	buf.append(dirPath).append(" 目录：");
	buf.append("</h3>\r\n");
	buf.append("<ul>");
	buf.append("<li>链接：<a href=\"../\">..</a></li>\r\n");
	for (File f : dir.listFiles()) {
	    if (f.isHidden() || !f.canRead()) {
		continue;
	    }
	    String name = f.getName();
	    if (!ALLOWED_FILE_NAME.matcher(name).matches()) {
		continue;
	    }
	    buf.append("<li>链接：<a href=\"");
	    buf.append(name);
	    buf.append("\">");
	    buf.append(name);
	    buf.append("</a></li>\r\n");
	}
	buf.append("</ul></body></html>\r\n");
	ByteBuf buffer = Unpooled.copiedBuffer(buf, CharsetUtil.UTF_8);
	response.content().writeBytes(buffer);
	buffer.release();
	ctx.writeAndFlush(response).addListener(ChannelFutureListener.CLOSE);
    }

    private static void sendRedirect(ChannelHandlerContext ctx, String newUri) {
	FullHttpResponse response = new DefaultFullHttpResponse(HTTP_1_1, FOUND);
	response.headers().set(LOCATION, newUri);
	ctx.writeAndFlush(response).addListener(ChannelFutureListener.CLOSE);
    }

    private static void sendError(ChannelHandlerContext ctx,
	    HttpResponseStatus status) {
	FullHttpResponse response = new DefaultFullHttpResponse(HTTP_1_1,
		status, Unpooled.copiedBuffer("Failure: " + status.toString()
			+ "\r\n", CharsetUtil.UTF_8));
	response.headers().set(CONTENT_TYPE, "text/plain; charset=UTF-8");
	ctx.writeAndFlush(response).addListener(ChannelFutureListener.CLOSE);
    }

    private static void setContentTypeHeader(HttpResponse response, File file) {
	MimetypesFileTypeMap mimeTypesMap = new MimetypesFileTypeMap();
	response.headers().set(CONTENT_TYPE,
		mimeTypesMap.getContentType(file.getPath()));
    }
}

```



### 4.Websocket协议开发

> 特点：
>
> 1. 单一tcp连接，全双工通信。
> 2. 对代理、防火墙、路由器透明。
> 3. 无头部信息、cookie和身份验证。
> 4. 无安全开销。
> 5. 通过ping/pong帧保持链路激活。
> 6. 服务器主动传递消息给客户端，不需要客户端轮询。

* 客户端发送服务端的http请求：

![](https://github.com/XwDai/learn/raw/master/notes/image/websocket%E5%AE%A2%E6%88%B7%E7%AB%AF%E8%AF%B7%E6%B1%82.png)

“1”表名这是一个申请协议升级的http请求，服务端生成应答信息返给客户端，客户端与服务端的websocket连接就建立了。并且这个连接会持续到某一方关闭。

* 服务端返回：

  ![](https://github.com/XwDai/learn/raw/master/notes/image/websocket%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%BA%94%E7%AD%94.png)

请求中的“2”是随机的，服务端会用这些数据构造出一个SHA-1的信息摘要：

1. 把“2”加上一个魔幻字符串“258EAFA5-E914-47DA-95CA-C5AB0DC85B11”。
2. 使用SHA-1加密。然后BASE-64编码。
3. 将值放到"3"中。

最后返回给客户端。

* websocket生命周期

  ![](https://github.com/XwDai/learn/raw/master/notes/image/websocket%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F.png)

  websocket连接关闭

  正常首先由服务端关闭，异常情况，客户端一个合理的周期后没有收到服务器的TCP Close，客户端可以发起TCP Close。

  websocket握手关闭消息带有一个状态码和一个可选的关闭的原因。

* netty websocket协议开发

  > server端

  ```java
  import io.netty.bootstrap.ServerBootstrap;
  import io.netty.channel.Channel;
  import io.netty.channel.ChannelInitializer;
  import io.netty.channel.ChannelPipeline;
  import io.netty.channel.EventLoopGroup;
  import io.netty.channel.nio.NioEventLoopGroup;
  import io.netty.channel.socket.SocketChannel;
  import io.netty.channel.socket.nio.NioServerSocketChannel;
  import io.netty.handler.codec.http.HttpObjectAggregator;
  import io.netty.handler.codec.http.HttpServerCodec;
  import io.netty.handler.stream.ChunkedWriteHandler;
  
  /**
   * @author lilinfeng
   * @date 2014年2月14日
   * @version 1.0
   */
  public class WebSocketServer {
      public void run(int port) throws Exception {
  	EventLoopGroup bossGroup = new NioEventLoopGroup();
  	EventLoopGroup workerGroup = new NioEventLoopGroup();
  	try {
  	    ServerBootstrap b = new ServerBootstrap();
  	    b.group(bossGroup, workerGroup)
  		    .channel(NioServerSocketChannel.class)
  		    .childHandler(new ChannelInitializer<SocketChannel>() {
  
  			@Override
  			protected void initChannel(SocketChannel ch)
  				throws Exception {
  			    ChannelPipeline pipeline = ch.pipeline();
  			    pipeline.addLast("http-codec",
  				    new HttpServerCodec());//将请求和应答编码或解码为http消息
  			    pipeline.addLast("aggregator",
  				    new HttpObjectAggregator(65536));//将http消息的多个部分组合成一条完整的http消息
  			    ch.pipeline().addLast("http-chunked",
  				    new ChunkedWriteHandler());//向客户端发送h5文件，主要用来支持浏览器和服务端进行websocket通信。
  			    pipeline.addLast("handler",
  				    new WebSocketServerHandler());
  			}
  		    });
  
  	    Channel ch = b.bind(port).sync().channel();
  	    System.out.println("Web socket server started at port " + port
  		    + '.');
  	    System.out
  		    .println("Open your browser and navigate to http://localhost:"
  			    + port + '/');
  
  	    ch.closeFuture().sync();
  	} finally {
  	    bossGroup.shutdownGracefully();
  	    workerGroup.shutdownGracefully();
  	}
      }
  
      public static void main(String[] args) throws Exception {
  	int port = 8080;
  	if (args.length > 0) {
  	    try {
  		port = Integer.parseInt(args[0]);
  	    } catch (NumberFormatException e) {
  		e.printStackTrace();
  	    }
  	}
  	new WebSocketServer().run(port);
      }
  ```

  ```java
  import static io.netty.handler.codec.http.HttpHeaders.isKeepAlive;
  import static io.netty.handler.codec.http.HttpHeaders.setContentLength;
  import static io.netty.handler.codec.http.HttpResponseStatus.BAD_REQUEST;
  import static io.netty.handler.codec.http.HttpVersion.HTTP_1_1;
  import io.netty.buffer.ByteBuf;
  import io.netty.buffer.Unpooled;
  import io.netty.channel.ChannelFuture;
  import io.netty.channel.ChannelFutureListener;
  import io.netty.channel.ChannelHandlerContext;
  import io.netty.channel.SimpleChannelInboundHandler;
  import io.netty.handler.codec.http.DefaultFullHttpResponse;
  import io.netty.handler.codec.http.FullHttpRequest;
  import io.netty.handler.codec.http.FullHttpResponse;
  import io.netty.handler.codec.http.websocketx.CloseWebSocketFrame;
  import io.netty.handler.codec.http.websocketx.PingWebSocketFrame;
  import io.netty.handler.codec.http.websocketx.PongWebSocketFrame;
  import io.netty.handler.codec.http.websocketx.TextWebSocketFrame;
  import io.netty.handler.codec.http.websocketx.WebSocketFrame;
  import io.netty.handler.codec.http.websocketx.WebSocketServerHandshaker;
  import io.netty.handler.codec.http.websocketx.WebSocketServerHandshakerFactory;
  import io.netty.util.CharsetUtil;
  
  import java.util.logging.Level;
  import java.util.logging.Logger;
  
  /**
   * @author lilinfeng
   * @date 2014年2月14日
   * @version 1.0
   */
  public class WebSocketServerHandler extends SimpleChannelInboundHandler<Object> {
      private static final Logger logger = Logger
  	    .getLogger(WebSocketServerHandler.class.getName());
  
      private WebSocketServerHandshaker handshaker;
  
      @Override
      public void messageReceived(ChannelHandlerContext ctx, Object msg)
  	    throws Exception {
  	// 传统的HTTP接入
  	if (msg instanceof FullHttpRequest) {
  	    handleHttpRequest(ctx, (FullHttpRequest) msg);
  	}
  	// WebSocket接入
  	else if (msg instanceof WebSocketFrame) {
  	    handleWebSocketFrame(ctx, (WebSocketFrame) msg);
  	}
      }
  
      @Override
      public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
  	ctx.flush();
      }
  
      private void handleHttpRequest(ChannelHandlerContext ctx,
  	    FullHttpRequest req) throws Exception {
  
  	// 如果HTTP解码失败，返回HHTP异常
  	if (!req.getDecoderResult().isSuccess()
  		|| (!"websocket".equals(req.headers().get("Upgrade")))) {
  	    sendHttpResponse(ctx, req, new DefaultFullHttpResponse(HTTP_1_1,
  		    BAD_REQUEST));
  	    return;
  	}
  
  	// 构造握手响应返回，本机测试
  	WebSocketServerHandshakerFactory wsFactory = new WebSocketServerHandshakerFactory(
  		"ws://localhost:8080/websocket", null, false);
  	handshaker = wsFactory.newHandshaker(req);
  	if (handshaker == null) {
  	    WebSocketServerHandshakerFactory
  		    .sendUnsupportedWebSocketVersionResponse(ctx.channel());
  	} else {
  	    handshaker.handshake(ctx.channel(), req);
  	}
      }
  
      private void handleWebSocketFrame(ChannelHandlerContext ctx,
  	    WebSocketFrame frame) {
  
  	// 判断是否是关闭链路的指令
  	if (frame instanceof CloseWebSocketFrame) {
  	    handshaker.close(ctx.channel(),
  		    (CloseWebSocketFrame) frame.retain());
  	    return;
  	}
  	// 判断是否是Ping消息
  	if (frame instanceof PingWebSocketFrame) {
  	    ctx.channel().write(
  		    new PongWebSocketFrame(frame.content().retain()));
  	    return;
  	}
  	// 本例程仅支持文本消息，不支持二进制消息
  	if (!(frame instanceof TextWebSocketFrame)) {
  	    throw new UnsupportedOperationException(String.format(
  		    "%s frame types not supported", frame.getClass().getName()));
  	}
  
  	// 返回应答消息
  	String request = ((TextWebSocketFrame) frame).text();
  	if (logger.isLoggable(Level.FINE)) {
  	    logger.fine(String.format("%s received %s", ctx.channel(), request));
  	}
  	ctx.channel().write(
  		new TextWebSocketFrame(request
  			+ " , 欢迎使用Netty WebSocket服务，现在时刻："
  			+ new java.util.Date().toString()));
      }
  
      private static void sendHttpResponse(ChannelHandlerContext ctx,
  	    FullHttpRequest req, FullHttpResponse res) {
  	// 返回应答给客户端
  	if (res.getStatus().code() != 200) {
  	    ByteBuf buf = Unpooled.copiedBuffer(res.getStatus().toString(),
  		    CharsetUtil.UTF_8);
  	    res.content().writeBytes(buf);
  	    buf.release();
  	    setContentLength(res, res.content().readableBytes());
  	}
  
  	// 如果是非Keep-Alive，关闭连接
  	ChannelFuture f = ctx.channel().writeAndFlush(res);
  	if (!isKeepAlive(req) || res.getStatus().code() != 200) {
  	    f.addListener(ChannelFutureListener.CLOSE);
  	}
      }
  
      @Override
      public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)
  	    throws Exception {
  	cause.printStackTrace();
  	ctx.close();
      }
  }
  ```

  

### 5.UDP(用户数据报协议)协议开发

> 特点：
>
> 1. 直接用ip协议进行UDP数据报的传输，在传输数据前，发送方和接收方相互互换信息使双方同步。
> 2. 面向无连接、不可靠（不对数据报分组、组装、校验、排序，报文发送者不知道报文是否被对方正确接收，接收方接收到数据报不会发送确认信号）的数据报投递服务。
> 3. 资源消耗小、处理速度快（比tcp快）、简单。
>
> 应用：音频、视频等可靠性要求不高的数据传输一般会使用UDP。

1. udp协议

   ![](https://github.com/XwDai/learn/raw/master/notes/image/udp%E5%8D%8F%E8%AE%AE.jpg)

* 注意：伪首部是一个临时结构，既不向上也不向下传递，仅仅为了保证可以校验套接字的正确性。

> client：

```java
import io.netty.bootstrap.Bootstrap;
import io.netty.buffer.Unpooled;
import io.netty.channel.Channel;
import io.netty.channel.ChannelOption;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.DatagramPacket;
import io.netty.channel.socket.nio.NioDatagramChannel;
import io.netty.util.CharsetUtil;

import java.net.InetSocketAddress;

/**
 * @author lilinfeng
 * @date 2014年2月14日
 * @version 1.0
 */
public class ChineseProverbClient {

    public void run(int port) throws Exception {
	EventLoopGroup group = new NioEventLoopGroup();
	try {
	    Bootstrap b = new Bootstrap();
	    b.group(group).channel(NioDatagramChannel.class)
		    .option(ChannelOption.SO_BROADCAST, true)
		    .handler(new ChineseProverbClientHandler());
	    Channel ch = b.bind(0).sync().channel();
	    // 向网段内的所有机器广播UDP消息
	    ch.writeAndFlush(
		    new DatagramPacket(Unpooled.copiedBuffer("谚语字典查询?",
			    CharsetUtil.UTF_8), new InetSocketAddress(
			    "255.255.255.255", port))).sync();
	    if (!ch.closeFuture().await(15000)) {
		System.out.println("查询超时!");
	    }
	} finally {
	    group.shutdownGracefully();
	}
    }

    public static void main(String[] args) throws Exception {
	int port = 8080;
	if (args.length > 0) {
	    try {
		port = Integer.parseInt(args[0]);
	    } catch (NumberFormatException e) {
		e.printStackTrace();
	    }
	}
	new ChineseProverbClient().run(port);
    }
}
```

> client handler：

```java
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.channel.socket.DatagramPacket;
import io.netty.util.CharsetUtil;

/**
 * @author lilinfeng
 * @date 2014年2月14日
 * @version 1.0
 */
public class ChineseProverbClientHandler extends
	SimpleChannelInboundHandler<DatagramPacket> {

    @Override
    public void messageReceived(ChannelHandlerContext ctx, DatagramPacket msg)
	    throws Exception {
	String response = msg.content().toString(CharsetUtil.UTF_8);
	if (response.startsWith("谚语查询结果: ")) {
	    System.out.println(response);
	    ctx.close();
	}
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)
	    throws Exception {
	cause.printStackTrace();
	ctx.close();
    }
}
```

> server：

```java
import io.netty.bootstrap.Bootstrap;
import io.netty.channel.ChannelOption;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.nio.NioDatagramChannel;

/**
 * @author lilinfeng
 * @date 2014年2月14日
 * @version 1.0
 */
public class ChineseProverbServer {
    public void run(int port) throws Exception {
	EventLoopGroup group = new NioEventLoopGroup();
	try {
	    Bootstrap b = new Bootstrap();
	    b.group(group).channel(NioDatagramChannel.class)
		    .option(ChannelOption.SO_BROADCAST, true)
		    .handler(new ChineseProverbServerHandler());
	    b.bind(port).sync().channel().closeFuture().await();
	} finally {
	    group.shutdownGracefully();
	}
    }

    public static void main(String[] args) throws Exception {
	int port = 8080;
	if (args.length > 0) {
	    try {
		port = Integer.parseInt(args[0]);
	    } catch (NumberFormatException e) {
		e.printStackTrace();
	    }
	}
	new ChineseProverbServer().run(port);
    }
}
```

> server handler：

```java
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.channel.socket.DatagramPacket;
import io.netty.util.CharsetUtil;
import io.netty.util.internal.ThreadLocalRandom;

/**
 * @author lilinfeng
 * @date 2014年2月14日
 * @version 1.0
 */
public class ChineseProverbServerHandler extends
	SimpleChannelInboundHandler<DatagramPacket> {

    // 谚语列表
    private static final String[] DICTIONARY = { "只要功夫深，铁棒磨成针。",
	    "旧时王谢堂前燕，飞入寻常百姓家。", "洛阳亲友如相问，一片冰心在玉壶。", "一寸光阴一寸金，寸金难买寸光阴。",
	    "老骥伏枥，志在千里。烈士暮年，壮心不已!" };

    private String nextQuote() {
	int quoteId = ThreadLocalRandom.current().nextInt(DICTIONARY.length);
	return DICTIONARY[quoteId];
    }

    @Override
    public void messageReceived(ChannelHandlerContext ctx, DatagramPacket packet)
	    throws Exception {
	String req = packet.content().toString(CharsetUtil.UTF_8);
	System.out.println(req);
	if ("谚语字典查询?".equals(req)) {
	    ctx.writeAndFlush(new DatagramPacket(Unpooled.copiedBuffer(
		    "谚语查询结果: " + nextQuote(), CharsetUtil.UTF_8), packet
		    .sender()));
	}
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)
	    throws Exception {
	ctx.close();
	cause.printStackTrace();
    }
}

```

* 说明：由于udp无连接，因此不需要为连接（ChannelPipeline）设置handler。

### 6.文件传输

> server：

```java
import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.LineBasedFrameDecoder;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.codec.string.StringEncoder;
import io.netty.util.CharsetUtil;

/**
 * @author lilinfeng
 * @date 2014年2月14日
 * @version 1.0
 */
public class FileServer {

    public void run(int port) throws Exception {
	EventLoopGroup bossGroup = new NioEventLoopGroup();
	EventLoopGroup workerGroup = new NioEventLoopGroup();
	try {
	    ServerBootstrap b = new ServerBootstrap();
	    b.group(bossGroup, workerGroup)
		    .channel(NioServerSocketChannel.class)
		    .option(ChannelOption.SO_BACKLOG, 100)
		    .childHandler(new ChannelInitializer<SocketChannel>() {
			/*
			 * (non-Javadoc)
			 * 
			 * @see
			 * io.netty.channel.ChannelInitializer#initChannel(io
			 * .netty.channel.Channel)
			 */
			public void initChannel(SocketChannel ch)
				throws Exception {
                //如果大文件传输，将文件全部映射到内存，很可能导致内存溢出，
                //可以使用netty的ChunkedWriteHandler来解决大文件或者码流传输过程中可能发生的内存溢出                   的问题
			    ch.pipeline().addLast(
				    new StringEncoder(CharsetUtil.UTF_8),
				    new LineBasedFrameDecoder(1024),
				    new StringDecoder(CharsetUtil.UTF_8),
				    new FileServerHandler());
			}
		    });
	    ChannelFuture f = b.bind(port).sync();
	    System.out.println("Start file server at port : " + port);
	    f.channel().closeFuture().sync();
	} finally {
	    // 优雅停机
	    bossGroup.shutdownGracefully();
	    workerGroup.shutdownGracefully();
	}
    }

    public static void main(String[] args) throws Exception {
	int port = 8080;
	if (args.length > 0) {
	    try {
		port = Integer.parseInt(args[0]);
	    } catch (NumberFormatException e) {
		e.printStackTrace();
	    }
	}
	new FileServer().run(port);
    }
}

```

> handler：

```java
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.DefaultFileRegion;
import io.netty.channel.FileRegion;
import io.netty.channel.SimpleChannelInboundHandler;

import java.io.File;
import java.io.RandomAccessFile;

/**
 * @author Administrator
 * @date 2014年3月9日
 * @version 1.0
 */
public class FileServerHandler extends SimpleChannelInboundHandler<String> {

    private static final String CR = System.getProperty("line.separator");

    /*
     * (non-Javadoc)
     * 
     * @see
     * io.netty.channel.SimpleChannelInboundHandler#messageReceived(io.netty
     * .channel.ChannelHandlerContext, java.lang.Object)
     */
    public void messageReceived(ChannelHandlerContext ctx, String msg)
	    throws Exception {
	File file = new File(msg);
	if (file.exists()) {
	    if (!file.isFile()) {
		ctx.writeAndFlush("Not a file : " + file + CR);
		return;
	    }
	    ctx.write(file + " " + file.length() + CR);
	    RandomAccessFile randomAccessFile = new RandomAccessFile(msg, "r");
        //fileRegion支持零拷贝。不支持零拷贝的操作系统，可能表现更差或者失败。
        //FileChannel至少有4个已知的bug，如果要使用零拷贝，需要升级jdk到1.6.0_18或者更高的版本。
	    FileRegion region = new DefaultFileRegion(
		    randomAccessFile.getChannel(), 0, randomAccessFile.length());
	    ctx.write(region);
	    ctx.writeAndFlush(CR);
	    randomAccessFile.close();
	} else {
	    ctx.writeAndFlush("File not found: " + file + CR);
	}
    }

    /*
     * (non-Javadoc)
     * 
     * @see
     * io.netty.channel.ChannelHandlerAdapter#exceptionCaught(io.netty.channel
     * .ChannelHandlerContext, java.lang.Throwable)
     */
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)
	    throws Exception {
	cause.printStackTrace();
	ctx.close();
    }
}

```

### 7.私有协议栈开发



## 十.源码分析

### 1.Bytebuf和相关辅助类

> jdk的ByteBuffer主要缺点：
>
> 1. 长度固定，一旦分配完成，容量不能动态扩展和收缩，当pojo对象大于bytebuf容量时，会发生索引越界异常。
> 2. 只有一个标识位置的指针position，读写需要手工flip()和rewind()等，必需小心处理，否则容易处理出错。
> 3. api功能有限，一些高级和实用特性不支持。

#### bytebuf设计：

1. 增加readIndex和writeIndex，读写不用调整指针。

2. discardReadBytes：重用已读的缓冲区，可以节省容量，防止容量不足导致的扩张。但是，他会有字节数组的内存复制，频繁调用将会导致性能下降。因此，调用前需要权衡下利弊。

3. nioBuffer：转换标准byteBuffer，共用缓冲区内容引用，返回的byteBuffer无法感知byteBuf的动态扩展。

4. 支持顺序读取(read/write)和随机读写(set/get)。两者都会对索引和长度校验，不同的是set不支持动态扩展缓冲区。set/get读写缓冲区不会修改读写索引。

5. byteBuf主要继承关系  ![](https://github.com/XwDai/learn/raw/master/notes/image/byteBuf%E4%B8%BB%E8%A6%81%E7%B1%BB%E7%BB%A7%E6%89%BF%E5%85%B3%E7%B3%BB.jpg)

   * 从内存分配的角度：

   1. 堆内存（HeapByteBuf）字节缓冲区：内存分配回收速度快，可以被jvm自动回收，缺点是如果进行socket io读写，需要额外做一次内存复制，将堆内存对应的缓冲区复制到内核channel中，性能有所下降。
   2. 直接内存字节缓冲区（DirectByteBuf）：非堆内存，在堆外内存分配，分配和回收速度会慢一些，但是将它写入或者从socket channel中读取时，会少一次内存复制，速度比堆内存块。

   > 最佳实践：
   >
   > 在io通信线程的读写缓冲区使用DirectByteBuf，后端业务消息的编解码模块使用HeapByteBuf，这样组合可以达到性能最优。

   * 从内存回收的角度：

   1. 普通的ByteBuf。
   2. 基于对象池的ByteBuf。维护了一个内存池，可以重用ByteBuf对象，提升内存使用效率，降低由于高负载导致的频繁GC。测试表明，在使用内存池后的Netty在高负载、大并发的冲击下内存和GC更平稳。

6. AbstractByteBuf

   1. 定义了一些公共属性，读写操作的公共行为方法。
   2. 全局 leakDetector：用于检测对象是否泄漏。
   3. 没有定义缓冲区实现。因为无法确定子类是基于直接内存还是对内存。

7. AbstractReferenceCountedByteBuf

   1. 引用计数器refCnt，最小值为1，当被释放和被申请次数相等时，就调用回收方法回收当前的ByteBuf。

8. UnpooledHeapByteBuf ：每次IO读写都会创建一个新的，频繁大内存分配和回收队性能会造成一定影响，但是相比堆外内存的申请和释放，成本还是低些。相比PooledHeapByteBuf，实现简单，不容易出现内存管理方面的问题，在性能满足的情况下，推荐使用UnpooledHeapByteBuf。

   1. 聚合了一个ByteBufAllocator分配内存
   2. 定义了一个byte数组作为缓冲区。直接使用ByteBuffer也是可以的，这里使用数组是为了提升性能和便捷的位操作。
   3. 定义了一个ByteBuffer变量用于实现ByteBuf到ByteBuffer的转换。

9. PooledByteBuf：

   1. PoolArena。netty的内存池实现，由多个Chunk组成的大块内存区域。
   2. PoolChunk。组织和管理多个Page的内存分配和释放。Chunk中的page被构建成一个二叉树。
   3. PoolSubpage。

10. 扩容算法：需要的新容量小于阈值4M时，以64为计数倍增（64->128->256字节，这样的扩张方式是可以被接受的），等于时直接使用阈值作为新缓冲区的大小，大于阈值时以步增的方式（防止内存膨胀和浪费），如果大于缓冲区的最大容量（ByteBuf可以设置最大容量）则使用最大容量作为扩容后的缓冲区容量。最后使用System.arrycopy进行内存复制到新缓冲区。

### 2.Channel和Unsafe

### 3.ChannelPipeline和ChannelHandler

### 4.EventLoop和EventLoppGroup

### 5.Future和Promise



## 十一.netty高级特性

### 1.java多线程编程在netty中的应用

### 2.netty架构剖析

### 3.netty行业应用



## 十二.netty参数配置


![](https://github.com/XwDai/learn/raw/master/notes/image/netty参数一.png)

![](https://github.com/XwDai/learn/raw/master/notes/image/netty参数二.png)