# xudp小小思考

## 关于v2ray 的 udp fullcone 问题

https://github.com/v2ray/v2ray-core/issues/2616

https://github.com/v2ray/v2ray-core/issues/1429



## 关于xray的fullcone的实现

https://github.com/XTLS/Xray-core/discussions/237

https://github.com/XTLS/Xray-core/discussions/252

它利用了mux，然后新写了个 xudp的代码

https://github.com/XTLS/Xray-core/search?q=xudp

xudp到底什么原理？

rprx说：
>扩展 Mux.Cool 协议 Keep 的元数据，使其像 New 一样带上 UDP 端口和地址信息，为方便交流，扩展后的协议命名为 XUDP。


## 代码分析

那么我们看看它的代码实现

### xudp

首先看xudp的代码 https://github.com/XTLS/Xray-core/blob/bf94fb53caf61244abeb031a6088566290702a0d/common/xudp/xudp.go

观察这个方法：WriteMultiBuffer

在第54行前，前面先写了4个字节的0，然后对 新旧数据分别使用New和 Keep格式，但是注意

而55-60行是什么意思？56、57行 是把消息的头部长度存放到1、2字节中，58、59行 是把承载数据的长度放到 请求头末尾，然后把数据写在后面。

似乎还是没有特殊的东东。


然后观察 ReadMultiBuffer 方法

首先l 是前两字节，是整个数据的长度，然后把剩余部分读到b这个buf中，

然后就有点奇怪，它读取剩余部分的 “第三个字节” （`b.Byte(2) `), 然后对它判断2或者4。但是承载数据的长度怎么会是2或者4呢？？

先往下看，发现 `b.Advance(5)` 这个东西，不难猜出，它是跳过了前面一段数据，读取里面蕴含的 端口信息， 并给b的UDP项赋值。

然后“第四个字节” 为1时，接着读取一遍数据，这个感觉也有待呢看不懂



### common/mux 和 outbound

rprx说他还改了 mux.cool，那我们应该去 common/mux 里查看

这回还是走捷径，直接查看commit历史，发现这个commit： https://github.com/XTLS/Xray-core/commit/1174ff3090967489e9ac195f9e0db5bf344817be

主要关注 

common/mux 里的

```
client.go, 
frame.go, 
reader.go, 
server.go, 
session.go, 
writer.go, 
```
和 core/xray.go 和 proxy/vless/outbound/outbound.go 

也就是说，xray的mux 包里所有的文件全修改了，全要看。xray.go里面只不过是删除一大段而已，似乎不重要.

client.go 只改了一行，先不看；server.go、session.go 同理。

reader, writer 和 frame文件似乎比较重要，先看frame.go

在 WriteTo 方法中，当 SessionStatus 不是 SessionStatusNew时（估计就是Keep吧），若b这个buf的UDP项不为空的情况下，写一个byte TargetNetworkUDP，然后写下该UDP项的地址和端口。

那么停一下，这里产生两个问题，buf.Buffer什么时候有“UDP”这一项了？这一项又是谁填充的？

在v2ray的代码里可以明确，找不到UDP这一项。那么还是找commit记录，发现 这个commit https://github.com/XTLS/Xray-core/commit/8f8f7dd66f1c116d55feaafde38f4c008592a70a

也就是说rprx在20年年底 就已经添加了这一项，目的是给 ss和 trojan添加udp支持

这一项到底是谁填充的目前还不知道，需要继续阅读其它代码。

然后frame.go 第125行，他加了一个Keep情况下的判断，果然，和我上面的推测吻合。

接着看 writer.go, 在 writeMetaWithFrame 函数中，添加了如下代码：

```go
if len(data) == 1 {
  frame.UDP = data[0].UDP
}
```
显然，就是在这里填充的 UDP项。别看这变量叫frame，实际上还是 Buffer类型。
这一段的意思是，如果 data这个 buf.MultiBuffer（即 `[]*Buffer`）里面实际上只有一段数据，而不是多段，那么，frame这个buffer的UDP项由 `data[0]` 的 UDP项决定。

writeMetaWithFrame 由  Writer.writeData 调用，而它又由 Writer.WriteMultiBuffer 调用。

所以，给 WriteMultiBuffer 传入的数据如果实际只有一段，而且是UDP的，则 frame的UDP也被设为相同的值，然后 Writer.writer 会写入这个frame和 实际数据。

然后再看reader.go ，这个 reader.go就比较有意思了，他首先把 NewPacketReader 的签名变成 `(reader io.Reader, dest *net.Destination)`, 
也就是说，在read执行之前就是已经知道目的地的地址了，然后在 ReadMultiBuffer 里，当使用udp时，就会把读取到的数据 的这个 buffer也添加一个 UDP项。

然后session.go里的变化就明白了，因为它 在NewReader里 传递进了目标地址，然后调用了 NewPacketReader 。
然后server.go里的变化就明白了，因为它就是在 ServerWorker.handleStatusKeep 和 handleStatusNew 方法中，调用 Session.NewReader。

这一切都不够，还要继续阅读。

下面观察 proxy/vless/outbound/outbound.go ，
首先要明确的是 outbound是用于客户端的，就是说客户端向服务器发送请求时所发送的包。

在第179行，在command为udp时，且 端口不是53（dns）也不是443（https），且 h.cone 时，把 request.Command 从 udp修改为 mux，域名改为和正常mux的域名一致，端口改为666.

因此在此就可以确认，xray的fullcone使用的是udp的666端口

然后在195行，判断端口为666且command为mux时，采用 xudp.NewPacketWriter。

然后就没了。等等，咋没了呢，我还是没搞懂，咋就实现了fullcone呢。

那么重新捋一捋。

