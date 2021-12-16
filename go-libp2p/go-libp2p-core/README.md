# go-libp2p-core package

这个包里面定义和实现了核心的接口和类型，而具体的实现分散在不同的包里面，比如Network相关实现在`go-libp2p-swarm`包里。这里按照`go-libp2p-core`包的目录结构来介绍一些主要的数据结构和接口。

### Data Type

#### peer

peer代表了p2p通信中的一个对等体，由一个ID和地址表示。

ID：每一个peer用一个ID来标识，这个ID是对peer的公钥做哈希并编码后得到的。

```go
type ID string
```

AddrInfo：用于传递具有一组地址的peer。`这里的地址采用的是p2p格式的Multiaddr，它是一种跨协议、跨平台的格式，用于表示internet地址，具有二进制和字符串表示形式，源自于IPFS项目。他的格式像这样：/ip4/127.0.0.1/tcp/1234/p2p/QmS3zcG7LhYZYSJMhyRZvTddvbNUqtt8BJpaSs6mi1K5Va`

```go
type AddrInfo struct {
	ID    ID
	Addrs []ma.Multiaddr
}

addr := AddrInfo {
  ID: QmS3zcG7LhYZYSJMhyRZvTddvbNUqtt8BJpaSs6mi1K5Va,
  Addrs: []ma.Multiaddr{"/ip4/127.0.0.1/tcp/1234",},
}
```

PeerRecord：目前，PeerRecord 包含对等方的公共侦听地址，但预计将来会扩展到包括其他信息。

```go
type PeerRecord struct {
	// PeerID is the ID of the peer this record pertains to.
	PeerID ID

	// Addrs contains the public addresses of the peer this record pertains to.
	Addrs []ma.Multiaddr

	// Seq is a monotonically-increasing sequence counter that's used to order
	// PeerRecords in time. The interval between Seq values is unspecified,
	// but newer PeerRecords MUST have a greater Seq value than older records
	// for the same peer.
	Seq uint64
}
```

要共享 PeerRecord，首先调用 Sign 将记录包装在 Envelope 中，并使用本地对等方的私钥对其进行签名：

```go
rec := &PeerRecord{PeerID: myPeerId, Addrs: myAddrs}
envelope, err := rec.Sign(myPrivateKey)
// The resulting record.Envelope can be marshalled to a []byte and shared publicly.
recordBytes, err := rec.MarshalSigned(myPrivateKey)
```



Set: 一组线程安全的peers，go语言中没有set结构，经常用map[type]struct{}结构来实现type类型的set集合。

```go
type Set struct {
	lk sync.RWMutex
	ps map[ID]struct{}

	size int
}
```

Peerstore: 提供Peer相关信息的线程安全存储。比如地址、公私钥、支持的协议等。

```go
type Peerstore interface {
	io.Closer

	AddrBook
	KeyBook
	PeerMetadata
	Metrics
	ProtoBook

	// PeerInfo returns a peer.PeerInfo struct for given peer.ID.
	// This is a small slice of the information Peerstore has on
	// that peer, useful to other services.
	PeerInfo(peer.ID) peer.AddrInfo

	// Peers returns all of the peer IDs stored across all inner stores.
	Peers() peer.IDSlice
}
```



#### network

Conn：表示到远程peer的连接，它复用stream。通常不需要直接使用 Conn，但获取另一端peer的信息可能很有用：stream.Conn().RemotePeer()。

```go
type Conn interface {
	io.Closer

	ConnSecurity
	ConnMultiaddrs
	ConnStat

	// ID returns an identifier that uniquely identifies this Conn within this
	// host, during this run. Connection IDs may repeat across restarts.
	ID() string

	// NewStream constructs a new Stream over this conn.
	NewStream(context.Context) (Stream, error)

	// GetStreams returns all open streams over this conn.
	GetStreams() []Stream
}
```

Stream：表示 libp2p 网络中两个代理之间的双向通道。Stream和Conn是n:1的关系，一个Conn可以有多个Stream，一个Stream只属于某个Conn。

