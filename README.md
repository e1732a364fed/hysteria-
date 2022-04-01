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

这里贴出来对比一下：

原作：
```
return utils.MaxByteCount(
	protocol.ByteCount(uint64((protocol.MinPacingDelay+protocol.TimerGranularity).Nanoseconds())*p.getAdjustedBandwidth())/1e9,
	maxBurstSizePackets*p.maxDatagramSize,
)
```
hysteria:

```
return maxByteCount(
	congestion.ByteCount((minPacingDelay+time.Millisecond).Nanoseconds())*p.getBandwidth()/1e9,
	maxBurstPackets*p.maxDatagramSize,
)
```

但是实际观察发现都是一回事，作者还是改了个名之类的。

然后是 TimeUntilSend方法，仔细看好像还是只是改了点名。无所谓


## brutal.go

实际上，cubic是单独的一种阻控，应该是早被用于tcp的；而quic默认也是用这种方式，但是hysteria这里就改了，用了一种被它成为brutal的方式，那么我们看一看。


实际上感性认识就知道，只是二者的接口一致罢了，实际实现肯定不同

我们看BrutalSender结构体，非常简单，对应官方的 cubicSender 结构体


```
type BrutalSender struct {
	rttStats        congestion.RTTStatsProvider
	bps             congestion.ByteCount
	maxDatagramSize congestion.ByteCount
	pacer           *pacer

	pktInfoSlots [pktInfoSlotCount]pktInfo
}
```
它保留了 cubicSender的 rttStats 和 pacer， 然后多了两个储存 配置值的地方，

然后关键是这个数组 `pktInfoSlots [pktInfoSlotCount]pktInfo`


```
type pktInfo struct {
	Timestamp int64
	AckCount  uint64
	LossCount uint64
}
```

pktInfoSlotCount为定值4

然后它为BrutalSender实现了下列方法

```
SetRTTStatsProvider
TimeUntilSend
HasPacingBudget
CanSend
GetCongestionWindow
OnPacketSent
OnPacketAcked
OnPacketLost
SetMaxDatagramSize
getAckRate
InSlowStart
InRecovery
MaybeExitSlowStart
OnRetransmissionTimeout

```

其中，SetRTTStatsProvider， getAckRate， 这两个方法是独有的，其它方法都是实现接口

然后， InSlowStart， InRecovery都直接返回false，而 cubic则不是，cubic是视情况而定的。


然后，我们直观的去看代码量，发现 brutal的 `OnPacketAcked` 和 `OnPacketLost` 比较长，肯定重要，从这里开始分析

肯定推测得知，OnPacketAcked 就是每收到一个包调用一次该函数，OnPacketLost同理

这里 OnPacketAcked 就是生成当前 时间的时间戳，然后更新 `b.pktInfoSlots[slot]`。那么这个 pktInfoSlots显然是一个 ring buffer

单单 OnPacketAcked 好像没啥用？查看 OnPacketLost，发现是类似的过程，只是把 `AckCount++` 变成了 `LossCount++`

似乎也没啥用啊？光记录有啥用。那么我们搜索到底谁还用了 pktInfoSlots，发现 `getAckRate` 方法用到了

它通过计算 ackCount 和 lossCount 的到rate，具体是 `rate := float64(ackCount) / float64(ackCount+lossCount)`

然后再搜就知道了，bs.pacer 以及 GetCongestionWindow 方法都用到了 getAckRate 方法

pacer的定义是
```
bs.pacer = newPacer(func() congestion.ByteCount {
	return congestion.ByteCount(float64(bs.bps) / bs.getAckRate())
})
```

GetCongestionWindow 是
```
func (b *BrutalSender) GetCongestionWindow() congestion.ByteCount {
	rtt := maxDuration(b.rttStats.LatestRTT(), b.rttStats.SmoothedRTT())
	if rtt <= 0 {
		return 10240
	}
	return congestion.ByteCount(float64(b.bps) * rtt.Seconds() * 1.5 / b.getAckRate())
}
```

对比 cubicSender的，它的pacer的函数是 BandwidthEstimate，暂时先不管；而 GetCongestionWindow直接是返回 c.congestionWindow，而这个window是在其他步骤经过复杂计算的，也暂时不用管。

在看pacer，pacer 在 

```
TimeUntilSend
HasPacingBudget
OnPacketSent
SetMaxDatagramSize
```
里用到了

而对比 cubicSender的，TimeUntilSend，HasPacingBudget 的代码完全一样，OnPacketSent的话 cubicSender的多了两行，被brutal的删掉了

而 SetMaxDatagramSize 也是一样，brutal的简单，删掉了一些cubicSender多多代码。

总之，目前直观解释，就是通过 GetCongestionWindow 方法被不断地调用，quic就获知了 需要的window大小，实际上就应该是每次发送包的最大长度吧。

实际上我们看到的这些代码和BBR类似。

我们可以看到，rttStats 挺重要，它提供了rtt的值，

RTT: https://www.cloudflare.com/learning/cdn/glossary/round-trip-time-rtt/

>Round-trip time (RTT) is the duration in milliseconds (ms) it takes for a network request to go from a starting point to a destination and back again to the starting point. RTT is an important metric in determining the health of a connection on a local network or the larger Internet, and is commonly utilized by network administrators to diagnose the speed and reliability of network connections.


实际上，lucas的cubic代码应该是来自 谷歌官方的quic实现

https://chromium.googlesource.com/external/github.com/google/proto-quic/+/4cc12d2679be1baceb162a7fde89b7183901fe84/src/net/quic/congestion_control/tcp_cubic_sender_bytes.cc


关于阻塞窗口：

>In TCP, the congestion window (CWND) is one of the factors that determines the number of bytes that can be sent out at any time.

总之，brutal似乎没啥特殊的，可以说是 “低等” 阻塞控制协议，用了特别简单的算法来简单 用加减乘除算出一个 阻塞窗口，而不是cubic这种更科学的方式。


它的优势就是，初始值就是配置的值，上来就用配置的最高速发包，然后发现丢包严重后，再线性降低发包速度，如果只能收到50%的包，那么就按50%的初始设置的值的速度进行发包。

TimeUntilSend的解释：

TimeUntilSend returns when the next packet should be sent. It returns the zero value of time.Time if a packet can be sent immediately.

总之quic也会调用 TimeUntilSend，先不管。

brutal的pacer也是一种线性的，十分简单的pacer，而不是 cubicSender中的复杂实现。


我们可以阅读一些中文文章来了解一下

https://hanpfei.github.io/2019/08/29/draft-ietf-quic-recovery/



pacer大概就是在 TimeUntilSend 应用，控制发包的节奏

>在网络中不加任何延迟地发送多个数据包会导致数据包爆发，这可能会造成短期的拥塞和丢包。实现必须（MUST）使用 pacing

这也可以解释为什么直接用udp暴力发包会导致高概率丢包，（我的实际测试中可能丢了99%，只有1%能发到。。。）

## 小总结

 根据上述分析，hy上来就以之前设置的mbps的1.5倍速度发包，然后慢慢发现网络不好后升高发包频率，最高升至无穷大。（如果rate是0.0000001，那么就按10的七次方乘1.5的速度发送）
 
 仔细一想就是慢启动啊！关键在于，因为网太差，导致发送方很慢才会收到 接受方 ”重新请求“ 的信号，才会知道 网不好，rate才会变小，然后发送速率才会提高

但是 有个 minAckRate，限制了rate最小值，没法到0.000001，目前hy 的设置是0.75，也就是说，最大就是 3/2 / (3/4) 就是2倍发包
