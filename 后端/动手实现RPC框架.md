## RPC概念

RPC - Remote Procedure Call

两个不同服务器上的方法有不同的内存空间，需要通过网络编程的方法来传递参数和结果

RPC用于调用远程机器上的方法，就像调用内部接口一样

#### 具体细节点

**序列化**

TCP基于字节流传输，需要把对象序列化为字节流，使用Kryo、Hessian、Protobuf、json等序列化工具

**网络传输**

TCP是字节流传输，每一次不一定是传输一个完整的包，即接收端无法区分每个数据包的边界

所以一般需要自定义协议，编码和解码

**服务发现/服务注册**

注册中心zk，网飞的eureka，阿里巴巴的nacos

#### **通用流程**

1. Client在注册中心查找要调用的方法，哪个ip+host有提供
2. client对要发送的请求和参数进行序列化
3. client对序列化后的包进一步编码，添加头
4. clilent把编码后的包发送到server（netty/socket，负载均衡）
5. 等待回来的包，解码
6. 得到结果

#### HTTP/RPC/RESTFUL

http和rpc本身没什么关系

HTTP是一个具体协议

RPC可以当做是一种思想，其中包括了许多内容

可以使用HTTP来作为RPC的实现方式

## RPC project 实现

#### 具体流程：

##### client:

1. 构造一个实现了InvocationHandler接口的Client代理对象，构造的时候传入了socketClient/nettyClient和用于查找服务的ServicesConfig
2. 使用RpcClientProxy的getProxy方法获得本地Service接口的代理类(Client本地只有要调用服务的接口，但没有具体的实现类)
3. 代理类调用接口中的方法，可以直接获得结果打印，结束，具体的工作都在代理类的invoke方法里
4. invoke方法入参是被代理对象调用的方法和参数
5. 首先根据要调用的方法名，服务接口名，参数，参数类型，加上UUID构造的一个RequestID构建一个RpcRequest对象。然后根据构造代理对象时是使用socket还是netty来进行不同的传输

   - **使用Socket时 :** 先调用服务发现，返回包装好的IP+Host（服务发现是根据要RpcRequest里要查找的服务名，在ZK指定的路径下面查找子节点，返回一个List `<String>`, 对这个list使用负载均衡（随机选一个String），返回一个包装好的InetSocketAddress）
     然后创一个Socket连上服务器，然后把RpcRequest对象用objectOutputStream发出去，然后用InputStream读取服务端返回作为RpcResponse返回，检查一下RequestID是否一致
   - **使用Netty时 ：** NettyRpcClient构造方法也是bootstrap.group一个eventLoopGroup，流水线也是绑定encoder和decoder，再加上clientHandler。还有服务发现、频道发现和未处理请求池。构造出一个bootstrap，然后每次bootstrap.connect上一个服务端时会把返回的channel给存下来复用
     代理类要发信息时，调用的是方法sendRpcRequest，这个方法里先服务发现，然后根据返回的IP+端口查看是否有对应的channel，没有就调bootstrap连接。获得channel后，把请求ID作为Key，一个completableFuture作为value，存进未完成请求里。根据**5**中的rpcRequest包装一个rpcMessage，channel.writeAndFlush(rpcMessage)。这时候要添加一个listener，future类型来判断是否把消息发完了

##### server with socket:

1. SocketServerMain中，创建服务的实现类，然后在ServiceConfig中登记（ServiceConfig就只是把创建的实现类**对象**赋值给Object）
2. 在SocketRpcServer中注册服务，具体为获取创建的单例ZKProvider，并把服务的实现类对象传给publishService方法
3. publishService获取本机IP，把要注册的服务名在**本地的MAP**和SET登记等操作，然后调用ZK注册
4. ZK注册实现类就是用Curator，路径设置为/my-rpc/服务接口/本机IP+Host，设置为ZK的Persistent节点
5. 总结：把要调用的服务接口，存到ZK里，节点为自己的IP+端口
6. 注册服务流程结束，SocketRpcServer.start()
7. start里，ServerSocket监听端口，调用socket的accept方法，每收到一个连接，就**把这个socket丢到线程池里处理**（做了一个RPCRequestHandler处理）
8. run()里，把从Stream里读的Object还原成RpcRequest，并把这个RPCRequest传给handle方法，得到返回的结果result
9. rpcRequestHandler的handle方法里，获取serviceProvider的服务名，获取在3中MAP存的服务实现类对象，并通过反射获取方法并传入Request的参数类型，然后调用method.invove(实现类对象,Request的参数)获得result
10. 回到8，把获得的result包装成**RpcResponse**，具体为构造一个success的预定义对象，里面存有result和requestID，从objectOutputStream写出去
    **总结：先在ZK注册服务，具体为服务接口包+名，本地也有个MAP来存储服务方法实现类用于反射；然后服务端监听端口，每一个Socket来了就丢给实现了Runnable接口的处理方法，处理方法里把InputStream的内容还原为RpcRequest，获取MAP中对应的实现类，用反射的方式获取Method（RpcRequest中有方法名和参数类型数组），mothod.invoke后得到结果并返回，从OutputStream返回回去。**

##### server with netty:

1~6的服务注册基本与socket相同，但是使用的是NettyRpcServer，start方法不同

