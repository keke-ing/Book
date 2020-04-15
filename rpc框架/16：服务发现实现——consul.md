## 第十六章：服务发现实现——consul

上一章我们介绍了服务发现的具体原理以及技术选型，本章我们将从代码实现的角度来介绍下如何使用 consul 来进行框架服务发现支持。

### 一、consul 介绍

关于 consul 的介绍，最直接的可以参考官方 github 仓库 [consul](https://github.com/hashicorp/consul)，或者直接参考 consul 官网 [https://www.consul.io](https://www.consul.io/) 。这里我们只介绍最基础的环境安装和 quick start

#### 1、consul 安装

进入官网 https://www.consul.io/downloads.html 下载 consul，假如是 mac 系统也可以使用 brew 安装。

```sh
brew install consul
```

验证是否安装成功，打开一个命令行工具，输入 consul 命令，如下：

```go
DEVLINTANG-MB0:Documents delvin$ consul
Usage: consul [--version] [--help] <command> [<args>]

Available commands are:
    acl            Interact with Consul's ACLs
    agent          Runs a Consul agent
    catalog        Interact with the catalog
    config         Interact with Consul's Centralized Configurations
    connect        Interact with Consul Connect
    debug          Records a debugging archive for operators
    event          Fire a new event
    exec           Executes a command on Consul nodes
    force-leave    Forces a member of the cluster to enter the "left" state
    info           Provides debugging information for operators.
    intention      Interact with Connect service intentions
    join           Tell Consul agent to join cluster
    keygen         Generates a new encryption key
    keyring        Manages gossip layer encryption keys
    kv             Interact with the key-value store
    leave          Gracefully leaves the Consul cluster and shuts down
    lock           Execute a command holding a lock
    login          Login to Consul using an auth method
    logout         Destroy a Consul token created with login
    maint          Controls node or service maintenance mode
    members        Lists the members of a Consul cluster
    monitor        Stream logs from a Consul agent
    operator       Provides cluster-level tools for Consul operators
    reload         Triggers the agent to reload configuration files
    rtt            Estimates network round trip time between nodes
    services       Interact with services
    snapshot       Saves, restores and inspects snapshots of Consul server state
    tls            Builtin helpers for creating CAs and certificates
    validate       Validate config files/directories
    version        Prints the Consul version
    watch          Watch for changes in Consul
```

假如出现上面的指令列表，则说明 consul 安装成功。

#### 2、consul 的运行

完成 consul 的安装后，就可以开始运行 agent，agent 可以运行为 server 或者 client 模式。每个数据中心必须至少拥有一个 server。一般建议在一个集群中部署 3~5 个 server，避免出现单点故障导致数据丢失。

假如 consul agent 的 client 模式，表现为一个非常轻量级的进程，用于注册服务、运行健康检查和转发对 server 的查询。

在 consul agent 的 client 模式下，所有注册到当前节点的服务会被转发到 server，本身是不持久化这些信息。在 consul agent 的 server 模式下，功能和 client 模式一样，唯一不同的是，它会把所有的信息持久化的本地，这样遇到故障，信息是可以被保留的。

快速启动一个单点的 consul，我们可以运行 

```sh
consul agent -dev
```

执行这个命令会输出一些日志，如下：

```sh
DEVLINTANG-MB0:Documents delvin$ consul agent -dev
==> Starting Consul agent...
           Version: 'v1.6.2'
           Node ID: '15993850-f9f0-1e07-08d0-7a27723c178b'
         Node name: 'DEVLINTANG-MB0'
        Datacenter: 'dc1' (Segment: '<all>')
            Server: true (Bootstrap: false)
       Client Addr: [127.0.0.1] (HTTP: 8500, HTTPS: -1, gRPC: 8502, DNS: 8600)
      Cluster Addr: 127.0.0.1 (LAN: 8301, WAN: 8302)
           Encrypt: Gossip: false, TLS-Outgoing: false, TLS-Incoming: false, Auto-Encrypt-TLS: false

==> Log data will now stream in as it occurs:

    2020/04/06 18:00:33 [DEBUG] agent: Using random ID "15993850-f9f0-1e07-08d0-7a27723c178b" as node ID
    2020/04/06 18:00:33 [DEBUG] tlsutil: Update with version 1
    2020/04/06 18:00:33 [DEBUG] tlsutil: OutgoingRPCWrapper with version 1
    2020/04/06 18:00:33 [INFO]  raft: Initial configuration (index=1): [{Suffrage:Voter ID:15993850-f9f0-1e07-08d0-7a27723c178b Address:127.0.0.1:8300}]
    2020/04/06 18:00:33 [INFO]  raft: Node at 127.0.0.1:8300 [Follower] entering Follower state (Leader: "")
    2020/04/06 18:00:33 [INFO] serf: EventMemberJoin: DEVLINTANG-MB0.dc1 127.0.0.1
    2020/04/06 18:00:33 [INFO] serf: EventMemberJoin: DEVLINTANG-MB0 127.0.0.1
    2020/04/06 18:00:33 [INFO] consul: Adding LAN server DEVLINTANG-MB0 (Addr: tcp/127.0.0.1:8300) (DC: dc1)
    2020/04/06 18:00:33 [INFO] consul: Handled member-join event for server "DEVLINTANG-MB0.dc1" in area "wan"
    2020/04/06 18:00:33 [INFO] agent: Started DNS server 127.0.0.1:8600 (udp)
    2020/04/06 18:00:33 [INFO] agent: Started DNS server 127.0.0.1:8600 (tcp)
    2020/04/06 18:00:33 [INFO] agent: Started HTTP server on 127.0.0.1:8500 (tcp)
    2020/04/06 18:00:33 [INFO] agent: Started gRPC server on 127.0.0.1:8502 (tcp)
    2020/04/06 18:00:33 [INFO] agent: started state syncer
==> Consul agent running!
    2020/04/06 18:00:33 [WARN]  raft: Heartbeat timeout from "" reached, starting election
    2020/04/06 18:00:33 [INFO]  raft: Node at 127.0.0.1:8300 [Candidate] entering Candidate state in term 2
    2020/04/06 18:00:33 [DEBUG] raft: Votes needed: 1
    2020/04/06 18:00:33 [DEBUG] raft: Vote granted from 15993850-f9f0-1e07-08d0-7a27723c178b in term 2. Tally: 1
    2020/04/06 18:00:33 [INFO]  raft: Election won. Tally: 1
    2020/04/06 18:00:33 [INFO]  raft: Node at 127.0.0.1:8300 [Leader] entering Leader state
    2020/04/06 18:00:33 [INFO] consul: cluster leadership acquired
    2020/04/06 18:00:33 [INFO] consul: New leader elected: DEVLINTANG-MB0
    2020/04/06 18:00:33 [INFO] connect: initialized primary datacenter CA with provider "consul"
    2020/04/06 18:00:33 [DEBUG] consul: Skipping self join check for "DEVLINTANG-MB0" since the cluster is too small
    2020/04/06 18:00:33 [INFO] consul: member 'DEVLINTANG-MB0' joined, marking health alive
    2020/04/06 18:00:34 [DEBUG] agent: Skipping remote check "serfHealth" since it is managed automatically
    2020/04/06 18:00:34 [INFO] agent: Synced node info
    2020/04/06 18:00:35 [DEBUG] tlsutil: OutgoingRPCWrapper with version 1
    2020/04/06 18:00:36 [DEBUG] agent: Skipping remote check "serfHealth" since it is managed automatically
    2020/04/06 18:00:36 [DEBUG] agent: Node info in sync
    2020/04/06 18:00:36 [DEBUG] agent: Node info in sync
```

-dev 这个参数是创建一个开发环境的 server 节点。该参数配置下，不会有任何持久化操作，即不会有任何数据写入到磁盘，所以它不能用于生产环境。

仔细看下面这一段信息：

```sh
2020/04/06 18:00:33 [INFO] agent: Started DNS server 127.0.0.1:8600 (udp)
2020/04/06 18:00:33 [INFO] agent: Started DNS server 127.0.0.1:8600 (tcp)
2020/04/06 18:00:33 [INFO] agent: Started HTTP server on 127.0.0.1:8500 (tcp)
2020/04/06 18:00:33 [INFO] agent: Started gRPC server on 127.0.0.1:8502 (tcp)
2020/04/06 18:00:33 [INFO] agent: started state syncer
```

可以看到 consul 启动了 DNS 、HTTP、gRPC 三个 server 来分别监听不同协议的请求。由于我们的实现是会采用 HTTP 协议通信，所以将会去请求 127.0.0.1:8500 这个地址。

我们可以执行下面命令来查看这个节点的信息：

```sh
consul members
```

显示的信息如下：

```sh
Node            Address         Status  Type    Build  Protocol  DC   Segment
DEVLINTANG-MB0  127.0.0.1:8301  alive   server  1.6.2  2         dc1  <all>
```

可以看到，consul agent -dev 这条命令创建了一个 server 类型的节点。

### 三、实现思路

上面对 consul 做了简单的介绍。要基于 consul 实现服务发现，我们首先需要明确实现思路。

#### 1、服务什么时候注册？

服务注册，框架在 Server 中提供了注册 Service 的 Register 方法。Register 中会将一个或多个 Service 添加到一个 Server 中。此时，这个 Service 应该去向 consul server 发起注册，consul 会将 Service 的服务名和服务地址保存在一个 k-v 存储中。与 consul 的通信有 DNS、HTTP、gRPC 三种协议，我们采取通用的 HTTP 协议实现。

#### 2、服务发现怎么实现？

因为我们采用的是客户端服务发现的模式，所以由 client 去请求 consul server，根据某个 Service 的服务名，去 consul 的 k-v 存储中获取到服务的地址。通信的方式也是 HTTP 协议。

#### 3、consul 如何初始化？

这里由于引入了 consul，要想使用 consul 提供的服务发现能力，我们在启动 Server 之前，要先进行 consul 的初始化，启动 consul 服务，这样才能保证 consul 能够接收我们的服务注册、服务发现的请求。

关于 consul 的初始化过程，我们将其统一放在插件初始化过程中去实现。这符合框架的插件化、可插拔的思想。

### 四、具体实现

明确了实现思路，接下来我们就开始代码实现。

首先，上面说到了，consul 是以插件的形式去实现，也就是在 plugin 目录下。

#### 1、定义 consul

consul 定义如下，由于 consul 是一个独立的服务，所以它有自己的一套配置，我们在 Consul 的定义里面加上了一些 consul 配置的属性。

```go
// Consul implements the server discovery specification
type Consul struct {
   opts *plugin.Options
   client *api.Client
   config *api.Config
   balancerName string  // load balancing mode, including random, polling, weighted polling, consistent hash, etc
   writeOptions *api.WriteOptions
   queryOptions *api.QueryOptions
}
```

#### 2、实现 Selector

由于 consul 是服务发现组件，所以它需要实现服务发现的统一接口 Selector，也就是 Select 函数，如下：

```go
// implements selector Select method
func (c *Consul) Select(serviceName string) (string, error) {

   nodes, err := c.Resolve(serviceName)

   if nodes == nil || len(nodes) == 0 || err != nil {
      return "", err
   }

   balancer := selector.GetBalancer(c.balancerName)
   node := balancer.Balance(serviceName,nodes)

   if node == nil {
      return "", fmt.Errorf("no services find in %s", serviceName)
   }

   return parseAddrFromNode(node)
}
```

Select 就分为两步了，第一步 Resolve 方法其实就是服务发现的过程，Balance 是负载均衡实现。这里后续会介绍，这里我们聚焦于服务发现，也就是 Resolve 这个方法，看看它是怎么去实现服务发现的。

```go
func (c *Consul) Resolve(serviceName string) ([]*selector.Node, error) {

   pairs, _, err := c.client.KV().List(serviceName, nil)
   if err != nil {
      return nil, err
   }

   if len(pairs) == 0 {
      return nil, fmt.Errorf("no services find in path : %s", serviceName)
   }
   var nodes []*selector.Node
   for _, pair := range pairs {
      nodes = append(nodes, &selector.Node {
         Key : pair.Key,
         Value : pair.Value,
      })
   }
   return nodes, nil
}
```

可以看到，Resolve 这个方法是通过一个服务名去获取服务列表。它的核心是调用了 c.client.KV().List(serviceName, nil) 这个方法，这里 c 是 Consul 对象，client 是 *api.Client，其实这里我们是通过 consul 的 api 去与 consul 进行通信，具体 api 是通过引用 github.com/hashicorp/consul/api 这个包进行实现，具体可以参考：[consul api](https://github.com/hashicorp/consul/tree/master/api) 。通过 api KV 的 List 方法，从 consul k-v 存储中获取服务列表。

#### 3、实现 Plugin

由于 consul 是以插件化的形式实现的，所以我们同样也需要实现插件的通用接口 Plugin。这里由于服务发现插件和 tracing 的插件都有其个性化配置，导致其初始化方法的结构不一样，所以细化了一下 Plugin 接口，分为 ResolverPlugin 和 TracingPlugin 两类，如下：

```go
// ResolverPlugin defines the standard for all server discovery plug-ins
type ResolverPlugin interface {
   Init(...Option) error
}

// TracingPlugin defines the standard for all tracing plug-ins
type TracingPlugin interface {
   Init(...Option) (opentracing.Tracer, error)
}
```

这里我们需要去实现 ResolverPlugin 这个接口，如下：

```go
func (c *Consul) Init(opts ...plugin.Option) error {

   for _, o := range opts {
      o(c.opts)
   }

   if len(c.opts.Services) == 0 || c.opts.SvrAddr == "" || c.opts.SelectorSvrAddr == "" {
      return fmt.Errorf("consul init error, len(services) : %d, svrAddr : %s, selectorSvrAddr : %s",
         len(c.opts.Services), c.opts.SvrAddr, c.opts.SelectorSvrAddr)
   }

   if err := c.InitConfig(); err != nil {
      return err
   }

   for _, serviceName := range c.opts.Services {
      nodeName := fmt.Sprintf("%s/%s", serviceName, c.opts.SvrAddr)

      kvPair := &api.KVPair{
         Key : nodeName,
         Value : []byte(c.opts.SvrAddr),
         Flags: api.LockFlagValue,
      }

      if _, err := c.client.KV().Put(kvPair, c.writeOptions); err != nil {
         return err
      }
   }


   return nil
}
```

前面我们说到了，Server 启动的时候需要将 Service 去 consul 上进行注册，这个方法实现了服务注册的过程。想将服务名和服务地址包装秤 KVPair 的形式，然后通过调用 consul api 的 Put 方法进行的将 KVPair 同步到 consul 的 k-v 存储。

#### 4、服务初始化过程

上面说到了 Init(opts ...plugin.Option) 这个方法实现了服务初始化。那么它什么时候被调用呢？

其实就是在服务启动之前，其实就是在 Serve() 这个方法里面，我们来看：

```go
func (s *Server) Serve() {

   err := s.InitPlugins()
   if err != nil {
      panic(err)
   }

   for _, service := range s.services {
      go service.Serve(s.opts)
   }

   ch := make(chan os.Signal, 1)
   signal.Notify(ch, syscall.SIGTERM, syscall.SIGINT, syscall.SIGQUIT, syscall.SIGSEGV)
   <-ch

   s.Close()
}
```

在 service 的 Serve 方法启动前，我们先进行插件的初始化，也就是 InitPlugins 这个方法。这个方法里面会去遍历所有的插件，然后进行插件初始化。

```go
func (s *Server) InitPlugins() error {
   // init plugins
   for _, p := range s.plugins {

      switch val := p.(type) {

      case plugin.ResolverPlugin :
         var services []string
         for serviceName, _ := range s.services {
            services = append(services, serviceName)
         }

         pluginOpts := []plugin.Option {
            plugin.WithSelectorSvrAddr(s.opts.selectorSvrAddr),
            plugin.WithSvrAddr(s.opts.address),
            plugin.WithServices(services),
         }
         if err := val.Init(pluginOpts ...); err != nil {
            log.Errorf("resolver init error, %v", err)
            return err
         }

    	....
      }


   }

   return nil
}
```

如上，当发现是服务发现类插件 ResolverPlugin，就调用 Init 方法进行初始化，所以 consul 的初始化动作就是在这一步进行。

#### 5、客户端初始化

我们知道，服务端 server 需要和 consul 通信来进行服务注册。那么客户端 client 也要和 consul 通信来进行服务发现，那么 client 如何知道 consul 地址呢？这里也需要在 client 进行一步 consul 初始化动作。如下：

```go
func main() {
   opts := []client.Option {
      client.WithTarget("127.0.0.1:8000"),
      client.WithNetwork("tcp"),
      client.WithTimeout(2000 * time.Millisecond),
      client.WithSelectorName(consul.Name),
   }
   c := client.DefaultClient
   req := &helloworld.HelloRequest{
      Msg: "hello",
   }
   rsp := &helloworld.HelloReply{}

   consul.Init("localhost:8500")
   err := c.Call(context.Background(), "/helloworld.Greeter/SayHello", req, rsp, opts ...)
   fmt.Println(rsp.Msg, err)
}
```

客户端进行 Call 调用下游之前，需要调用 consul.Init 方法设置 consul server 的监听地址，如下：

```go
// Init implements the initialization of the consul configuration when the framework is loaded
func Init(consulSvrAddr string, opts ... plugin.Option) error {
   for _, o := range opts {
      o(ConsulSvr.opts)
   }

   ConsulSvr.opts.SelectorSvrAddr = consulSvrAddr
   err := ConsulSvr.InitConfig()
   return err
}
```

这里的 Init 与 server 端调用 consul Init 一样，底层都是调用了 InitConfig 方法进行配置初始化。到这里，整个链路算是完整的串联起来。

### 小结

本章先对 consul 做了一个快速介绍，然后介绍了框架服务发现实现的思路和具体代码实现。代码 demo 可以参考：[consul](https://github.com/lubanproj/gorpc/tree/master/plugin/consul)