```go
type Stream interface {
	mux.MuxedStream

	// ID returns an identifier that uniquely identifies this Stream within this
	// host, during this run. Stream IDs may repeat across restarts.
	ID() string

	Protocol() protocol.ID
	SetProtocol(id protocol.ID)

	// Stat returns metadata pertaining to this stream.
	Stat() Stat

	// Conn returns the connection this stream is part of.
	Conn() Conn
}
```

State: 存储给定Stream/Conn的相关元数据，比如连接方向、打开链接的时间、连接是不是短暂的、额外的信息。

```go
type Stat struct {
	// Direction specifies whether this is an inbound or an outbound connection.
	Direction Direction
	// Opened is the timestamp when this connection was opened.
	Opened time.Time
	// Transient indicates that this connection is transient and may be closed soon.
	Transient bool
	// Extra stores additional metadata about this connection.
	Extra map[interface{}]interface{}
}
```

MuxedStream：含有read、write方法的接口，代表一条连接中双向的io管道。

MuxedConn：表示与远程对等方的连接，该连接已扩展为支持流复用。 MuxedConn 允许单个 net.Conn 连接承载许多逻辑上独立的双向二进制数据流。与 network.ConnSecurity 一起，MuxedConn 是 transport.CapableConn 接口的一个组件，它代表一个“原始”网络连接，该连接已经“升级”以支持安全通信和流复用的 libp2p 功能。

```go
type MuxedConn interface {
	// Close closes the stream muxer and the the underlying net.Conn.
	io.Closer

	// IsClosed returns whether a connection is fully closed, so it can
	// be garbage collected.
	IsClosed() bool

	// OpenStream creates a new stream.
	OpenStream(context.Context) (MuxedStream, error)

	// AcceptStream accepts a stream opened by the other side.
	AcceptStream() (MuxedStream, error)
}
```

Multiplexer: 多路复用器用流复用实现包装了一个 net.Conn 并返回一个 MuxedConn，该 MuxedConn 支持在底层 net.Conn 上打开多个流。

```go
type Multiplexer interface {

	// NewConn constructs a new connection
	NewConn(c net.Conn, isServer bool) (MuxedConn, error)
}
```

Network: 用来连接外部世界的借口，使用Swarm来汇集连接。Dialer用来建立和远端peer的连接。

```go
type Network interface {
	Dialer
	io.Closer

	// SetStreamHandler sets the handler for new streams opened by the
	// remote side. This operation is threadsafe.
	SetStreamHandler(StreamHandler)

	// NewStream returns a new stream to given peer p.
	// If there is no connection to p, attempts to create one.
	NewStream(context.Context, peer.ID) (Stream, error)

	// Listen tells the network to start listening on given multiaddrs.
	Listen(...ma.Multiaddr) error

	// ListenAddresses returns a list of addresses at which this network listens.
	ListenAddresses() []ma.Multiaddr

	// InterfaceListenAddresses returns a list of addresses at which this network
	// listens. It expands "any interface" addresses (/ip4/0.0.0.0, /ip6/::) to
	// use the known local interfaces.
	InterfaceListenAddresses() ([]ma.Multiaddr, error)

	io.Closer
}

type Dialer interface {
	// Peerstore returns the internal peerstore
	// This is useful to tell the dialer about a new address for a peer.
	// Or use one of the public keys found out over the network.
	Peerstore() peerstore.Peerstore

	// LocalPeer returns the local peer associated with this network
	LocalPeer() peer.ID

	// DialPeer establishes a connection to a given peer
	DialPeer(context.Context, peer.ID) (Conn, error)

	// ClosePeer closes the connection to a given peer
	ClosePeer(peer.ID) error

	// Connectedness returns a state signaling connection capabilities
	Connectedness(peer.ID) Connectedness

	// Peers returns the peers connected
	Peers() []peer.ID

	// Conns returns the connections in this Netowrk
	Conns() []Conn

	// ConnsToPeer returns the connections in this Netowrk for given peer.
	ConnsToPeer(p peer.ID) []Conn

	// Notify/StopNotify register and unregister a notifiee for signals
	Notify(Notifiee)
	StopNotify(Notifiee)
}
```

Notifiee：希望从Network中接收通知的接口,NotifyBundle实现了Notifiee接口。

