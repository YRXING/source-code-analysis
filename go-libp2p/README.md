# go-libp2p 源码解析

## 介绍

libp2p就是帮助你链接节点的一个库。它的特性是什么？就是任意两个节点，不管在哪里，不管处于什么环境，不管运行什么操作系统，不管是不是在NAT之后，只要他们有物理上链接的可能性，那么libp2p就会帮你完成这个链接。libp2p使用了*multiaddr*，一个自描述的地址形式，可以理解为不同协议不同地址类型的一个封装。这使得libp2p可以不透明的处理系统中的所有地址，支持网络层中的各种传输协议。

### Multiaddr

Multiaddr 旨在使网络地址面向未来、可组合且高效。

- 支持任何网络协议的地址
- 自描述的
- 符合简单的语法，使它们易于解析和构建
- 具有人类可读和高效的机器可读表示
- 封装的很好，允许包装和解开



```go
import ma "github.com/multiformats/go-multiaddr"

// construct from a string (err signals parse failure)
m1, err := ma.NewMultiaddr("/ip4/127.0.0.1/udp/1234")

// construct from bytes (err signals parse failure)
m2, err := ma.NewMultiaddrBytes(m1.Bytes())

// true
strings.Equal(m1.String(), "/ip4/127.0.0.1/udp/1234")
strings.Equal(m1.String(), m2.String())
bytes.Equal(m1.Bytes(), m2.Bytes())
m1.Equal(m2)
m2.Equal(m1)

// get the multiaddr protocol description objects
m1.Protocols()
// []Protocol{
//   Protocol{ Code: 4, Name: 'ip4', Size: 32},
//   Protocol{ Code: 17, Name: 'udp', Size: 16},
// }

m, err := ma.NewMultiaddr("/ip4/127.0.0.1/udp/1234")
// <Multiaddr /ip4/127.0.0.1/udp/1234>

sctpMA, err := ma.NewMultiaddr("/sctp/5678")

m.Encapsulate(sctpMA)
// <Multiaddr /ip4/127.0.0.1/udp/1234/sctp/5678>

udpMA, err := ma.NewMultiaddr("/udp/1234")

m.Decapsulate(udpMA) // up to + inc last occurrence of subaddr
// <Multiaddr /ip4/127.0.0.1>

//Multiaddr allows expressing tunnels very nicely.
printer, _ := ma.NewMultiaddr("/ip4/192.168.0.13/tcp/80")
proxy, _ := ma.NewMultiaddr("/ip4/10.20.30.40/tcp/443")
printerOverProxy := proxy.Encapsulate(printer)
// /ip4/10.20.30.40/tcp/443/ip4/192.168.0.13/tcp/80

proxyAgain := printerOverProxy.Decapsulate(printer)
// /ip4/10.20.30.40/tcp/443
```



libp2p还是一个工具库，这些工具的功能主要包括链接复用、ID交换、中继、NAT穿越、dht发现、RTT统计等。

