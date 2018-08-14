# proxyme 一个http代理

使用java NIO的http代理。支持https。建议不要再chrome上使用本代理，因为chrome本身会请求很多谷歌的api，结果被墙住了，又只有两个线程，导致其他都被阻塞，很尴尬。

之前也打算做过这个东西，结果做出来的有点缺陷（现在想可能是selector中锁的问题，忘记了）。这大概隔了半年，这个项目的http代理功能实现了。



## 性能与内存

占用cpu不到1%

内存最大35m（不含jvm自身）。GC次数和时间很少

总的来说，性能可以了吧。

## 思路

两个线程，每个线程一个selector。

localSelector线程，负责接收本地浏览器的连接请求和读写浏览器到代理的socketChannel

remoteSelector线程，负责读写web服务器到代理的socketChannel。

ChannelBridge类,持有localSocketChannel和remoteSocketChannel。职责是处理请求和响应，并转发。

RequestHeader类，职责是格式化请求行和请求头。

## 实现中的注意点

首先是健壮性！每一个try块都是很重要的！都解决了一个问题

其次是锁的问题：

selector.select()会占有锁，channel.register(selector)需要持有同样的锁。

如果调用上面的两个方法的语句在两个线程中，会让channel.regiter等很久很久，导致响应难以及时得到。

而在实现中，这是一个生产者消费者问题。localSelector线程根据本地浏览器请求产生了一个从代理到web服务器的remoteChannel。而remoteSelector要接收这个remoteChannel,这也就是消费了。

很自然的，避免上面锁等待最好的方法：localSelector生成remoteChannel，将其放入队列。remoteSelector线程从队列中取。再结合selector.wakeup()使其从阻塞中返回，可以快速地接收（register）这个remoteChannel。

这两点，就是最最重要的两点了。

另外还有，因为代理而需要改变的请求头了，参见`com.arloor.proxyme.RequestHeader.reform()`方法。

最后，https代理实现中的坑。http代理传输的内容是明文，字节肯定大于0，而https传输的字节可能小于0。因为这个，传输https数据的bybebuff时，要特意指定bytebuff的limit为实际大小。

http代理不神秘。

## 问题

有个致命的问题：如果输了一个不存在的host，需要大概25s才能知道这个host无法解析。这一段时间，25s都无法处理浏览器的请求。

这就带来新的任务啦。读写channel可以用线程池搞。多弄几个线程用于读写。这也不是难事。

另外上传文件（向远程的请求过快过大）会失败，写的byte数为0，这也是一个问题。很可能是tcp缓冲区的问题。

## 运行日志

```
11:14:31.760 [localSelector] INFO com.arloor.proxyme.LocalSelector - 接收浏览器连接: /127.0.0.1:50057
11:14:31.760 [localSelector] INFO com.arloor.proxyme.ChannalBridge - 请求—— CONNECT www.bing.com:443 HTTP/1.1
11:14:31.838 [localSelector] INFO com.arloor.proxyme.ChannalBridge - 创建远程连接: www.bing.com/202.89.233.101:443
11:14:31.838 [remoteSlector] INFO com.arloor.proxyme.RemoteSelector - 注册remoteChannel到remoteSelector。remoteChannel: www.bing.com/202.89.233.101:443
11:14:31.838 [localSelector] INFO com.arloor.proxyme.ChannalBridge - 发送请求517 -->www.bing.com/202.89.233.101:443
11:14:31.947 [remoteSlector] INFO com.arloor.proxyme.ChannalBridge - 接收响应2720 <-- www.bing.com/202.89.233.101:443
11:14:31.963 [remoteSlector] INFO com.arloor.proxyme.ChannalBridge - 接收响应1360 <-- www.bing.com/202.89.233.101:443
11:14:31.963 [remoteSlector] INFO com.arloor.proxyme.ChannalBridge - 接收响应1360 <-- www.bing.com/202.89.233.101:443
11:14:31.963 [remoteSlector] INFO com.arloor.proxyme.ChannalBridge - 接收响应1361 <-- www.bing.com/202.89.233.101:443
11:14:31.963 [localSelector] INFO com.arloor.proxyme.ChannalBridge - 发送请求93 -->www.bing.com/202.89.233.101:443
11:14:31.963 [localSelector] INFO com.arloor.proxyme.ChannalBridge - 发送请求811 -->www.bing.com/202.89.233.101:443
11:14:32.041 [remoteSlector] INFO com.arloor.proxyme.ChannalBridge - 接收响应120 <-- www.bing.com/202.89.233.101:443
11:14:32.041 [localSelector] INFO com.arloor.proxyme.ChannalBridge - 发送请求38 -->www.bing.com/202.89.233.101:443
11:14:32.119 [remoteSlector] INFO com.arloor.proxyme.ChannalBridge - 接收响应38 <-- www.bing.com/202.89.233.101:443
11:14:32.166 [remoteSlector] INFO com.arloor.proxyme.ChannalBridge - 接收响应481 <-- www.bing.com/202.89.233.101:443
11:14:32.181 [remoteSlector] INFO com.arloor.proxyme.ChannalBridge - 接收响应38 <-- www.bing.com/202.89.233.101:443
11:14:32.244 [localSelector] INFO com.arloor.proxyme.ChannalBridge - 发送请求260 -->www.bing.com/202.89.233.101:443
11:14:32.369 [remoteSlector] INFO com.arloor.proxyme.ChannalBridge - 接收响应421 <-- www.bing.com/202.89.233.101:443
```
