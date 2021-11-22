# Apiserver-network-proxy源码解读

k8s支持使用ssh隧道来保护从控制面到节点的通信路径。在这种配置下，apiserver建立一个到集群中各节点的ssh隧道并通过这个隧道传输所有到kubelet、节点、pod或服务的请求。然而ssh隧道目前已被弃用，Konnectivity是对此通信通道的替代品，它提供TCP层的代理，包括两部分：Konnectivity server和Konnectivity agent，分别运行在控制面网络和节点网络中。启动Konnectivity服务后，所有控制面到节点的通信都通过这些连接传输。

## Proposal

将用户发起的网络流量与 API 服务器发起的流量分开是一个有用的概念。云提供商希望控制 API 服务器到 Pod、节点和服务网络流量的实现方式。云提供商可以选择在隔离网络上运行他们的 API 服务器（控制网络）和集群节点（集群网络）。

目标: 允许将控制网络与集群网络隔离

值得注意的是: 节点网络可能与主网络完全脱节, 它可能具有与主网络或其他网络隔离方式重叠的IP地址.应假设集群和主网络之间的直接 IP 可路由性。以后的版本可能会放宽对所有节点的要求。

![image-20211111200930045](https://tva1.sinaimg.cn/large/008i3skNly1gwbgwqlp50j30kx0dht9e.jpg)



目前,支持三种KAS的出站请求类型: Master、Etcd、Cluster. 这三种网络被称为Network Context

```go
type EgressType int

const (
    // Master is the EgressType for traffic intended to go to the control plane.
    Master EgressType = iota
    // Etcd is the EgressType for traffic intended to go to Kubernetes persistence store.
    Etcd
    // Cluster is the EgressType for traffic intended to go to the system being managed by Kubernetes.
    Cluster
)

// NetworkContext is the struct used by Kubernetes API Server to indicate where it intends traffic to be sent.
type NetworkContext struct {
    // EgressSelectionName is the unique name of the
    // EgressSelectorConfiguration which determines
    // the network we route the traffic to.
    EgressSelectionName EgressType
}
```



## gRPC服务定义

整个项目中,server和agent是双向流模式的gRPC服务.这里需要了解gRPC服务的一些底层原理和使用.

proxy server是一个grpc服务器, 它对外一共提供两种服务: ProxyService和AgentService,都是双向流模式的gRPC服务.

```protobuf
service ProxyService {
  rpc Proxy(stream Packet) returns (stream Packet) {}
}

service AgentService {
  // Agent Identifier?
  rpc Connect(stream Packet) returns (stream Packet) {}
}

message Packet {
  PacketType type = 1;

  oneof payload {
    DialRequest dialRequest = 2;
    DialResponse dialResponse = 3;
    Data data = 4;
    CloseRequest closeRequest = 5;
    CloseResponse closeResponse = 6;
    CloseDial closeDial = 7;
  }
}
```

从定义就可以看出,ProxyService是提供给apiserver等需要代理的客户端的,而AgentService是提供给agent来建立反向代理的.

PacketType一共有6种类型.

![image-20211118210441139](https://tva1.sinaimg.cn/large/008i3skNly1gwjluakchmj30lj060jrg.jpg)

- DialRequest: 发起对后端服务的访问请求(这个后端服务就是agent要代理的服务),该数据包字段有protocol、后端服务的地址address、一个连接标识random.

  ![image-20211118211316777](https://tva1.sinaimg.cn/large/008i3skNly1gwjm38tx0mj30fz076wel.jpg)

- DialResponse: 当proxy agent收到DialRequest数据包时,返回connectID和random. connectID是与后端服务建立的TCP连接的标识.

  ![image-20211119094731912](https://tva1.sinaimg.cn/large/008i3skNly1gwk7vzqw3qj30hs07dwep.jpg)

- CloseRequest: 断开连接请求,数据包中只有一个connectID.

- CloseReqspnse: 断开连接响应,数据包中有错误信息error和connectID.

- CloseDial: 只有一个DialRequse中的random

- Data: 这个是正常的数据包.

  ![image-20211119095011782](https://tva1.sinaimg.cn/large/008i3skNly1gwk7yrxgjhj30hy075glp.jpg)

## Proxy Server

A gRPC proxy server, receives requests from the API server and forwards to the agent.

proxy server一共提供两种gRPC服务,一种是ProxyService,代理上游客户端连接,一种是AgentService,供下游proxy agent反向连接使用.



### BackendManager

<font color=red>BackendManager是proxy server管理与proxy agent的gRPC连接的组件</font>

Proxy server与proxy agent的gRPC连接被封装成一个backend结构体,它实现了Backend接口.

```go
type Backend interface {
	Send(p *client.Packet) error
	Context() context.Context
}

type backend struct {
	// TODO: this is a multi-writer single-reader pattern, it's tricky to
	// write it using channel. Let's worry about performance later.
	mu   sync.Mutex // mu protects conn
	conn agent.AgentService_ConnectServer
}
```

可以看到,里面有一个AgentService服务Connect方法的流引用.<font color=red>所以可以把Backend当成一个proxy server和proxy agent的gRPC连接.</font>

BackendManager的功能为: 根据proxy策略选择一个Backend,存储所有的Backend,检查Backend是否准备就绪.

![image-20211119164415731](https://tva1.sinaimg.cn/large/008i3skNly1gwkjxpfxhvj30wl0grjtc.jpg)

根据不同的代理策略,一共有三种BackendManager实现,分别以不同的方式管理着Backend.

首先三种代理策略分别为:

- default: 这种策略下proxy server会随机选择一个健康的backend建立隧道(这个建立隧道是proxy agent到后端服务的TCP连接,而不是proxy server和proxy agent的gRPC连接, 这个gRPC连接是通过proxy agent反向建立的)
- destHost: 这种策略下,proxy server会选择一个主机名与request.Host相同的backend
- defaultRoute: 这种策略下,只会将流量转发到已通过agent identifier显式表明它们为默认路由提供服务的代理.

可以看到,不同的manager的唯一区别就是Backend方法的实现,即如何选择backend.而存储Backend的`BackendStroage`实现方式只有一种.

#### DefaultBackendStorage

```go
// DefaultBackendStorage is the default backend storage.
type DefaultBackendStorage struct {
	mu sync.RWMutex //protects the following
	// A map between agentID and its grpc connections.
	// For a given agent, ProxyServer prefers backends[agentID][0] to send
	// traffic, because backends[agentID][1:] are more likely to be closed
	// by the agent to deduplicate connections to the same server.
	backends map[string][]*backend
	// agentID is tracked in this slice to enable randomly picking an
	// agentID in the Backend() method. There is no reliable way to
	// randomly pick a key from a map (in this case, the backends) in
	// Golang.
	agentIDs []string
	// defaultRouteAgentIDs tracks the agents that have claimed the default route.
	defaultRouteAgentIDs []string
	random               *rand.Rand
	// idTypes contains the valid identifier types for this
	// DefaultBackendStorage. The DefaultBackendStorage may only tolerate certain
	// types of identifiers when associating to a specific BackendManager,
	// e.g., when associating to the DestHostBackendManager, it can only use the
	// identifiers of types, IPv4, IPv6 and Host.
	idTypes []pkgagent.IdentifierType
}
```

- backends: 存储着agentID到与它的gRPC连接的映射.可能觉得proxy agent可以拥有相同的agentID,所以这里backend为slice.<font color=red>但相同agentID的backend,proxy server只选择第一个去使用.</font>
- agentIDs: 存储着已经建立连接的agentID
- defaultRouteAgentIDs: 存储着身份标识为defaultRoute的agentID.
- random: 随机数生成器.用来随机返回某个agentID下的backend供proxy server使用
- idTypes: 存储着该manager所能接受的合法的agent身份标识.比如DestHostBackendManager只能存储IPv4、IPv6、Host类型的agent.所以当创建DefaultBackendStorage的时候,只需要传入所能接受的idTypes集合就行了.

AddBackend、RemoveBackend和NumBackend方法的逻辑也很简单.

除了实现了BackendStorage接口中的三个方法外,DefaultBackendStorage还实现了一个GetRandomBackend()方法,用来随机返回已经存储的某一个agentID下的backend.

![image-20211119180548140](https://tva1.sinaimg.cn/large/008i3skNly1gwkmak2x16j30pt09iq42.jpg)

还有一个RedinessManager下的Ready()方法, 该方法就判断backend数量是否为0.

![image-20211122095909996](https://tva1.sinaimg.cn/large/008i3skNly1gwnp34mk3pj30m60axwf9.jpg)

#### DefaultBackendManager

由前面得知,不同的BackendManager只是Backend()方法的实现逻辑不一样,即从Storage中获取Backend的方式不同.而他们后端的存储统一使用DefaultBackendManager.所以他们有统一的结构体定义,即: 内嵌一个DefaultBackendStorage结构体.

```go
type XXXBackendManager struct {
  *DefaultBackendStorae
}
```

DefaultBackendManager是从DefaultBackendManager中随机获取一个backend.即:<font color=red>随机获取一个gRPC连接传递信息.它支持的IdentifierType只有UID类型.</font>

![image-20211122101533158](https://tva1.sinaimg.cn/large/008i3skNly1gwnpk4djjzj30ow06n3z3.jpg)

#### DestHostBackendManager

<font color=red>他所支持的IdentifierType有IPv4、IPv6、Host.</font>

![image-20211122102207949](https://tva1.sinaimg.cn/large/008i3skNly1gwnpqzhngtj30zl0c875o.jpg)

DestHostBackendManager就是从context中获取k-v信息,然后去backends里面找.

#### DefaultRouteBackendManager

<font color=red>他所支持的IdentifierType只有default-route.</font>

![image-20211122102632653](https://tva1.sinaimg.cn/large/008i3skNly1gwnpvke3clj30px0a4dh6.jpg)

而他获取backend的方式就是随机从defaultRouteAgentID里面获取一个agentID,返回该连接.以DefaultRoute方式代理的agentID单独存放在一个数组里.

### ProxyClientConnection

ProxyClientConnection代表了逻辑上的一个客户端代理连接,即:真正客户端到proxy server之间的连接.后端服务的所有数据都要通过proxy server返回给客户端.

```go
type ProxyClientConnection struct {
	Mode      string
	Grpc      client.ProxyService_ProxyServer
	HTTP      io.ReadWriter
	CloseHTTP func() error
	connected chan struct{}
	connectID int64
	agentID   string
	start     time.Time
	backend   Backend
}
```

该结构体只有一个send()方法,用来返回给客户端数据.

Mode: 客户端连接proxy server的方式, 支持两种: grpc和http-connect.

Grpc: 客户端以gRPC方式连接到proxy server时的流引用.

HTTP: 如果Mode为http-connect, proxy server返回给客户端的数据通过此方法返回.

CloseHTTP: 当proxy agent返回给proxy server一个CloseResponse时候,如果proxy server时http-connect的方式,将会调用此方法关闭http连接

backend: 前端连接对应的后端连接,客户端发送给proxy server的数据, proxy server要找到对应的后端连接才能把数据发送到正确的proxy agent.

### 结构定义

```go
type PendingDialManager struct {
	mu          sync.RWMutex
	pendingDial map[int64]*ProxyClientConnection
}

// ProxyServer
type ProxyServer struct {
	// BackendManagers contains a list of BackendManagers
	BackendManagers []BackendManager

	// Readiness reports if the proxy server is ready, i.e., if the proxy
	// server has connections to proxy agents (backends). Note that the
	// proxy server does not check the healthiness of the connections,
	// though the proxy agents do, so this readiness check might report
	// ready but there is no healthy connection.
	Readiness ReadinessManager

	// fmu protects frontends.
	fmu sync.RWMutex
	// conn = Frontend[agentID][connID]
	frontends map[string]map[int64]*ProxyClientConnection

	PendingDial *PendingDialManager

	serverID    string // unique ID of this server
	serverCount int    // Number of proxy server instances, should be 1 unless it is a HA server.

	// Allows a special debug flag which warns if we write to a full transfer channel
	warnOnChannelLimit bool

	// agent authentication
	AgentAuthenticationOptions *AgentTokenAuthenticationOptions

	proxyStrategies []ProxyStrategy
}
```

BackendManagers: 不同代理类型的backendmanager,即: proxy server为不同的代理方式提供不同的backend存储和管理方式.

Readiness: 检查proxy server是否就绪,即是否和proxy agent有连接,而不检查连接是否正常(proxy agent会检查),所以proxy server可能就绪但连接不健康.

frontends: 存储着proxy server到真正客户端的连接.通过frontends\[agentID][connID]来唯一确定一个真正客户端到proxy server的连接.connID是proxy agent与真正后端服务建立的tcp连接.

PendingDial: 

AgentAuthenticationOptions: 用来进行身份验证的

proxyStrategies: 存储着该proxy server的代理方式, 不同的存储方式会有不同的BackendManager负责后端连接, 该字段在初始化时候会给出.

通过结构定义可以看到: <font color=red>proxy server同时管理着到后端代理proxy agent的gRPC连接和与真正前面的连接</font>

### 连接管理

#### 后端连接管理

proxy server和proxy agent之间的gRPC连接通过BackendManager管理着,proxy server实现的addBackend、getBackend和removeBackend方法也都是通过遍历这些manager然后调用他们底层的增删改方法实现的.

在proxy agent反向连接proxy server的时候,会在header传入`agentID`和`identifiers`字段,proxy server就是根据这些标识来把gRPC连接存储到对应的BackendManager中.

![image-20211122122134799](https://tva1.sinaimg.cn/large/008i3skNly1gwnt7akzzgj31030q9dkl.jpg)

#### 前端连接管理

通过agentID和connID来唯一确定一个客户端和proxy server之间的连接,即: ProxyClientConneciton, 前端连接都存储在frontedns变量中.

#### 两者关系

我们知道多个客户端可以发送请求到同一个proxy server,所以前端连接和后端gRPC连接是 `1:n` 的关系.

通过ProxyClientConnection的backend字段可以拿到对应的后端连接

也可以给定agentID和backend查找所有对应的前端连接.

![image-20211122140941834](https://tva1.sinaimg.cn/large/008i3skNly1gwnwbrrm1qj30xq0bl3zm.jpg)



<img src="https://tva1.sinaimg.cn/large/008i3skNgy1gwnwimnup2j30tq0n9gn7.jpg" alt="未命名文件 (2)" style="zoom:67%;" />



### 服务定义

proxy server一共提供两种gRPC方法: 一个用于proxy agent的Connect方法,一个用于代理客户端的Proxy方法.

#### Connect



## Proxy Agent

A gRPC proxy agent, connects to the proxy and then allows traffic to be forwarded to it.



### Client

#### 1、结构定义

client运行在节点网络中, 它主动与运行在控制网络中的proxy server建立连接.<font color=red>可以把Client想象成一条封装好的到proxy server实例的连接.</font>

```go
type Client struct {
	nextConnID int64

	connManager *connectionManager

	cs *ClientSet // the clientset that includes this AgentClient.

	stream           agent.AgentService_ConnectClient
	agentID          string
	agentIdentifiers string
	serverID         string // the id of the proxy server this client connects to.

	// connect opts
	address string
	opts    []grpc.DialOption
	conn    *grpc.ClientConn
	stopCh  chan struct{}
	// locks
	sendLock      sync.Mutex
	recvLock      sync.Mutex
	probeInterval time.Duration // interval between probe pings

	// file path contains service account token.
	// token's value is auto-rotated by kubernetes, based on projected volume configuration.
	serviceAccountTokenPath string

	warnOnChannelLimit bool
}
```

几个关键字段:

- nextConnID: 标记下一次连接的connectID,此ConnctID是client与后端服务建立的tcp连接标识.一个client建立一个与proxy server的反向gRPC连接,多个与同一子网下的后端服务的TCP连接.而代理的这些TCP连接同一由connManager管理.

- connManager: 即管理着connectID到TCP连接的映射,只不过这里的TCP连接被connContext结构体包装了一下.

  ```go
  // connContext tracks a connection from agent to node network.
  type connContext struct {
  	conn      net.Conn
  	connID    int64
  	cleanFunc func()
  	dataCh    chan []byte
  	cleanOnce sync.Once
  	warnChLim bool
  }
  
  type connectionManager struct {
  	mu          sync.RWMutex
  	connections map[int64]*connContext
  }
  ```

  

- stream: 它是client与proxy server建立连接后返回的流引用,用来和控制网络中的proxy server通信.Client的Send和Recv方法用的就是stream的send和recv, 是真正收发数据的实体.

  ![image-20211118201036904](https://tva1.sinaimg.cn/large/008i3skNly1gwjka1hid5j30lq0fywfn.jpg)

- address: 通常是proxy server外部负载均衡器的地址.

- stopCh: 用来指示关闭client,就是一个无缓冲Channle.

- conn: 这个是grpc.Dial()方法返回的连接,他的作用:

  用来创建grpc客户端

  通过本身GetState()方法检查连接状态,如果连接不正常,就回Close掉,并在ClientSet中清除自己.

  调用Close()方法关闭gRPC连接



#### 2、连接建立

当通过`newAgentClient`函数创建一个Client时候,Client会通过自己的Connect函数创建到proxy server实例的grpc流.

![image-20211118194536745](https://tva1.sinaimg.cn/large/008i3skNly1gwjjk3f2y9j30ps0p1jtp.jpg)

client在建立连接的过程中(就是调用远程grpc server的connect方法)会把自己的`agentID`和`agentIdentifiers`通过stream的context发送给server, 因为proxy server要在BanckendStorage中存储来自agent的连接,这些相当于标识.

然后在返回的流引用中拿到proxy server返回的`serverID`和`serverCount`.(proxy server会在header中返回serverID和serverCount),并把serverCount返回,以供ClientSet判断是否还需要建立连接,具体过程在ClientSet讲解.至此, 一条连接就建立成功了.

#### 3、开始代理

连接建立后,开始真正的代理来自proxy server的数据,这些都是Serve()函数完成的.

该函数不断的Recv()数据包,然后根据数据包的类型去做相应的处理.

![image-20211118204823545](https://tva1.sinaimg.cn/large/008i3skNly1gwjldcpyprj30si0jyabh.jpg)

接收的数据包一共有三种类型:`DIAL_REQ`、`DATA `、`CLOSE_REQ`,分别对应来自proxy server的拨号请求、数据包、关闭请求.

- DialRequst: 当收到此数据包时,client会根据数据包中的address和protocol来建立与后端服务的tcp连接,然后把连接交给自己的connManager管理, 并将connID返回给proxy server,由它来管理.

  ![image-20211119102328457](https://tva1.sinaimg.cn/large/008i3skNly1gwk8xeiu3sj30v00opacn.jpg)

  connContext中的cleanFunc是关闭与后端tcp的连接,然后做相应的清除工作: 从connManager中移除此connContext, 发送给proxy server CloseResponse数据包.

  ![image-20211119105639807](https://tva1.sinaimg.cn/large/008i3skNly1gwk9vxjv5vj30p30eh3zj.jpg)

  当与后端服务连接建立完毕后,开起两个协程,remoteToProxy是从tcp连接中读取数据,然后返回给proxy server.

  ![image-20211119103418728](https://tva1.sinaimg.cn/large/008i3skNly1gwk98ocqbnj30wm0fygn2.jpg)

  proxyToRemote的作用正好相反, 它从connContext中的dataCh字段中读取数据,然后写入tcp连接中,因为agent从gRPC流中读取的数据都会存在connContext中的dataCh字段中.

  ![image-20211119103456876](https://tva1.sinaimg.cn/large/008i3skNly1gwk99c5e1qj310z0d1gmr.jpg)

  这里写入的时候需要注意一点: 如果没有返回错误,证明成功写入, 如果有错误但是返回成功写入的字节数`0< n < len(d)`证明只成功写入了部分数据,需要重新写入剩下的部分.

  不得不说这两个名字起的很有歧义,我觉得应该是proxyToLocal和localToProxy.

- Data: 如果是正常的Data数据包, 就从connManager中拿到对应connID的connContext,把数据写入context的缓存dataCh字段中.然后由上文提到的proxyToRemote函数消费.
- CloseRequset: 如果是关闭与后端服务连接的请求数据包,从connManager中拿到响应connID的connContext,执行connContext的cleanup函数进行清理.



#### 4、连接关闭

一个Client里面维护着多种连接: 一个gRPC连接,和多个与后端服务的TCP连接.Client的Close()方法会关闭gRPC连接,并关闭stopCh通道,使得监听这个通道的协程关闭.

![image-20211119133352763](https://tva1.sinaimg.cn/large/008i3skNly1gwkefifkesj30ke07x0t7.jpg)

client所有功能的开启是通过Serve()函数,此函数开启了三个协程:probe、proxyToRemote、remoteToproxy.其中probe函数是周期性的检测gRPC连接的状态,如果连接不正常,就回调用ClientSet的RemoveClient函数,该函数就是调用Client的Close函数并把Client从ClientSet中清除掉.

当Serve()函数收到stopCh信号后,也会执行defer退出函数,首先会遍历connManager中的TCP连接,执行cleanup函数把这些连接清除掉,返回给proxy server CloseRespne信息,然后调用ClientSet的RemoveClient函数,将client彻底清除掉.

![image-20211119134247068](https://tva1.sinaimg.cn/large/008i3skNly1gwkeosbcj2j30x2091my2.jpg)

所以一旦调用Client的Close()函数,那么所有的gRPC和TCP连接都会清除掉.同时也会在ClientSet中清除掉自己.

### ClientSet

ClientSet包含一组client集合.是逻辑意义上的一个agent客户端.

<font color=red>为了保证proxy server的高可用, 一般会部署多个server副本, 每个副本有一个具体的serverID, 然后通过一个负载均衡器管理.而Client是与某个具体server通信的实例,通过上面结构体也可以看到它里面是有一个与某个具体server创建连接后返回的流引用和grpc.ClientConn.而ClientSet可以理解为是逻辑上的一个agent,因为一个agent要与所有的server副本建立连接,也就会有多个Client实例,所以ClientSet就是所有这些连接的集合.</font>

```go
type ClientSet struct {
	mu      sync.Mutex         //protects the clients.
	clients map[string]*Client // map between serverID and the client
	// connects to this server.

	agentID     string // ID of this agent
	address     string // proxy server address. Assuming HA proxy server
	serverCount int    // number of proxy server instances, should be 1
	// unless it is an HA server. Initialized when the ClientSet creates
	// the first client.
	syncInterval time.Duration // The interval by which the agent
	// periodically checks that it has connections to all instances of the
	// proxy server.
	probeInterval time.Duration // The interval by which the agent
	// periodically checks if its connections to the proxy server is ready.
	syncIntervalCap time.Duration // The maximum interval
	// for the syncInterval to back off to when unable to connect to the proxy server

	dialOptions []grpc.DialOption
	// file path contains service account token
	serviceAccountTokenPath string
	// channel to signal shutting down the client set. Primarily for test.
	stopCh <-chan struct{}

	agentIdentifiers string // The identifiers of the agent, which will be used
	// by the server when choosing agent

	warnOnChannelLimit bool
}
```

三个time.Duration作用:

- syncInterval: 定期检查它是否与所有server实例建立连接的间隔
- probeInterval: 定期检查与所有server实例连接是否就绪的间隔
- syncIntervalCap: 当无法连接到server时，syncInterval 回退的最大间隔

其他字段:

- agentIdentifiers: 是proxy agent的身份标识, 被proxy server用来选择proxy agent.一共有6种身份标识

  ```go
  const (
  	IPv4         IdentifierType = "ipv4"
  	IPv6         IdentifierType = "ipv6"
  	Host         IdentifierType = "host"
  	CIDR         IdentifierType = "cidr"
  	UID          IdentifierType = "uid"
  	DefaultRoute IdentifierType = "default-route"
  )
  ```

我们使用proxy agent功能实际上就是建立一个ClientSet实例,然后调用它的Serve()方法就可以了.Serve方法会开启一个协程,开始进行连接建立的同步过程.

```go
func (cs *ClientSet) Serve() {
	go cs.sync()
}
```

![image-20211119112635677](https://tva1.sinaimg.cn/large/008i3skNly1gwki0eudjvj30ku0flmy2.jpg)

这个函数不断的调用syncOnce()函数,并监听退出信号.如果收到退出信号会调用shutdown方法,该方法会清除所有存储的Client实例.

`注意ClientSet的退出信号是外部传进来的,和Client使用的退出信号不是同一个.`

所以这个方法的核心功能就是syncOnce()函数的功能.

![image-20211119135425008](https://tva1.sinaimg.cn/large/008i3skNly1gwkmesm15wj30wp0go404.jpg)

一次同步过程所做的工作如下:

- 检查已经建立的Client数量和serverCount字段的对比.如果数量不够,证明还没有与所有的proxy server建立连接,此时就会new一个新的Client去connect负载均衡器的地址,负载均衡器会随机选择一个proxy server副本建立连接,并返回serverID和集群中的serverCount数量.
- 调用AddClient把建立的Client实例纳管进来.因为是随机选择的proxy server副本,所以很有可能已经存在Client实例与它建立了连接,AddClient会去做检查,如果已经存在,会返回错误,此时就调用Client的Close函数清除掉gRPC连接.
- 如果纳管成功,调用Client的Serve()函数开始工作.

ClientSet会不断检查serverCount字段和已经创建的Client数量,必要时去创建连接,开启服务,直到收到外部终止信号后停止服务.

