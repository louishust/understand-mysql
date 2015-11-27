# 深入理解max_connect_errors参数

有些安全措施需要防止暴力破解数据库密码，记得去年因为MySQL的内存比较函数
使用存在漏洞，导致暴力破解很快就可以进入系统，那么就有人发现一个全局变量
max_connect_errors，是不是设置了这个值就可以防止保利破解了呢？？？


我们看一下官方文档是如何解释这个参数的：

>If more than this many successive connection requests from a host are interrupted without a successful connection, the server blocks that host from further connections. You can unblock blocked hosts by flushing the host cache. To do so, issue a FLUSH HOSTS statement or execute a mysqladmin flush-hosts command. If a connection is established successfully within fewer than max_connect_errors attempts after a previous connection was interrupted, the error count for the host is cleared to zero. However, once a host is blocked, flushing the host cache is the only way to unblock it. The default is 100.

大概的意思是如果连续的连接请求失败，那么主机就会被记入黑名单。
一切看起来那么美好，其实不然。

经过用户错误密码连续失败后，发现主机并没有记入黑名单。

其实这个参数的主要作用是防止在建立连接的过程中失败，MySQL的认证是三次握手，如果仅仅是telnet MySQL端口的话，不真正建立连接，那么此参数就生效了。


```
➜  build  telnet 172.16.54.177 3306
Trying 172.16.54.177...
Connected to 172.16.54.177.
Escape character is '^]'.
kHost '172.16.54.177' is blocked because of many connection errors; unblock with 'mysqladmin flush-hosts'Connection closed by foreign host.

```

如上所示，telnet之后，host被加入黑名单，需要flush host来解除。
所以，这个参数其实用处并不大，因为这个参数仅仅是为了防止端口探测，
但是谁的扫描器会不断探测端口，基本上探测一次找到就OK了。


## 那么如何才能解决防止暴力破解呢？

1. MySQL服务器放到内网中，不暴露到公网中。
2. 创建用户时指定具体的IP，不允许使用%用户。
3. 设置防火墙规则，只允许固定的APP服务器连接数据库。
4. 开发一个认证插件，实现连续登录失败锁定帐号的功能。