```go
type Notifiee interface {
	Listen(Network, ma.Multiaddr)      // called when network starts listening on an addr
	ListenClose(Network, ma.Multiaddr) // called when network stops listening on an addr
	Connected(Network, Conn)           // called when a connection opened
	Disconnected(Network, Conn)        // called when a connection closed
	OpenedStream(Network, Stream)      // called when a stream opened
	ClosedStream(Network, Stream)      // called when a stream closed

	// TODO
	// PeerConnected(Network, peer.ID)    // called when a peer connected
	// PeerDisconnected(Network, peer.ID) // called when a peer disconnected
}
```



#### transport

CapableConn：提供libp2p所需要的基本能力的一条connection：流复用、加密和认证。

```go
type CapableConn interface {
	mux.MuxedConn
	network.ConnSecurity
	network.ConnMultiaddrs

	// Transport returns the transport to which this connection belongs.
	Transport() Transport
}
```

Transport：代表你可以连接或从其他peer接收connections的任何设备。它里面有Dial和Listen方法，可以让你建立与其他peer的连接（可以判断你提供的地址是否可以建立连接），也可以监听入站连接。其中的Listener类似于net.Listener，唯一的区别就是Accept方法返回的是CapableConn并且公开了MultiAddr方法。

```go
type Transport interface {
	Dial(ctx context.Context, raddr ma.Multiaddr, p peer.ID) (CapableConn, error)
	
	CanDial(addr ma.Multiaddr) bool

	Listen(laddr ma.Multiaddr) (Listener, error)

	// Protocol returns the set of protocols handled by this transport.
	Protocols() []int

	// Proxy returns true if this is a proxy transport.
	Proxy() bool
}
```

TransportNetwork：是Network的升级，用来处理transports，除了Network接口中的方法外，还增加了一个AddTransport方法，用来增加一个transport。

#### connmgr

Decayer: 由支持衰减标签的connection managers实现。衰减标签是其值随时间自动衰减的标签。

ConnManager: 跟踪到peers的连接，并允许消费者将元数据与每个peer相关联。它允许根据实现定义的启发式来修剪连接。 ConnManager 允许 libp2p 对打开的连接总数实施上限。支持衰减标签的 ConnManagers 实现了 Decayer。如果支持，使用 SupportsDecay 函数将实例安全地转换为 Decayer。

```go
type ConnManager interface {
	// TagPeer tags a peer with a string, associating a weight with the tag.
	TagPeer(peer.ID, string, int)

	UntagPeer(p peer.ID, tag string)

	// UpsertTag updates an existing tag or inserts a new one.
	//
	// The connection manager calls the upsert function supplying the current
	// value of the tag (or zero if inexistent). The return value is used as
	// the new value of the tag.
	UpsertTag(p peer.ID, tag string, upsert func(int) int)

	GetTagInfo(p peer.ID) *TagInfo

	// TrimOpenConns terminates open connections based on an implementation-defined
	// heuristic.
	TrimOpenConns(ctx context.Context)

	// Notifee returns an implementation that can be called back to inform of
	// opened and closed connections.
	Notifee() network.Notifiee

	// Protect protects a peer from having its connection(s) pruned.
	//
	// Tagging allows different parts of the system to manage protections without interfering with one another.
	//
	// Calls to Protect() with the same tag are idempotent. They are not refcounted, so after multiple calls
	// to Protect() with the same tag, a single Unprotect() call bearing the same tag will revoke the protection.
	Protect(id peer.ID, tag string)

	// Unprotect removes a protection that may have been placed on a peer, under the specified tag.
	//
	// The return value indicates whether the peer continues to be protected after this call, by way of a different tag.
	// See notes on Protect() for more info.
	Unprotect(id peer.ID, tag string) (protected bool)

	// IsProtected returns true if the peer is protected for some tag; if the tag is the empty string
	// then it will return true if the peer is protected for any tag
	IsProtected(id peer.ID, tag string) (protected bool)

	// Close closes the connection manager and stops background processes.
	Close() error
}

// TagInfo stores metadata associated with a peer.
type TagInfo struct {
	FirstSeen time.Time
	Value     int

	// Tags maps tag ids to the numerical values.
	Tags map[string]int

	// Conns maps connection ids (such as remote multiaddr) to their creation time.
	Conns map[string]time.Time
}
```



