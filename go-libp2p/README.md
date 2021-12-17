# go-libp2p 源码解析

## 介绍

libp2p就是帮助你链接节点的一个库。它的特性是什么？就是任意两个节点，不管在哪里，不管处于什么环境，不管运行什么操作系统，不管是不是在NAT之后，只要他们有物理上链接的可能性，那么libp2p就会帮你完成这个链接。libp2p使用了*multiaddr*，一个自描述的地址形式，可以理解为不同协议不同地址类型的一个封装。这使得libp2p可以不透明的处理系统中的所有地址，支持网络层中的各种传输协议。

libp2p还是一个工具库，这些工具的功能主要包括链接复用、ID交换、中继、NAT穿越、dht发现、RTT统计等。

![img](http://ipfs-file.oss-cn-shenzhen.aliyuncs.com/0/2018-11/07150051664-896e7abd-9c93-468e-b8c8-38b6761a92d9)



## 核心组件

![未命名文件-2](https://tva1.sinaimg.cn/large/008i3skNly1gxfyqjecygj30qh0h2mz9.jpg)

核心接口定义在[go-libp2p-core](https://github.com/YRXING/source-code-analysis/tree/main/go-libp2p/go-libp2p-core)里面。

### transport

它在应用层和传输层之间，作用类似于socket，之所以封装它一是现在世界上流行的传输协议可能只有tcp、ws等这些，但协议是不断演进的。 二是我们平常去找ip地址，有个弊端就是没法指定协议。所以这个东西的核心就是把各个传输层全部抽象为一个统一的接口，只要匹配好接口就可以了。这也是为什么我们说开发p2p网络不用再关注底层传输协议的原因。

### upgrader

传统的https协议中，底层是tcp，上面加了一个加密套接字层，其实这个加密套接字层就是upgrader，但是libp2p中，它的功能更多了一些，大概有四层。









go-libp2p包的作用主要是创建一个p2p host。

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



