#1主要成员
#1.1、ChannelPipeline
具体实现类DefaultChannelPipeline
ChannelPipeline维护了一个双向的ChannelHandlerContext链表，每个ChannelHandlerContext依赖一个ChannelHandler，
每个Channel（包括ServerChannel）对应一个ChannelPipeline.
#1.2ChannelHandlerContext
抽象子类AbstractChannelHandlerContext
ChannelHandlerContext负责invoke ChannelHandler，并将事件交给下个ChannelHandlerContext.
特别地HeadContext作为Context链表的头节点，TailContext作为Context链表的尾节点.
#1.3ChannelHandler
ChannelHandler分为ChannelInboundHandler和ChannelOutboundHandler，ChannelInboundHandler处理数据输入，
ChannelOutboundHandler处理数据输出.
#1.4Channel
抽象子类AbstractNioChannel.

#1.5NioEventLoopGroup and NioEventLoop
NioEventLoopGroup相当于一个线程池，NioEventLoop相当于一个线程.
netty为每一个ChannelHandlerContext和Channel分配一个NioEventLoop，NioEventLoop的作用是执行Handler和处理IO事件.
在ChannelHandlerContext初始化时，可以为其指定一个NioEventLoopGroup从而分配了一个NioEventLoop，否则Channel的NioEventLoop将作为
其NioEventLoop; 实际上一个Channel下的所有ChannelHandlerContext拥有同一个NioEventLoop，Channel的NioEventLoop和
ChannelHandlerContext的NioEventLoop可以不同，在这种情况下，当ChannelHandler所在线程正在阻塞时（处理消息），该通道依然可以接收数据
（但不能处理，ChannelHandler所属线程正在阻塞，这时将会创建一个task放进NioEventLoop的队列中）和发送数据.
#1.6Bootstrap(ServerBootstrap)
Bootstrap作为引导程序，bind方法(ServerBootstrap)引导端口绑定，启动服务端；connect方法引导客户端连接服务端.
在ServerBootstrap中可以指定两个NioEventLoopGroup，一个称作bossGroup，另一个称作workerGroup，bossGroup为端口监听分配NioEventLoop，
这就意味着ServerChannel单独地注册在一个Selector上; 当建立连接时，workerGroup为连接分配NioEventLoop，NioEventLoop将channel注册.


#2执行机制
#2.1ServerBootstrap.bind
init/register/bind
1、init. 创建一个ServerChannel，分配NioEventLoop，初始化ChannelPipe和ChannelHandler（主要包括ServerBootstrapAcceptor）等
2、register. 调用NioEventLoop.register注册channel，到此NioEventLoop已经启动
3、bind. 创建绑定端口Task，并交给NioEventLoop.execute

#2.2
NioEventLoop检测channel可读并读取后，将数据对象传递给ChannelPipe.fireChannelRead，进而数据对象在ChannelHandlerContext链表中传递
并被ChannelHandler处理.
事实上，当ServerChannel可accept/可读时，在NioEventLoop中看来是同一接口方法，它们的实现是不同的，向ChannelPipe传递的对象也是不一样的.

当客户端发起连接时，bind.NioEventLoop向ChannelPipe传递的是NioSocketChannel对象，最终被传递到ServerBootstrapAcceptor处理（init和register）

#2.3Bootstrap.connect
该过程和ServerBootstrap.bind相似，不再详说

#2.4

                                                        ChannelPipe
                               -------------------------------------------------------------------
                              |    _____________          _____________         _____________     |   
                              |   |             |        |             |       |             |    |
          OutputDirection<----|---| HeadContext |<------>|   Context   |<----->| TailContext | ---|---->InputDirection(read/accept)
          (write/connect/bind)|   |             |        |(maybe multi)|       |             |    |
                              |    -------------          -------------         -------------     |
                               __________||____________________||____________________||___________
                                         ||                    ||                    ||
                                         ||                    ||                    ||
                                         \/                    \/                    \/
                                    ChannelHandler       ChannelHandler       ChannelHandler