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

client.go, 
frame.go, 
reader.go, 
server.go, 
session.go, 
writer.go, 

和 core/xray.go 和 proxy/vless/outbound/outbound.go 


