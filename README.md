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

