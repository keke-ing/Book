## 第五章：框架搭建 —— transport 实现

前面我们分别介绍了 client 和 server 层的实现，在介绍 client 和 server 时，都涉及到了 transport。transport 就是传输层通信能力的实现。它提供了基于 tcp/udp 等协议的最底层通信能力的实现。接下来我们就来看看它是怎么实现的。

### 一、transport 接口定义

transport 作为传输层，分为 client 发送方和 server 接收方两种类型，

server 传输层主要提供一种监听和处理请求的能力，定义如下：

```go
type ServerTransport interface {
   // monitoring and processing of requests
   ListenAndServe(context.Context, ...ServerTransportOption) error
}
```

ListenAndServe 这个方法就是用来实现请求的监听和处理，所有的 server transport 都需要实现这个方法，同时设计成 interface 接口的方式，主要是为了实现可插拔，支持业务自定义。同时，假如在底层需要支持其他第三方协议，比如需要支持 http 协议，则只要新增一种支持 http 的 transport 即可。

client 传输层主要提供一种向下游发送请求的能力，定义如下：

```go
type ClientTransport interface {
   // send requests
   Send(context.Context, []byte, ...ClientTransportOption) ([]byte, error)
}
```

Send 这个方法主要是用来发起请求调用，传参除了上下文 context 之外，还有二进制的请求包 request，返回是一个二进制的完整数据帧。这里设计成 interface 接口的形式，同样是为了可插拔、支持业务自定义。

### 二、client/server 通信

我们先从最简单也是最基础的 client/server 通信说起。一个简单 c/s 模型，server 主要功能是监听连接，处理请求，假如使用 tcp 协议实现，代码如下：

**server.go**

```go
func main() {
   lis, err := net.Listen("tcp", "127.0.0.1:8000")
   if err != nil {
      panic(err)
   }

   for {
      conn , err := lis.Accept()
      defer conn.Close()
      if err != nil {
         panic(err)
      }

      buffer := make([]byte, 1024)
      recvNum , err := conn.Read(buffer)

      msg := string(buffer[:recvNum])
      fmt.Println("recv from client: ",msg)

      conn.Write([]byte("world"))
   }

}
```

此时，client 使用 tcp 协议实现如下：

**client.go**

```go
func main() {

   conn , err := net.Dial("tcp","127.0.0.1:8000")
   if err != nil {
      panic(err)
   }

   if _, err := conn.Write([]byte("hello")); err != nil {
      fmt.Println(err)
   }

   buffer := make([]byte, 1024)
   recvNum , err := conn.Read(buffer)

   msg := string(buffer[:recvNum])
   fmt.Println("recv from server: ",msg)
}
```

运行 go run server.go ，然后另起终端运行 go run client.go，server 会收到 client 的 "hello" 请求，cilent 则会收到 server 的 "world" 响应。

### 三、问题延伸

现在我们完成了一个基于 tcp 协议的 client/server 通讯服务。我们之前说到了，transport 的核心能力是提供 client 和 server 的底层通信实现。上面的代码已经实现了 client 和 server 互发消息，是不是已经达到我们的目标了呢？慢，我们之前的目标可是高性能，假如深入思考一下，它其实存在以下几个问题：

1、server 对请求的读写处理是同步的，假如这里同时有两个请求 A、B，只有等请求 A 的读写操作完全完成时，才能进行请求 B 的读写操作，这无疑产生了很严重的等待时间浪费。

2、server 每次监听到一个请求都会创建一个连接 conn，用 conn 去进行数据的读写，读写完成之后连接 conn 会被关闭，假如同时有成千上万个请求需要处理，每次请求都需要创建和销毁 conn 对象，会造成很大的性能损耗。

3、server 和 client 每次处理消息时，需要用到一块内存 buffer 用来进行消息读取，这一块内存也会存在频繁创建和销毁问题。

4、假如 client 或者 server 需要发送的数据包过大，tcp 协议会进行拆包，即把大的数据包拆成多份发送。假如 client 需要发送的数据包过小，tcp 协议底层会进行粘包，即多个数据包合并发送。这种情况下，上面的代码是无法支持的，会出现 client 或者 server 读取到的数据包不完整。

5、client 每次发送一个数据包都需要创建一个连接 conn，假设请求量过大，可能会出现单机连接数不够用的情况。

我们一步步来，逐一进行问题解决。

- 要解决问题1，这里需要对 server 代码进行改造一下，使 server 的读写异步化，这里其实使用 go 语言的协程机制很方便实现，如下：

```go
func main() {
   lis, err := net.Listen("tcp", "127.0.0.1:8000")
   if err != nil {
      panic(err)
   }

   for {
      conn , err := lis.Accept()
      defer conn.Close()
      if err != nil {
         panic(err)
      }


      go handleConn(conn)
   }

}

func handleConn(conn net.Conn) {
   buffer := make([]byte, 1024)
   recvNum , err := conn.Read(buffer)
   if err != nil {
      fmt.Println(err)
   }

   msg := string(buffer[:recvNum])
   fmt.Println("recv from client: ",msg)

   conn.Write([]byte("world"))
}
```

对连接 conn 的处理我们统一放到 handleConn 这个函数中，然后使用 go 关键字，新起一个协程去进行处理，这样就能实现异步化处理请求，避免了同步的时间浪费。

