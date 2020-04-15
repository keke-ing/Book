## 第三章：框架搭建 —— server

我们的框架代码地址是 [gorpc](https://github.com/lubanproj/gorpc)

### 一、server 的结构

要搭建 server 层，首先我们要明确 server 层需要支持哪些能力，其实 server 的核心就是提供服务请求的处理能力。server 侧定义服务，发布服务，接收到服务的请求后，根据服务名和请求的方法名去路由到一个 handler 处理器，然后由 handler 处理请求，得到响应，并且把响应数据发送给 client。

按照这个思路，我们可以先定义出 server 的结构，如下：

```go
type Server struct {
   opts *ServerOptions
   services map[string]Service
}
```

ServerOptions 是通过选项模式来透传业务自己指定的一些参数，比如服务监听的地址 address，网络类型 network 是 tcp 还是 udp，后端服务的超时时间 timeout 等。

```go
type ServerOptions struct {
   address string  // listening address, e.g. :( ip://127.0.0.1:8080、 dns://www.google.com)
   network string  // network type, e.g. : tcp、udp
   protocol string  // protocol typpe, e.g. : proto、json
   timeout time.Duration       // timeout
   serializationType string   // serialization type, default: proto

   selectorSvrAddr string       // service discovery server address, required when using the third-party service discovery plugin
   tracingSvrAddr  string         // tracing plugin server address, required when using the third-party tracing plugin
   tracingSpanName string       // tracing span name, required when using the third-party tracing plugin
   pluginNames []string         // plugin name
   interceptors []interceptor.ServerInterceptor
}
```

services 是一个 Service 的 map，每个 Service 表示一个服务，一个 server 可以发布多个服务，用服务名 serviceName 作 map 的 key。我们来看一下 Service 的定义和结构，Service 的接口定义了每个服务需要提供的通用能力，包括 Register （处理函数 Handler 的注册）、提供服务 Serve，服务关闭 Close 等方法

```go
// Service 定义了某个具体服务的通用实现接口
type Service interface {
   Register(string, Handler)
   Serve(*ServerOptions)
   Close()
}

type service struct{
   svr interface{}          // server
   ctx context.Context       // 每一个 service 一个上下文进行管理
   cancel context.CancelFunc   // context 的控制器
   serviceName string        // 服务名
   handlers map[string]Handler
   opts *ServerOptions       // 参数选项
}
```

顺便说一下 service 这个结构，它是 Service 接口的具体实现。它的核心是 handlers 这个 map，每一类请求会分配一个 Handler 进行处理

### 二、server 提供的能力

#### 1、server 创建

server 的创建比较简单，主要是 ServerOptions 和 service map 的初始化，这里通过选项模式对 opts 进行赋值

```
func NewServer(opt ...ServerOption) *Server{
   s := &Server {
      opts : &ServerOptions{},
      services: make(map[string]Service),
   }
   for _, o := range opt {
      o(s.opts)
   }
   return s
}
```

#### 2、服务注册

服务注册主要是将 Service 添加到 server 的 service map 里面，这里就牵涉到服务的定义了，定义一个服务有两种方式

- 通过 go struct 定义

  这种定义方式是按照协议 go struct 的方式，去申明一个服务，例如以下是一个服务的定义：

  ```go
  package helloworld
  
  import "context"
  
  type Service struct {
  
  }
  
  type HelloRequest struct {
     Msg string
  }
  
  type HelloReply struct {
     Msg string
  }
  
  func (s *Service) SayHello(ctx context.Context, req *HelloRequest) (*HelloReply, error) {
     rsp := &HelloReply{
        Msg : "world",
     }
  
     return rsp, nil
  }
  ```

  定义了一个服务之后，通过调用 s.RegisterService 去进行服务注册

  ```go
  s.RegisterService("/helloworld.Greeter", new(helloworld.Service))
  ```

  这里会通过反射的方式，通过 service 引用去获取 service 的每一个方法生成一个 handler，然后添加到 service 的 handler map 里面。当接收到 client 发过来的请求是，通过服务的方法名获取到这个服务的 handler，进而处理请求，得到响应，发送给客户端。反射调用的 demo 可以参考：[helloworld](https://github.com/lubanproj/gorpc/tree/master/examples/helloworld)

- 通过 proto 文件定义

  这种定义方式是按照 protobuf 的标准去定义一个 Service，然后使用框架自带的代码生成工具生成 service的描述文件，然后调用 server 的 RegisterService 方法进行服务注册。例如以下是一个服务的定义

  ```go
  syntax = "proto3";
  
  package helloworld;
  
  service Greeter {
    rpc SayHello (HelloRequest) returns (HelloReply) {}
  }
  
  message HelloRequest {
    string msg = 1;
  }
  
  message HelloReply {
    string msg = 1;
  }
  ```

  定义了一个服务之后，通过框架自带的 proto 生成工具插件，会根据 proto 文件去生成 service 的描述信息，包括服务名、服务每一个方法的方法名和方法的 handler

  ```go
  var _Greeter_serviceDesc = &gorpc.ServiceDesc{
     ServiceName: "helloworld.Greeter",
     HandlerType: (*GreeterService)(nil),
     Methods : []*gorpc.MethodDesc{
        {
           MethodName: "SayHello",
           Handler:    GreeterService_SayHello_Handler,
        },
     },
  }
  ```

  同样会暴露出一个注册服务的 RegisterService 接口供 server 进行调用。跟反射的方式不一样的是，这里是在编译时就已经生成好了调用代码，而反射是在程序运行时去动态生成的。代码生成调用方式的 demo 可参考：[helloworld2](https://github.com/lubanproj/gorpc/tree/master/examples/helloworld2)

### 3、服务运行

服务运行的核心方法是 Serve()，它的逻辑非常简单，就是会遍历 service map 里面所有的 service，然后运行 service 的 Serve 方法

```go
func (s *Server) Serve() {

   for _, service := range s.services {
      go service.Serve(s.opts)
   }

   ch := make(chan os.Signal, 1)
   signal.Notify(ch, syscall.SIGTERM, syscall.SIGINT, syscall.SIGQUIT, syscall.SIGSEGV)
   <-ch

   s.Close()
}
```

service 的 Serve 方法会去构建 transport ，监听客户端请求，根据请求的服务名 serviceName 和请求的方法名 methodName 调用相应的 handler 去处理请求，然后进行回包。



### 4、服务参数的传递 —— 选项模式

这里想单独介绍下选项 (Option) 模式，选项模式的实现方式基本会贯穿整个框架。**几乎所有开源组件对于参数选项的传递都是使用这个模式**。

现在有一个场景，你创建一个 server 的时候，要设置若干不同类型的参数，例如有下列参数：

```go
type ServerOptions struct {
   address string  // listening address, e.g. :( ip://127.0.0.1:8080、 dns://www.google.com)
   network string  // network type, e.g. : tcp、udp
   protocol string  // protocol typpe, e.g. : proto、json
   timeout time.Duration       // timeout
   serializationType string   // serialization type, default: proto

   selectorSvrAddr string       // service discovery server address, required when using the third-party service discovery plugin
   tracingSvrAddr  string         // tracing plugin server address, required when using the third-party tracing plugin
   tracingSpanName string       // tracing span name, required when using the third-party tracing plugin
   pluginNames []string         // plugin name
   interceptors []interceptor.ServerInterceptor
}
```

一般情况下，很多人的实现方式可能是：

```go
func main() {
   opts := &ServerOptions{
      address : "ip://127.0.0.1:8080",
      network : "tcp",
      protocol : "proto",
   }
   NewServer(opts)
}

func NewServer(opts *ServerOptions) *Server {
   return &Server{
      opts: opts,
   }
}

type Server struct {
   opts *ServerOptions
}
```

这种方式要求 ServerOptions 里面的属性 address、network 等必须和调用函数 main 在同一个包下，假如是不同包的话，address、network 要被定义成公共变量 Address 和 Network ，否则无法访问（go 里面首字母小写代表这个变量是私有的，只能同一个包下访问，首字母大写是共有变量，才能在不同包下访问）。一般情况下，考虑到面向对象的封装性，当前包下访问的变量建议定义为私有变量。

一种更为优雅的实现方式即用选项模式实现，它的实现方式就如下面这段代码：

```go
type ServerOptions struct {
   address string  // listening address, e.g. :( ip://127.0.0.1:8080、 dns://www.google.com)
   network string  // network type, e.g. : tcp、udp
   protocol string  // protocol typpe, e.g. : proto、json
}

type ServerOption func(*ServerOptions)

func WithAddress(address string) ServerOption{
   return func(o *ServerOptions) {
      o.address = address
   }
}

func WithNetwork(network string) ServerOption {
   return func(o *ServerOptions) {
      o.network = network
   }
}
```

在创建 Server 的时候我们可能需要指定一些配置信息，比如监听地址 address、网络类型 network，我们是这样传递的

```go
opts := []gorpc.ServerOption{
   gorpc.WithAddress("127.0.0.1:8000"),
   gorpc.WithNetwork("tcp"),
   gorpc.WithProtocol("proto"),
   gorpc.WithTimeout(time.Millisecond * 2000),
}
s := gorpc.NewServer(opts ...)
```

NewServer 里面设置这些参数时会遍历所有的 ServerOption ，进行变量设置，如下：

```go
func NewServer(opt ...ServerOption) *Server{
	s := &Server {
		opts : &ServerOptions{},
		services: make(map[string]Service),
	}

	for _, o := range opt {
		o(s.opts)
	}
  ...
}
```

这样既实现了参数的完美透传，又满足了面向对象的封装性。



### 小结

本章内容主要介绍了 server 的结构和 server 的创建、服务的注册和运行。可以有两种服务定义的方式，参数的透传使用选项模式。下一章我们将会介绍 client

