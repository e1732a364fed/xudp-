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

那么我们看看它的代码实现

首先看xudp的代码 https://github.com/XTLS/Xray-core/blob/bf94fb53caf61244abeb031a6088566290702a0d/common/xudp/xudp.go

观察这个方法：WriteMultiBuffer

在第54行前，前面先写了4个字节的0，然后对 新旧数据分别使用New和 Keep格式，但是注意

而55-60行是什么意思？56、57行 是把消息的头部长度存放到1、2字节中，58、59行 是把承载数据的长度放到 3、4字节中，然后把数据写在后面。

似乎还是没有特殊的


然后观察 ReadMultiBuffer 方法

首先l 是前两字节，是整个数据的长度，然后把剩余部分读到b这个buf中，

然后就有点奇怪，它读取剩余部分的 “第三个字节” （`b.Byte(2) `), 然后对它判断2或者4。但是承载数据的长度怎么会是2或者4呢？？

显然，似乎承载数据的 内容 的设置，并不是在 xudp.go 文件中

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

也就是说，xray的mux 包里所有的文件全修改了，全要看。

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

writeMetaWithFrame 由  Writer.writeData 调用，而它又由 Writer.WriteMultiBuffer 调用

所以，给 WriteMultiBuffer 传入的数据如果实际只有一段，而且是UDP的，则 frame的UDP也被设为相同的值，然后 Writer.writer 会写入这个frame和 实际数据。


