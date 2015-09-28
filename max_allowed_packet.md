# 深入理解max_allowed_packet参数

运维过MySQL的人都应该对这个参数不陌生，因为大家可能偶尔会遇到如下的错误提示：

```
Got packet bigger than 'max_allowed_packet' bytes
```
然后大家就去修改这个参数，把它调大，然后问题解决，那你知道你真正解决了什么么？本文就具体介绍这个参数到底控制了什么。


##  什么是packet

首先我们需要知道什么是packet？

MySQL是C/S结构，client与server之间的通信需要遵循约定的规则，通信的最小单元就是packet，通信约定的规则就是packet的结构。


###  packet的结构

packet包含两部分：header与body。

header: 4个字节，前三个字节表示整个packet的长度（包括header），第4字节表示packet序列号，即packet number。

body：是packet的具体内容，根据不同的语句类型，可以有很多类型的body，body的第1个字节表示语句类型，后续字节表示具体的内容。

### 如果packet大小超过3个字节表示范围？

packet的body因为是不固定的，而header表示长度的只有3个字节，
3个字节的表示范围0 - 2^24-1，那么如果超过这个范围怎么办？

这时候如果packet的长度大于或等于2^24-1，那么会设置packet的header前三个字节为2^24 - 1,表示此packet还没结束，也就是说如果一个packet超过16M，那么就会拆分成多个packet。


### insert语句packet示例

假设client发送了一条语句：

```
insert into t1 values(1);
```

上面的语句共22个字节，所以body的长度是22+1=23（因为还有一个语句类型标志字节）。那么header前三个字节用于存储23，第4个字节就是包的序列号，从0开始。

body的字节内容就是：类型+ "insert into t1 values(1)"
这里的类型是COM_QUERY。表示QUERY类型的语句，当然这里的COM_QUERY不是指查询，而是包含了DDL和DML的语句。

还有一些其它类型的标识，比如COM_INIT_DB,我们执行use命令时，就是传输的这个标志，还有COM_QUIT，客户端连接退出的时候就会发送这个标志位。


## max_allowed_packet只是服务器的参数么？

其实max_allowed_packet参数不仅仅是server的参数，还是client的参数，因为包的传递是相互的。

不论对于server还是client，他们使用了同样的底层网络代码，所以行为也是一样的，对于这个参数的作用也是一致的。

客户端与服务器的这两个参数要保持一致，不然会出现各种问题。


## 什么情况导致Got bigger packet?

如果是net write操作，也就是发送消息的操作，那么packet不会受到
max_allowed_packet参数的限制，而相反的，如果是net read操作，就会
受到这个限制。

也就是说，如果服务器传递一个很大的packet，此packet超过了client端设置的max_allowed_packet值，那么客户端就无法接收这个消息，因为客户端不可能根据传过来的packet无限制的申请内存，接受失败后就会主动断开与server的连接。

相应的，如果客户端发送一个非常大的packet，比如一条非常长的语句，
那么server端接收的时候也会对比max_allowed_packet进行验证，如果超过了max_allowed_packet,则接收失败，主动断开客户端连接。



## max_allowed_packet与net_buffer_length的关系

max_allowed_packet表示接收的packet的最大的大小，而net_buffer_length表示本身消息缓冲区的大小，因为packet是先写入本身的缓冲区，然后才通过socket发送。所以net_buffer_length的设置要小于max_allowed_packet，因为反之的话，没有意义，对于一个大的packet，即使buffer够，也会受到max_allowed_packet的参数限制。

net_buffer_length这个参数用于初始化net的buffer大小，但是这个大小是可变的，如果接收的packet大于这个大小，则会动态调整buffer大小，最大调整到与max_allowed_packet一样大小。这就是这两个参数的关系。


## 参数默认值

以5.5.39为例，具体的参数值随版本的变化有调整。

客户端max_allowed_packet取值范围4096-2G，默认值是16M

客户端net_buffer_length取值范围1024-512M，默认值16K

服务器max_allowed_packet取值范围1024-1G，默认值是1M

服务器net_buffer_length取值范围1024-1M, 默认值16K
