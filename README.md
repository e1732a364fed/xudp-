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


总之，之所以ss、socks5能fullcone，是因为它们都可以直接传输在udp上；而且socsk5就算传输在tcp上，因为协议厉害，也是可以传输udp的fullcone的。trojan和socks5类似。

实际上trojan就相当于一个更防探测 的socks5.

然后就差vless和vmess，因为协议限制，传输数据时并不附带地址，也就是说是设计时就没搞好，所以正常情况不行。而xray的方法就是利用mux协议来做到，绕过vless。

所以实际上没有加强vless、vmess，只不过是绕过了它们，改用新协议而已。

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

这里的outbound实际上的行为就是，你虽然说vless拨号，但是我这里硬把你改成了mux拨号，所以实际上发送出去的流量是mux格式，和vless毫无关系（vmess的也是一样的）；或者说vless里套了一个mux协议，所以，低级协议可以通过内含高级协议来进行扩展。

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
也就是说，ss、socks5 这种可以发送udp的代理，上面确实已经证明可以是fullcone的了。但是 mux的部分我还是没有找到证据。

关键点在于，mux中接到的实际上是 tcp流量，而向 freedom发送时却变成了udp； 而 ss这种用 udp发送过来的流量，直接就可以重用这个 udp端口接着直接用 freedom 向远程udp发送数据。

然而，最新的 transport/internet/dialer.go 似乎和那个还不一样：

https://github.com/XTLS/Xray-core/blob/main/transport/internet/dialer.go

是用的 RegisterTransportDialer 的dialer，不过找来找去发现还是 DialSystem 的套壳，然后是 effectiveSystemDialer 套壳，然后是 transport/internet/system_dialer.go  的 DefaultSystemDialer 套壳。

总之，也许如果没有源udp的端口的话，可能会在后面得到端口后什么位置把原端口写入了ctx？但是还是找不到。也就是说，mux方法的fullcone应该说是不持久的，重新dial的话应该就会得到不同的端口。

## 关于NAT测试

按理说，它是可以通过NAT测试得到fullcone的。关键就在于，它只dial了一遍。上文是始终没有找到dial多遍能使用原端口的证据；但是如果是dial一遍的话，后面紧接着的read或者write都是通过原本已经建立的conn来进行的（就是那个 PacketConnWrapper），当然就不会变换端口，测得的当然是fullcone

问题是，如果客户端关了这个连接后，很久以后，比如一个小时以后，那 PacketConnWrapper肯定失效了，因为关闭了连接后，这个东西就没用了，被内存所回收。然后再次访问远程相同的udp端口的话，就会重新执行dial，此时的话，mux方法就会使用新的不同的端口进行通讯了，那么就不是fullcone了。

这个行为就和ss的fullcone的实现不一样？因为ss那种原生就可以用udp传输的，原端口是保存着ctx中的，而且只要dial，就会通过 udpWorker把原端口保留在ctx中。不过感觉还是没读明白，有机会一定要完全解释清楚。

总之本文中极可能蕴含着巨大错误啊，实在是搞不懂。


所谓udp over tcp, 其实转换了好几遍，先是客户端的应用程序向我们代理客户端发送udp请求，然后我们包tcp发送到代理服务器，然后代理服务器解析后，再重新用udp发送到远程服务器。此时代理服务器与远程服务器是直连的对等的，没nat存在，所以远程服务器发送到代理服务器的回复，代理服务器可以直接收到，然后它就重新用tcp包装，然后发送给客户端的代理程序，然后客户端解析后，再重新把它变成udp发送给客户端应用程序。这就是一个完整的过程。此时无论怎么发送，肯定都是fullcone，关键在于根本没关闭链接，就像直接在vps上发udp是一样的。


fullcone解决什么问题？打洞问题。即所谓客户端当做游戏服务器，或者多个客户端之间的p2p连接问题。就是说此时，会有一个奇葩的你没访问过的ip加port这么一个地址（即另一个藏着另一个nat后面的游戏客户端）向你的vps发udp数据，向你拨号的端口依然是你之前访问远程服务器所用的端口，此时的话，你的vps自然是也能收到的，然后也能传递给你，然后你的自己的客户端应用程序就会收到链接信号，游戏可以玩了！


但是，如果你的客户端关闭了连接（比如你的游戏没关闭，但是你的代理客户端出了问题，你重启了一下代理客户端，此时显然关闭再打开了一遍），然后再重新建立与vps的连接，此时，mux的问题似乎就产生了！远程的另一个玩家可不知道你重新开闭过代理它会继续向之前他使用好使的那个端口发送信号。遗憾的是，根据上文的探讨，mux办法是不会存储多个dial后前一个的信息的导致了你的端口更改了！然后你的小伙伴就尴尬了，发现你突然掉线了


而ss这种不需要mux的原生udp的就好很多，因为udp属于无状态协议，你客户端开闭N遍实际上代理服务端根本不知道，所以它依然储存着原始信息，小伙伴向你继续发送到信息仍然会被收到


重中之重是什么，为什么我一再探究这个区别？实际上是因为，我已经想到了在udp over tcp中可以保持状态的办法，实现全然与rprx的方法不同，所以当然有必要先指出不同办法的必要性，即解决上述问题。
而且新办法一定会比上文讨论的办法要简单得多，没有那么多中间代码。

## sagernet版 v2ray也采用了 fullcone

它直接把xray代码和v2ray代码杂糅在了一起
见
相关sagernet的commit在 https://github.com/SagerNet/v2ray-core/commit/dbad7b0ce458675aea08c5eed772dd36a6a463d6

22年1月新添加的这个commit。

# 后记

本文的探索，不得不又陷入了小白探索v2ray代码的泥淖，

下面的源码分析也提到过
https://medium.com/@jarvisgally/v2ray-源代码分析-b4f8db55b0f6

>使用上下文传递参数实际上是实现了一个参数可以随意变化的接口，对于V2Ray这种可拔插的组件架构，会导致代码难以阅读和维护，因为无法确定哪个组件对上下文中的参数做了哪些处理，往往只有运行时才能确定；初次接触项目的时候，经常会发现有参数从上下文中取出来了，但是一时半会找不到是在哪里写进去的，很容易迷失；此外，随着越来越多人加入项目，由于通过上下文传递参数是「方便」的途径，渐渐的就会加入各种奇奇怪怪的东西了。

本文的探索，最后的源地址 似乎就是从 上下文提取出来的。

v2ray太复杂了，可能是我见过最复杂的代理之一。

