# hysteria-




关于hysteria，实际上就是自定义了quic中的一种阻塞控制方式

可以查看 pkg/core/client.go 和 pkg/core/server.go

server文件较短，client文件较长

server.go 主要是用了一个 CongestionFactory, 唯一用到的地方是 Server.handleControlStream

NewClient 函数 和 NewServer 函数 有传入 这个factory参数

在 cmd/server.go 里的 server 函数中，可以看到该函数很简单，就是

```
func(refBPS uint64) congestion.CongestionControl {
    return hyCongestion.NewBrutalSender(congestion.ByteCount(refBPS))
}
```

而 ByteCount 只是 int64的别名而已

(cmd/client.go 里的是一样的）



而这个 CongestionFactory还是在 client.go 里定义的

而这个唯一用到的地方是 Client.handleControlStream


仔细观察，发现它在 server的 AcceptStream之后， 和 client的 OpenStreamSync

被调用

本质上就是一次连接的自定义数据头

第一字节要求不是3就是2，

其余要求用 struc 包来序列化一个 clientHello 结构
```
type clientHello struct {
    Rate    transmissionRate
    AuthLen uint16 `struc:"sizeof=Auth"`
    Auth    []byte
}

type transmissionRate struct {
    SendBPS uint64
    RecvBPS uint64
}
```
总之 就是协商 serverSendBPS和 serverRecvBPS， 然后进行auth身份验证

没什么特殊的

关键是最后调用了
session.SetCongestionControl(s.congestionFactory(serverSendBPS))



 
原来 SetCongestionControl(congestion.CongestionControl) 中的 congestion.CongestionControl 是一个接口

里面定义了大量方法，而 CongestionFactory 是一个 函数，传入 serverSendBPS 就会生成一个 该接口的实现



在
pkg/congestion/



另外发现的比较诡异的事情是

brutal.go 和 pacer.go 引用了
github.com/lucas-clemente/quic-go/congestion
包，但是该包根本不存在

仔细考察，发现go.mod里有这么一句

```
 replace github.com/lucas-clemente/quic-go => github.com/tobyxdd/quic-go v0.25.1-0.20220224051149-310bd1bfaf1f
```

然后 tobyxdd的 quic-go包里有一个 congestion/interface.go 文件，里面包含了很多定义。

有必要这样吗？？

主要是 它用了 quic-go 包中的 internal/protocol 里的 protocol.ByteCount
和 protocol.PacketNumber
而这两个是internal里的所以无法随意引用


总之比较奇葩


另外就是发现 quic-go包的 session 也是没有 SetCongestionControl 方法的，这个一样是 hysteria自己改的


实际上根据quic的标准，阻塞控制是可以自定义的，因此实际上 toby的quic 包更加科学

如此的话，我们直接引用 toby的quic-go 包即可

但是，我们直接引用的话又出错，显示
```
go: github.com/tobyxdd/quic-go@v0.25.0: parsing go.mod:
	module declares its path as: github.com/lucas-clemente/quic-go
	        but was required as: github.com/tobyxdd/quic-go
```

也就是说toby这个包压根就没打算让别人去引用，所以连path都没改

那么我们自己fork一个即可

结果fork了还不够，因为里面引用了自己的大量的 internal包，所以还真只能用 replace的方法，好坑啊

总之，我们如果要使用hysteria的自定义阻控的方法，那就要 replace

这只是一种临时的方法。根据quic标准，正常的quic包应该会提供自定义阻控的方式，而lucas的包却没提供，总之它确实还是不够标准化。

标准的 QUIC 的拥塞控制算法设计为可插拔模块，显然lucas的不行

实际上hysteria的作者toby也提过这一部分，但是因为某些原因无法被merge

https://github.com/lucas-clemente/quic-go/issues/776

https://github.com/lucas-clemente/quic-go/pull/2442




# 具体的hysteria 阻塞控制机制

关注 brutal.go 和 pacer.go

如果能修改这两个文件，就能试图改进冷启动问题

里面使用了token bucket pacing 令牌桶算法

令牌桶算法的原理是系统会以一个恒定的速度往桶里放入令牌，而如果请求需要被处理，则需要先从桶里获取一个令牌，当桶里没有令牌可取时，则拒绝服务，令牌桶算法通过发放令牌，根据令牌的发放频率做请求频率限制，容量限制等。

总之先学一下再说。。。


这个pacer是从lucas官方的包里拷贝出来然后魔改的

https://github.com/lucas-clemente/quic-go/blob/master/internal/congestion/pacer.go


brutal文件则是来自 官方的 cubic_sender.go

https://github.com/lucas-clemente/quic-go/blob/63b7354a25ea94d082d1ff551d77bbdbf28b2295/internal/congestion/cubic_sender.go


我们要做的，和 研究xtls类似，把官方 文件下载下来，然后通过diff命令比较与魔改版的异同。

## pacer.go

作者对pacer的改动较少，先分析pacer.go

const部分，把 `const maxBurstSizePackets = 10` 删了，改成了 

```
const (
	maxBurstPackets = 10
	minPacingDelay  = time.Millisecond
)

```
就是 10那个改了个名，然后加了个 minPacingDelay 为1毫秒

pacer结构体没有变化，只是getAdjustedBandwidth改个名改成了 getBandwidth 

newPacer函数的参数中的函数的 签名变化了一下，但是实际上没差别，底层都是 int64

（即 `func newPacer(getBandwidth func() Bandwidth)` 改成了 `func newPacer(getBandwidth func() congestion.ByteCount)` ）

所以结构体里成员的改名很合理

然后里面的 `p.budgetAtLastSent = p.maxBurstSize()` 改成了 `budgetAtLastSent: maxBurstPackets * initMaxDatagramSize,`

而 `initMaxDatagramSize = 1252` 是在 brutal.go 里定义的，这里确实变化较大，但是影响应该不大，因为只是 初始条件改变了一丢丢

然后 `maxBurstSize()` 改动了，


