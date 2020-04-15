## 第四章：框架搭建 —— client

上一章我们讲解了框架搭建的 server 搭建，这一章我们将接着讲 client 模块的搭建。

先贴上代码地址： [gorpc](https://github.com/lubanproj/gorpc) 可以参考 client 模块

### 一、client 的结构

要开发一个 client 模块，首先我们要明确 client 层需要提供哪些能力，client 的核心能力毋庸置疑就是拼装参数，发送一个 rpc 请求，然后收 server 的回包，解析回包，返回结果给业务层。这里面可能牵涉到服务发现、负载均衡、连接池、协议的打解包等，每种能力都有相应的模块去实现，底层通信的能力由 transport 去实现。所以 client 其实就是将这些能力组合到发送请求、处理 server 回包的这个流程里面来。

思路理清楚了，client 的结构就比较简单了。我们先来定义一个 interface 来约定所有的 client 都需要实现通用接口，Invoke 这个方法表示向下游服务发起调用。为什么要这么设计呢？我们的框架 gorpc 中，所有的组件都是可插拔，支持业务自定义的。client 也是，只要业务实现了 Client 这个接口，那么就可以无缝替换 client 这个组件。

```go
// global client interface
type Client interface {
   Invoke(ctx context.Context, req , rsp interface{}, path string, opts ...Option) error
}
```

接下来我们定义 Client 接口的框架默认实现 defaultClient ，如下：

```go
type defaultClient struct {
   opts *Options
}
```

defaultClient 的结构也很简单，它的唯一属性 opts 是参数选项，主要定义了业务调用时需要透传的一些信息，包括服务名 serviceName、方法名 method、调用地址 target 等，跟 server 相同，Options 也是用选项模式进行传值的。

```go
// Options defines the client call parameters
type Options struct {
   serviceName string // service name
   method string // method name
   target string  // format e.g.:  ip:port 127.0.0.1:8000
   timeout time.Duration  // timeout
   network string  // network type, e.g.:  tcp、udp
   protocol   string  // protocol type , e.g. : proto、json
   serializationType string // seralization type , e.g. : proto、msgpack
   transportOpts transport.ClientTransportOptions
   interceptors []interceptor.ClientInterceptor
   selectorName string      // service discovery name, e.g. : consul、zookeeper、etcd
}

type Option func(*Options)

func WithServiceName(serviceName string) Option {
   return func(o *Options) {
      o.serviceName = serviceName
   }
}

func WithMethod(method string) Option {
   return func(o *Options) {
      o.method = method
   }
}
```

### 二、创建 client

创建一个 client 之前，必须想清楚 client 创建的时机，是每次请求的时候创建，还是全局创建一个唯一的 client 呢？

第一种方式每次请求的时候创建一个 client，请求的上下文和参数都是协程私有的，所以不会出现并发安全性问题。但是缺点是每次请求都创建 client，会造成对象的频繁创建和销毁，会造成很大的性能浪费。

第二种方式全局创建换一个唯一的 client，这样的好处是 client 只被创建一次，就不会有频繁创建和销毁导致对象导致的性能消耗，不过由于 client 是全局唯一的，所以多个协程会共用一个 client，在 client 的设计中，就要考虑并发安全问题。

考虑到我们的框架可能会产生每秒钟数万甚至数十万的请求，假如是按照第一种方式去实现，那么对性能的消耗是巨大的，所以我们选择第二种方式实现，即全局创建一个唯一 client。所以，我们很容易想到用单例模式来实现，如下：

```go
// use a global client
var DefaultClient = New()

var New = func() *defaultClient {
   return &defaultClient{
      opts : &Options{
         protocol : "proto",
      },
   }
}
```

### 三、两种调用方式

上一章说到了 server 提供服务有两种方式，一种是通过反射的方式，一种是通过 proto 文件代码生成的方式。与之相对应，client 也有两种方式往下游发起调用。

第一种方式，server 通过反射的方式，动态生成 service 的 handler。与之对应，我们在 client 中为 Invoke 方法上层封装了一个 Call 方法暴露给上游调用，如下：

```go
// call by reflect
func (c *defaultClient) Call(ctx context.Context, servicePath string, req interface{}, rsp interface{},
   opts ...Option) error {

   // reflection calls need to be serialized using msgpack
   callOpts := make([]Option, 0, len(opts)+1)
   callOpts = append(callOpts, opts ...)
   callOpts = append(callOpts, WithSerializationType(codec.MsgPack))

   // servicePath example : /helloworld.Greeter/SayHello
   err := c.Invoke(ctx, req, rsp, servicePath, callOpts ...)
   if err != nil {
      return err
   }

   return nil
}
```

这里主要运用了代理模式的思想，Call 方法在这里完成将业务透传的参数与 defaultClient 的选项参数 Options 进行拼接一起向下游传递。其实这种设计也是为了可以和第二种代码生成的调用方式进行兼容。

第二种方式，server 是通过 proto 文件代码生成的方式，生成 service 的描述信息。同时，会为 service 的每一个方法生成 client 调用的 stub 代码，下面就是生成的一段 stub 代码，它为 client 生成了一个 proxy 代理，同时为 Greeter  这个 service 的 SayHello 这个方法生成了一段调用代码，如下所示：（详情可见：[stub代码](https://github.com/lubanproj/gorpc/blob/master/examples/helloworld2/helloworld/helloworld.gorpc.go)）

```go
func NewGreeterClientProxy(opts ...client.Option) GreeterClientProxy {
   return &GreeterClientProxyImpl{client: client.DefaultClient, opts: opts}
}

func (c *GreeterClientProxyImpl) SayHello(ctx context.Context, req *HelloRequest,
   opts ...client.Option) (*HelloReply, error) {

   callopts := make([]client.Option, 0, len(c.opts)+len(opts))
   callopts = append(callopts, c.opts...)
   callopts = append(callopts, opts...)

   rsp := &HelloReply{}
   err := c.client.Invoke(ctx, req, rsp, "/helloworld.Greeter/SayHello", callopts...)
   if err != nil {
      return nil, err
   }

   return rsp, nil
}
```

### 四、invoke 函数

上面说到了，业务发起 client 后端调用，一共有两种方式。但是无论是反射的方式，还是代码生成的方式，最终都会调用 invoke 函数。invoke 完成了一个客户端的完整动作：

#### 1、request 的序列化

通过客户端透传下来的序列化类型参数，去获取 Serialization 对象，然后通过 Serialization 对 request 进行序列化，会把请求体序列化成二进制数据，如下：

```go
serialization := codec.GetSerialization(c.opts.serializationType)
payload, err := serialization.Marshal(req)
if err != nil {
   return codes.NewFrameworkError(codes.ClientMsgErrorCode, "request marshal failed ...")
}
```

#### 2、对请求体二进制进行打包

经过序列化后，我们得到了二进制的请求体数据，接下来我们需要把数据包，按照我们自定义私有协议的格式，拼装出协议体，再进行 proto 序列化成二进制数据，向下游发送，关于自定义私有协议的格式会在协议一章进行具体介绍，关于打解包的原理会在 codec 章节介绍，也可以参考 [codec](https://github.com/lubanproj/gorpc/tree/master/codec)

```go
// assemble header
request := addReqHeader(ctx, payload)
reqbuf, err := proto.Marshal(request)
if err != nil {
   return err
}
```

#### 3、调用 transport 向 server 发送请求，得到 server 回包

之前我们说到过，底层 tcp 通信的能力是通过 transport 实现的，这里先创建一个 client transport，然后调用 transport 的 Send 函数往下游发送请求，会收到 server 返回的一个完整响应帧数据。

```go
clientTransport := c.NewClientTransport()
clientTransportOpts := []transport.ClientTransportOption {
   transport.WithServiceName(c.opts.serviceName),
   transport.WithClientTarget(c.opts.target),
   transport.WithClientNetwork(c.opts.network),
   transport.WithClientPool(connpool.GetPool("default")),
   transport.WithSelector(selector.GetSelector(c.opts.selectorName)),
   transport.WithTimeout(c.opts.timeout),
}
frame, err := clientTransport.Send(ctx, reqbody, clientTransportOpts ...)
```

#### 4、对 server 回包进行解包

第3步中 server 的响应回包数据是包含了帧头在内的完整帧数据，需要从帧数据里面解析出 server response ，这里会调用 codec Decode 方法进行解包。

```go
rspbuf, err := clientCodec.Decode(frame)
if err != nil {
   return err
}

// parse protocol header
response := &protocol.Response{}
if err = proto.Unmarshal(rspbuf, response); err != nil {
   return err
}
```

#### 5、response 的反序列化

通过调用 Serialization 对象的 Unmarshal 方法进行反序列化，如下：

```go
serialization.Unmarshal(response.Payload, rsp)
```

整个完整的流程可以参考 client 实现 [client](https://github.com/lubanproj/gorpc/blob/master/client/client.go)



### 小结：

本章主要介绍了 client 的结构、client 的创建过程、以及通过 proto 文件代码生成和反射两种方式进行调用。 invoke 函数实现了 client 发送请求的一个完整流程。