7. start里，创建的ServerBootstrap.group bossGroup和workerGroup两个NioEventLoopGroup，channel用NioServerSocketChannel，在childHandler的pipleline里添加了RpcMessageEncoder和Decoder，还有绑定了DefaultEventExecutorGroup线程池的NettyRpcServerHandler
8. NettyRpcServerHandler里重写channelRead方法，查看msg是否为RpcMessage类，判断这个RpcMessage的类型是心跳还是正常请求，并构造一个用于响应客户端的rpcMessage对象
9. 如果是心跳，则把rpcMessage头设置为心跳回应，Data设置为PONG常数
10. 如果是正常请求，把使用rpcMessage.getData方法得到的内容转型为RpcRequest，调用上方**server with socket:9**的rpcRequestHandler方法处理这个请求，把得到的结果放进一个RpcResponse里（与Request对应，都是RpcMessage的Data），设置这个RpcResponse为success类型，存入这个Rpc请求的RequestID。把封装好的RpcResponse作为rpcMessage的Data
11. ctx.writeAndFlush(rpcMessage)出去

**附childHandler的pipleline里的encode和decoder：**

**decoder :** 目的是把ByteBuf in转换为RpcMessage，继承了Netty的LengthFieldBasedFrameDecoder，可以根据构造的参数调用父类decode方法，准确的读到某个偏移开始的值，但是一顿操作读到的还是整个RpcMessage大小的数据；

把读到的ByteBuf判断，大于Message头部大小的话，调用自己的解码方法。byteBuf读魔数、版本并校验，读总长度、消息类型、序列化压缩类型、请求ID。然后根据这些读到的内容重建一个RpcMessage（这个Message就是给8~10的Handler分析的），如果是心跳类型，则设置内容为固定值然后返回就行

如果是请求，那么需要继续往后面读真正的RpcRequest，根据前面读到的总长度-头部长度，读到byte[]里，根据头里的信息解压并反序列化（反序列化时要根据头中消息类型为request/response在序列化时转型），设置为RpcMessage的Data后返回RpcMessage。

**encoder ：**

目的是把rpcMessage转为ByteBuf out，往out里写上各种头信息，空出长度等等再写，然后把RpcMessage中的data（request/response）序列化，然后写到out里，无返回值

**为什么使用netty的时候需要用Message再包装？**

使用了netty，要有 full length，粘包拆包机制

因为要引入心跳等，所以要有messageType（RPC请求/响应，心跳请求/响应）

压缩类型、序列化类型

#### 代理类：

使用的是JDK动态代理

使用getProxy方法获得代理对象，内部是调用Proxy.newProxyInstance，参数是类加载器、被代理类实现的一些接口，实现了InvocationHandler接口的对象，当动态代理调用一个方法时，这个方法的调用会被转发到实现了InvocationHandler接口的invoke方法来调用

使用ClientProxy代理包装RpcRequest，并且根据是socket还是netty来  调用对应的rpc sendRPCRequest方法

#### 服务注册：

服务被注册进zk时，完整的服务名称作为父节点，子节点是对应的服务地址（ip+port）

然后获取服务时，只需要在完整的服务名称下面找子节点，并调用负载均衡类选择一个合适的就行

返回host+post，包装成InetSocketAddress

#### **封装对象：**

RPC请求对象：RpcRequest

需要封装要调用的方法名/接口名，传的参数，传的参数的类型，一个用于标记的RequestID

RPC响应对象：RpcResponse

内部对象全为private，需要调用success或者fail方法来构建，里面分别设置success的code以及头信息message，并把requestID写进去，再把data写进去，并返回这个Response对象

RPC信息对象：RpcMessage 在用Netty传输的时候要用到

4Byte魔数，1Byte版本号，4Byte信息长度，1Byte信息类型（心跳，RPC请求，RPC响应），压缩类型，序列化类型，请求的ID

一般的二进制协议：头部（魔数，protocol version），长度域，actual content

#### Socket传输：

socket.connect连接，然后获取ObjectOutputStream，然后调用writeObject方法把rpcRequest写入，然后开个ObjectInputStream等待返回，返回从里面得到的内容（RPCResponse）

#### **Netty传输：**

**客户端：**

默认构造方法构造一个bootstrap，group NIOEventLoopGroup，channel使用NIOSocketchannel，handler使用了IdleStateHandler发送心跳，还有编解码器，NettyRpcClientHandler等

NettyRpcClientHandler里实现channelRead，userEventTriggered和exceptionCaught

channelRead对Object msg参数做判断，是否为RpcMessage，判断类型并做出对应反应（是心跳还是RpcResponse）

userEventTriggered与IdleStateHandler有关，用于发送心跳

**服务端：**

也是使用bootstrap创建，绑定bossgroup和workergroup，channel使用NIOServerSocketchannel，

后面的handler里往channel添加了心跳机制，编解码器，还有NettyRpcServerHandler

NettyRpcServerHandler里面也同样是实现channelRead，userEventTriggered和exceptionCaught方法

channelRead

#### 序列化：

序列化：用ByteArrayOutputStream新建一个Hessian2Output，拿新建的Hessian2Output对象writeObject即可，然后返回byteArrayOutputStream.toByteArray()

反序列化：用ByteArrayInputStream新建一个Hessian2Input对象，拿新建的Hessian2Input对象readObject，得到object对象，并转型成传入的class对象

#### 池：

UnprocessedRequests里面是一个static final的map，里面存放requestId和CompletableFuture的映射。client发RPC请求的时候存一个，当收到server回应时把存的CompletableFuture拿出来，并complete

channelMapper也是创建了一个map，把创建的channel（key是inetSocketAddress对象）映射到这个map里

#### 单例工厂模式

serviceProvider是实现了ZkServiceProviderImpl.class的单例对象，调用publishService