#### event

这个目录定义了各种类型的事件，一共有以下几种类型：

- EvtLocalAddressesUpdate事件：当本地主机的监听地址集发生改变时候会发出此事件，比如新增、删除等。

- GenericDHTEvent事件：表示一个DHT事件

- EvtPeerIdentificationCompleted/Failed：表示peer身份识别事件

- EvtNATDeviceTypeChanged：表示NAT设备类型因传输协议改变而产生的事件

- EvtPeerConnectednessChanged：应该在每次与给定peer的“connectedness”发生变化时发出。里面包含发生变化的远程peer的ID以及新的connectedness状态。

  connectedness表示与给定节点的连接容量。它用于向服务和其他对等点发送信号，告知节点是否可访问。分别有NotConnected、Connected、CanConnect、CannotConnect四种。

- EvtPeerProtocolsUpdate/EvtLocalProtocalsUpdate：第一个表示连接的远程peer增加或删除了一些protocol发出的事件，第二个是当stream handlers从本地host attach或detach时候发出的事件。

- EvtLocalReachabilityChanged：当本地节点的reachability属性改变时候发出的。

  reachability表示一个节点的可达性，有Reachability、ReachabilityPublic（公网可达）、ReachabilityPrivate（公网不可达）三种。

消息离不开订阅者，发出者以及消息总线。

Emitter：消息发送者。

```go
type Emitter interface {
	io.Closer

	// Emit emits an event onto the eventbus. If any channel subscribed to the topic is blocked,
	// calls to Emit will block.
	//
	// Calling this function with wrong event type will cause a panic.
	Emit(evt interface{}) error
}
```

Subscription：消息订阅者，可以订阅多种类型的消息。如果想要订阅所有的消息，使用WildcardSubsciption。

```go
type Subscription interface {
	io.Closer

	// Out returns the channel from which to consume events.
	Out() <-chan interface{}
}
```

Bus：基于类型的消息分发系统，具体的方法和使用都在下面的注释里。

```go
type Bus interface {
	// Subscribe creates a new Subscription.
	//
	// eventType can be either a pointer to a single event type, or a slice of pointers to
	// subscribe to multiple event types at once, under a single subscription (and channel).
	//
	// Failing to drain the channel may cause publishers to block.
	//
	// If you want to subscribe to ALL events emitted in the bus, use
	// `WildcardSubscription` as the `eventType`:
	//
	//  eventbus.Subscribe(WildcardSubscription)
	//
	// Simple example
	//
	//  sub, err := eventbus.Subscribe(new(EventType))
	//  defer sub.Close()
	//  for e := range sub.Out() {
	//    event := e.(EventType) // guaranteed safe
	//    [...]
	//  }
	//
	// Multi-type example
	//
	//  sub, err := eventbus.Subscribe([]interface{}{new(EventA), new(EventB)})
	//  defer sub.Close()
	//  for e := range sub.Out() {
	//    select e.(type):
	//      case EventA:
	//        [...]
	//      case EventB:
	//        [...]
	//    }
	//  }
	Subscribe(eventType interface{}, opts ...SubscriptionOpt) (Subscription, error)

	// Emitter creates a new event emitter.
	//
	// eventType accepts typed nil pointers, and uses the type information for wiring purposes.
	//
	// Example:
	//  em, err := eventbus.Emitter(new(EventT))
	//  defer em.Close() // MUST call this after being done with the emitter
	//  em.Emit(EventT{})
	Emitter(eventType interface{}, opts ...EmitterOpt) (Emitter, error)

	// GetAllEventTypes returns all the event types that this bus knows about
	// (having emitters and subscribers). It omits the WildcardSubscription.
	//
	// The caller is guaranteed that this function will only return value types;
	// no pointer types will be returned.
	GetAllEventTypes() []reflect.Type
}
```



#### protocol

Router：允许用户增加或删除protocol handlers，当收到进入的stream请求，Router会检查所有注册进来的handler来决定哪个适合处理这个stream，金叉顺序为注册顺序，当有多个处理器适合的时候只有第一个会被调用。

