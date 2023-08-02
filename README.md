RPC通信原理和技术选型
===
分布式网络通信框架,利用muduo网络库
方法调用，所涉及的标识识别，参数序列化和反序列化利用protobuf
zookeeper作为RPC服务注册中心

### 为什么使用序列化反序列化protobuf，不用json、xml?
1、protobuf是二进制存储的，xml和json都是文本存储
2、Protobuf不需要存储额外信息

### ProtoBuf的使用:
使用时创建xxx.proto文件
使用命令protoc xxx.proto --cpp_out=路径
在路径下生成xxx.pb.h和xxx.pb.cc文件

#### 注意：在proto文件中使用bytes代替string类型，减少字符转化，提高编码效率


### Rpcprovider
rpc服务提供者代码的理解： 需要高并发接收请求，利用muduo库
主要做3件事，用muduo实现网络功能、用protobuf实现序列化和反序列化、在rpc框架上注册服务
1、rpc服务提供方通过RpcProvider向RpcProvider上注册服务对象和服务方法
2、通过protobuf提供的抽象层的service和method把所调用对象和所调用方法记录在一个map表中
3、启动一个muduo库（epoll +多线程的服务器），就可以接收远程的连接
4、远程有新连接过来，muduo库会回调OnConnection方法，不需要做特殊操作，如果有客户端断开，RpcProvider服务端收到了把相应连接shutdown，释放socket资源
5、OnMessage等待远端的调用请求，当数据过来后会按照协商好的数据格式解析，最终解析出service_name和method_name,然后还有参数，方法调用参数的数据。然后从抽象层动态生成method、reques请求和response响应，把参数数据反序列化写入request请求里，
Response响应的数据是由业务部分填写，然后生成Closure的回调，然后在框架上调用业务的方法(service->CallMethod)

### 业务部分
业务里面的方法被调用，从request请求里拿数据执行本地业务，让后填写response响应消息,最后执行done->Run()的回调，回调调用了绑定的方法。
绑定的方法执行就是把响应序列化然后通过网络发送回去，让后由rpc的服务提供者主动关闭这条连接，节省资源，给其他的rpc客户端继续提供服务。

### Rpcconsumer
rpc服务调用者代码的理解： 这一端不需要高并发 普通的tcp调用即可
1、初始化mprpc框架，定义一个代理的stub对象，调用MprpcChannel类的对象，这个MprpcChannel类是继承自RpcChannel类，并重写RpcChannel类的callMethod方法。填写rpc请求的参数，以及对应的response响应.
2、通过stub代理对象调用rpc方法
3、rpc方法实现数据的组装、序列化，然后发送rpc请求，wait等待处理的响应，接收rpc响应，将响应数据反序列化。
4、调用方从rpc调用响应读取结果，或者错误信息的错误号将其打印出来

### 为什用zookeeper?
Rpc网络通信框架中必须在一个地方去找到我现在想调用的Rpc服务在哪一台机器上
需要在分布式环境中使用它的服务配置中心的记录所有发布Rpc的机器的主机ip号和端口号