![img](http://ipfs-file.oss-cn-shenzhen.aliyuncs.com/0/2018-11/07150051664-896e7abd-9c93-468e-b8c8-38b6761a92d9)



## 核心组件

![未命名文件-2](https://tva1.sinaimg.cn/large/008i3skNly1gxfyqjecygj30qh0h2mz9.jpg)

核心接口定义在[go-libp2p-core](https://github.com/YRXING/source-code-analysis/tree/main/go-libp2p/go-libp2p-core)里面。

### transport

它在应用层和传输层之间，作用类似于socket，之所以封装它一是现在世界上流行的传输协议可能只有tcp、ws等这些，但协议是不断演进的。 二是我们平常去找ip地址，有个弊端就是没法指定协议。所以这个东西的核心就是把各个传输层全部抽象为一个统一的接口，只要匹配好接口就可以了。这也是为什么我们说开发p2p网络不用再关注底层传输协议的原因。

#### upgrader

传统的https协议中，底层是tcp，上面加了一个加密套接字层，其实这个加密套接字层就是upgrader，但是libp2p中，它的功能更多了一些，大概有四层。

![未命名文件-3](https://tva1.sinaimg.cn/large/008i3skNly1gxgllml6ggj30ez04oglq.jpg)

filter upgrader：这一层就是判断一个地址在不在我的黑白名单里面。

protector upgrader：这一层叫保护网络，也叫私有网络。因为它的运行原理就是你如果建了一个分布式应用，如果在私网下建，首先你要生成一个密钥，然后把这个密钥分发到所有的节点中，在每一个节点在初始化的时候，会用这个密钥对链接进行设定，设定了以后，在通信的时候，就会先互换一个随机数，再用这个随机数以及这个密钥来加密整个的传输信道。

secure upgrader：就是类似TLS的加密链接层。这一层目前使用的加密方式有两种，一个是对称加密，一个是非对称加密，非对称加密用来握手，对称加密用来加密信道。

Mux upgrader：链路复用器。顾名思义就是复用了同一个链路，在一个链路上面可以打开多个链接。

#### relay

![未命名文件-4](https://tva1.sinaimg.cn/large/008i3skNly1gxgmc76v61j30fc09wgm7.jpg)

中继transport，它对中国的部分场景还是非常有用的，主要是因为涉及到NAT的问题。中继的实现方案是这样一个框架图，底层是tcp链接，再监听，然后新的tcp链接进来并进行升级，得到了中继的一个流，如果中继是代理到这里就结束了。

如果是末端节点，那么它把这个stream又抽象成了一个链接的概念，进而在监听器上面起了一个新的链接，叫中继链接，对中继链接进行升级，得到一个stream，就到业务逻辑了。中继很有意思的一点，不管中继listener监听到了多少物理链接，底层对应的都是一个物理链接，所以中继场景下的链接都是轻量级的。

### peerinfo

代表了一个节点的相关信息，它包含的内容就两点，一个是ID，通过公钥，经过一次哈希得到的，也是一个节点的唯一标志。另外就是地址信息，这个ID的具体地址是什么，包含3份信息，监听地址、观测地址、NAT映射地址。

### peerstore

类似于我们传统生活中的电话本、通信地址薄，会记录所有你知道的人的相关信息，比如说key book会记录一些公私钥的信息，metrics会记录链接的耗时时间，通过加权平均值的方式对这个节点进行评估。addr book就是地址的一个信息，默认的实现里面是一个带超时的地址信息，当然这个超时时间可以设为零。最后一个data store就是给地址打标签的意思。

### swarm

swarm可以说是libp2p的一个核心，因为它才是真正的网络层。所有跟网络相关的东西全部在swarm里面，地址簿、链接、监听器都在这管理，它得有一个回调机制。让上层注册，当有一个新的stream进来以后，要调用一个相当于中转函数的概念。transport管理是说一个节点可以支持多种transport，dialer就是拨号器，不过它实际上有点复杂，它里面有三种拨号器，一个是同步拨号器，一个是后台限制拨号器，一个是有限制的拨号器，这三个拨号器会共同作用于整个拨号的过程。

然后讲一下NAT，这块可能大家都会比较关注，从目前实现方式来讲，NAT叫穿越不如叫映射。当下它的一个实现方式，就是我启动一个节点，先检查一下我附近的NAT设备运行的是什么协议，再获取到NAT的一个设备地址，然后对这个监听地址进行周期性的端口映射。但很可惜的是，目前只限于两种协议（Upnp、NAT-pmp）且在仅有一层NAT的时候有可能成功。美国的情况和中国的差异非常大，在美国，普通人的手机都能拿到公网ip地址。

### host

![未命名文件-5](https://tva1.sinaimg.cn/large/008i3skNly1gxgnu2v2f2j30kj04y0sv.jpg)

host，是我们操作的核心，一个host底下有这么几个部分组成，首先network，ID service就是我们身身份互换的一个功能，然后nat-manager就是刚刚说的映射，conn-manager是链接管理。



## go-libp2p源码

官方称这个package为entry point，也就是使用这套工具的入口点，有的核心实现分散在不同的package里面，下面是他的项目结构：

![image-20211217141020242](https://tva1.sinaimg.cn/large/008i3skNly1gxgsu35auoj306v0anjrk.jpg)

主要就三个目录，config、examples、p2p。

### examples

这个目录主要展示了几个使用示例程序，可以更快速的上手开发自己所需要的应用程序。对大多数应用来说，host是需要开始使用的基本构建块。host是一种抽象，可以在群集智商管理服务，它提供了一个干净的界面来连接指定的远程节点上的服务。

在下面的代码片段中，我们生成自己的ID并指定我们想要监听的地址：

```go
// Set your own keypair  
// 配置自身的密钥对
priv, _, err := crypto.GenerateEd25519Key(rand.Reader)
if err != nil {
	panic(err)
}

h2, err := libp2p.New(ctx,
	// Use your own created keypair  
	// 使用自身创建的密钥对
	libp2p.Identity(priv),

	// Set your own listen address
	// The config takes an array of addresses, specify as many as you want.  
	// 配置自身的监听地址
	// 该配置采用地址数组的形式，想指定多少就可指定多少
	libp2p.ListenAddrStrings("/ip4/0.0.0.0/tcp/9000"),
)
if err != nil {
	panic(err)
}

fmt.Printf("Hello World, my second hosts ID is %s\n", h2.ID())
```

可以通过传递不同的`Option`给New方法来配置不同的host，从而构建我们所需要的p2p应用。











```go
func New(opts ...Option) (host.Host, error) {
	return NewWithoutDefaults(append(opts, FallbackDefaults)...)
}
```

FallbackDefaults提供一些默认选项：

- 如果没有提供transport和监听地址，node将会监听"/ip4/0.0.0.0/tcp/0" 和 "/ip6/::/tcp/0"并使用TCP和websocket作为传输协议。

- 如果没有提供multiplexer，node将会使用"yamux/1.0.0" 和 "mplux/6.7.0" stream connection multiplexers。
- 如果没有提供安全的transport，host会使用tls来加密所有流量。
- 如果没有提供peer identity，它将产生随机的RSA密钥对来生成identity。
- 如果没有提供peerstore，host将会初始化一个空的peerstore。

![截屏2021-12-16 下午3.21.17](https://tva1.sinaimg.cn/large/008i3skNly1gxfp9prx7hj30mr02swei.jpg)

Option是用来组装Config的。

在options.go文件中包含了所有提供libp2p配置选项的方法，一些默认的选项在default.go中。

- ListenAddrString(s ...string) Option ：该Option将为解析过的字符串形式的地址添加到config中的ListenAddr字段中。相对应的还有ListenAddr(addrs ...ma.Multiaddr) Option，这个函数产生的Option是使用以解析过的地址。

- Security(name string, tpt interface{}) Option ：填充Config的SecurityTransports字段，该字段用来构造安全的transport，这里的名字是protocol name，与之相对应的有NoSecurity。

- Muxer(name string, tpt interface{}) Option：逻辑和Security一样，只不过填充的是Muxers字段。

  ![image-20211216155720914](https://tva1.sinaimg.cn/large/008i3skNly1gxfqb41irjj30qc07ydgc.jpg)



其他的方法有Transport、Peerstore、PrivateNetwork等，每一个Config字段都有一个对应的方法来生成Option去填充它。

而在default.go文件中提供了一些默认Option，比如DefualtSecurity、DefaultMuxers等。







核心的创建逻辑在config结构体的NewNode方法。