```go
type Router interface {
	AddHandler(protocol string, handler HandlerFunc)

	// AddHandlerWithFunc registers the given handler to be invoked
	// when the provided match function returns true.
	//
	// The match function will be invoked with an incoming protocol
	// ID string, and should return true if the handler supports
	// the protocol. Note that the protocol ID argument is not
	// used for matching; if you want to match the protocol ID
	// string exactly, you must check for it in your match function.
	AddHandlerWithFunc(protocol string, match func(string) bool, handler HandlerFunc)

	RemoveHandler(protocol string)

	// Protocols returns a list of all registered protocol ID strings.
	// Note that the Router may be able to handle protocol IDs not
	// included in this list if handlers were added with match functions
	// using AddHandlerWithFunc.
	Protocols() []string
}
```

Negotiator：就如入站通信流使用哪种协议达成一致

```go
type Negotiator interface {

	// NegotiateLazy will return the registered protocol handler to use
	// for a given inbound stream, returning as soon as the protocol has been
	// determined. Returns an error if negotiation fails.
	//
	// NegotiateLazy may return before all protocol negotiation responses have been
	// written to the stream. This is in contrast to Negotiate, which will block until
	// the Negotiator is finished with the stream.
	NegotiateLazy(rwc io.ReadWriteCloser) (io.ReadWriteCloser, string, HandlerFunc, error)
	
	Negotiate(rwc io.ReadWriteCloser) (string, HandlerFunc, error)

	// Handle calls Negotiate to determine which protocol handler to use for an
	// inbound stream, then invokes the protocol handler function, passing it
	// the protocol ID and the stream. Returns an error if negotiation fails.
	Handle(rwc io.ReadWriteCloser) error
}
```

Switch：负责将入站流量分派给相应的处理器，它既是一个Negotiator，也是一个Router

```go
type Switch interface {
	Router
	Negotiator
}
```



#### host

host：参与p2p网络的对象，它实现协议或提供服务。它可以像服务器一样处理请求，也可以像客户端一样发出请求。

```go
type Host interface {
	// ID returns the (local) peer.ID associated with this Host
	ID() peer.ID

	// Peerstore returns the Host's repository of Peer Addresses and Keys.
	Peerstore() peerstore.Peerstore

	// Returns the listen addresses of the Host
	Addrs() []ma.Multiaddr

	// Networks returns the Network interface of the Host
	Network() network.Network

	// Mux returns the Mux multiplexing incoming streams to protocol handlers
	Mux() protocol.Switch

	// Connect ensures there is a connection between this host and the peer with
	// given peer.ID. Connect will absorb the addresses in pi into its internal
	// peerstore. If there is not an active connection, Connect will issue a
	// h.Network.Dial, and block until a connection is open, or an error is
	// returned. // TODO: Relay + NAT.
	Connect(ctx context.Context, pi peer.AddrInfo) error

	// SetStreamHandler sets the protocol handler on the Host's Mux.
	// This is equivalent to:
	//   host.Mux().SetHandler(proto, handler)
	// (Threadsafe)
	SetStreamHandler(pid protocol.ID, handler network.StreamHandler)

	// SetStreamHandlerMatch sets the protocol handler on the Host's Mux
	// using a matching function for protocol selection.
	SetStreamHandlerMatch(protocol.ID, func(string) bool, network.StreamHandler)

	// RemoveStreamHandler removes a handler on the mux that was set by
	// SetStreamHandler
	RemoveStreamHandler(pid protocol.ID)

	// NewStream opens a new stream to given peer p, and writes a p2p/protocol
	// header with given ProtocolID. If there is no connection to p, attempts
	// to create one. If ProtocolID is "", writes no header.
	// (Threadsafe)
	NewStream(ctx context.Context, p peer.ID, pids ...protocol.ID) (network.Stream, error)

	// Close shuts down the host, its Network, and services.
	Close() error

	// ConnManager returns this hosts connection manager
	ConnManager() connmgr.ConnManager

	// EventBus returns the hosts eventbus
	EventBus() event.Bus
}
```