- 要解决问题2，我们可以采用长连接的方式，每次 client 和 server 建立连接后，这个连接默认存活，只有在对端关闭连接时，或者连接一直空闲、超过指定时间没有发送消息时，这个连接才关闭。在连接的存活期内，对对端发来的请求进行循环读写，这样就能避免短连接不断创建和关闭造成的性能损耗。这里只需要更改上面的 handleConn 这个方法即可，如下：

```go
func handleConn(conn net.Conn) {

   for {
      buffer := make([]byte, 1024)
      recvNum , err := conn.Read(buffer)
      if err == io.EOF {
         // client 连接关闭
         break
      }

      if err != nil {
         fmt.Println(err)
         break
      }

      msg := string(buffer[:recvNum])
      fmt.Println("recv from client: ",msg)

      conn.Write([]byte("world"))
   }

}
```

使用一个 for 循环对连接循环读取，当遇到 io.EOF 这个错误时，说明 client 已经关闭连接，循环退出，server 关闭这个连接。对空闲连接的检测放在 client 去做，这里先不讲解，放在连接池 pool 的章节进行介绍。

- 要解决问题3，我们只需要初始化一块大内存，每次消息读取复用这块内存进行读取即可，比较简单，就不详细介绍了。
- 要解决问题4，这里需要两个机制，第一是当客户端包过大时，进行循环发包，直到包发完为止。第二是服务端调用 io.ReadFull 函数，保证读取到的包大小刚好等于客户端发过来的包大小，这样就能实现拆包粘包。对 client 代码改动如下：

```go
func main() {

   conn , err := net.Dial("tcp","127.0.0.1:8000")
   if err != nil {
      panic(err)
   }

   if _, err := conn.Write([]byte("hello")); err != nil {
      fmt.Println(err)
   }

   buffer := make([]byte, 1024)
   recvNum , err := conn.Read(buffer)

   msg := string(buffer[:recvNum])
   fmt.Println("recv from server: ",msg)

   req := []byte("hello")
   sendNum := 0
   num := 0
   // 循环发包
   for sendNum < len(req) {
      num , err = conn.Write(req)
      if err != nil {
         fmt.Println(err)
         break
      }
      sendNum += num
   }

   recvNum , err = conn.Read(buffer)

   msg = string(buffer[:recvNum])
   fmt.Println("recv from server: ",msg)
}
```

同时，改动 server 的 handleConn 方法，使用 io.ReadFull 进行包读取，如下：

```go
func handleConn(conn net.Conn) {

   for {
      buffer := make([]byte, 5)
     // 使用 io.ReadFull 进行包读取
      recvNum , err := io.ReadFull(conn,buffer)
      if err == io.EOF {
         // client 连接关闭
         break
      }

      if err != nil {
         fmt.Println(err)
         break
      }

      msg := string(buffer[:recvNum])
      fmt.Println("recv from client: ",msg)

      conn.Write([]byte("world"))
   }

}
```

这里需要注意的是假如包的大小没有达到指定的 buffer 大小， io.ReadFull 会在这里进行阻塞，一直等到读取完 buffer 大小的包然后再返回，所以这里需要提前知道包的长度，包的长度会在我们自定义协议的协议头里面可以获取到。

- 要解决问题 5，我们可以采用连接池的方式进行连接复用，这里在讲解连接池 pool 的时候会详细讲解，这里就不进行赘述。

### 四、对 udp 请求的处理

udp 请求跟 tcp 请求不同，udp 是无状态的协议。所以没有拆包粘包、长连接这些问题。因为 udp 的包最大不能超过 64 k，所以对 udp 的处理的原理就是 server 分配一块 64k 的内存，去进行包的接收，然后把响应数据发给对端，其他过程跟 tcp 类似，这里就不进行详细介绍，详情可以参考：[transport](https://github.com/lubanproj/gorpc/tree/master/transport)

### 五、与其他组件的交互

由于 client 请求需要向一个指定的 server ip 地址去发送消息，client 调用的时候可能是设置了服务名或者域名。那么首先要跟服务发现模块 resolver 交互，获取后端服务的地址列表，然后再根据客户端设置的负载均衡策略（轮询、随机、加权轮询、一致性哈希等），通过负载均衡算法得到一个 ip 地址。由于引入了连接池 pool 实现了客户端连接复用，所以每次请求需要通过连接池 pool 去获取一个连接 conn。同时，由于每次发送的包是一个附带了协议信息的完整消息，所以这里每次发包，需要调用 codec 的 encode 方法进行打包，每次收包，需要调用 codec 的 decode 方法进行解包。

所以 transport 这里，虽然是底层模块，但其实跟其他组件也是密不可分的，这些组件会在后续一一介绍。结合这些组件的完整代码可以参考：

[client_transport](https://github.com/lubanproj/gorpc/blob/master/transport/client_transport.go)

[server_transport](https://github.com/lubanproj/gorpc/blob/master/transport/server_transport.go)



### 小结

这一章主要介绍了传输层 transport 的实现，从一个简单的 client/server 模型，到一个高性能的组件，再通过其他组件的交互，共同实现了一个高性能、高可用的 transport 组件。