啥是fullcone？ fullcone就是，每次客户端向服务端的同一个端口发送udp数据，都要使用同一个客户端端口。就这么简单。

那么，只要 xray的服务端 在接收到 之前 发送过的目标端口 的数据 的 第二个数据包时（即Keep中的数据），要使用之前发送第一个数据包 所使用的端口（即发送New数据到远程服务器时所用的端口）

那它真的做到了吗？

首先回到mux的代码，因为此时mux就充当了客户端。 server.go 中 handleStatusKeep 中，在Get一个Session后，会读取请求的数据，然后发送到远端（s.output) 

而这个Get的Session是在 handleStatusNew 里存放的。output项是 link.Writer， 而 `link, err := w.dispatcher.Dispatch(ctx, meta.Target)`

然后 `w.dispatcher 是 routing.Dispatcher`, routing.Dispatcher 是一个 interface，而 `app/dispatcher/default.go`里的 DefaultDispatcher 显然就是一个实现

不过里面似乎没有什么有效信息。总之，目前的理解就是，handleStatusKeep 中 所存放的那个 Session 的 output 应该是原来的output，所以端口自然也是原来的端口，理由是 udp在建立连接后，所使用的端口是固定的。

所以，关键点就是 meta.SessionID，而meta是 FrameMetadata，是直接 读取 ServerWorker.link.Reader 的。 重新查看 frame.go , 看到

```go
type FrameMetadata struct {
	Target        net.Destination
	SessionID     uint16
	Option        bitmask.Byte
	SessionStatus SessionStatus
}
```


而这个 SessionID 被 writer.go的 NewResponseWriter 里 的id 赋值，感觉又循环了，到底谁给 SessionID 赋值的？

思来想去，感觉是和 xudp有关。向mux的server发送数据的只能是 xudp.NewPacketWriter 的 WriteMultiBuffer 函数

重新查看mux.cool 协议，
https://xtls.github.io/development/protocols/muxcool.html#%E6%96%B0%E5%BB%BA%E5%AD%90%E8%BF%9E%E6%8E%A5-new

发现原本Keep超简单，就是 0x2, 然后一个数据。同时我们也看到，Keepalive是 0x4

重新查看 xudp.go，就不难理解了，它查看“第三个字节”，是因为前两个字节是数据段的长度，第三个字节表示 New、Keep或者KeepAlive。所以这里的2和4又是魔数，真实差评啊。

然后就发现，它根本就没设置session ID，即预留的四个字节中，只设置了前两位的长度，第3、4位数空的；然后Keep数据本来 这四个字节后面就应该马上连接承载数据，而rprx在中间插了一段，即xudp中第51行所做的事情，
先放了一个2，然后放进去了端口和地址

然后第97行的“第三字节”的判断就豁然开朗了，因为前面先读了长度，所以现在的“第1、2字节” 就是原来的sessionID项，因为留空 就直接跳过了，然后这个“第三字节”就是在Keep中特别设置好的“2”，

然后 advance后，显然就是读这个端口和地址。读完端口地址后，后面接的就又是普通的 承载数据了.

但是还是没读懂那个“4” 的判断。谁给这个数据写“4” 了？？似乎根本不在 xudp.go里。也许在其他位置有人写了4，也许rprx留了个坑，这个4是以防万一的东西。


总之，sessionID似乎并不重要，重要的还是那个 buffer.UDP 这个  net.Destination。就是说，不知为何，只要指定 目标 UDP，如果ip+端口是与之前相同，那就会使用相同的端口发送。

那么这个逻辑又是在哪里实现的呢？

还是考虑上文提到的那个 给Buffer添加UDP项 那个 commit，https://github.com/XTLS/Xray-core/commit/8f8f7dd66f1c116d55feaafde38f4c008592a70a

这个commit中，给ss和trojan添加了fullcone功能，说明肯定这里是关键。

考察 proxy/freedom/freedom.go 文件


再考察  transport/internet/system_dialer.go 文件的 ReadFrom 和 WriteTo

发现 freedom里 ReadMultiBuffer 会使用 ReadFrom 函数， 而它会把读到的udp地址放到buffer中

然后 WriteMultiBuffer 中，它会 用 WriteTo函数，来给指定的 b.UDP 的目标位置发送数据



这里的关键是，如何记住第一次所使用的源端口呢？应该是 PacketConnWrapper

这个乍一看，每次都会dial，每次都会拨号，但udp只有第一次 是直接拨号，而第二次拨号的话，实际上并没有拨号，而是直接用原来的连接，只要源端口一样就行

dial第一次后，如果是udp，就会返回 PacketConnWrapper；

第二次dial的话，它是会 根据src参数来返回之前所用过的，这是我目前的理解；

然后src参数谁给的呢？似乎是 transport/internet/dialer.go 的 DialSystem 中，利用 ctx 中存储的 outbound.Gateway 来 获取到 src 地址

而这个Gateway 似乎又是 app/proxyman/inbound/worker.go  的 udpWorker.callback 里 如下代码创建的：

```
ctx = session.ContextWithInbound(ctx, &session.Inbound{
	Source:  source,
	Gateway: net.UDPDestination(w.address, w.port),
	Tag:     w.tag,
})
```

总之，错综复杂啊，很多地方我都可能搞错了，比如这里为什么时inbound而不是outbound

无论怎样，它可能 是想办法利用 ctx中存储的 原地址的值 读取到了原来的dialer，然后dial出来的就直接是 原来的 PacketConnWrapper？

总之感觉没有完全看明白。。

仔细思考，应该是说，inbound是针对有udp数据传到服务器的情况，显然不是用于我们的mux，而是ss这种可以有udp流量的情况。
也就是说，ss、trojan这种可以发送udp的代理，上面确实已经证明可以是fullcone的了。但是 mux的部分我还是没有找到证据。
