# 闰秒导致MySQL CPU飙高

偶然发现一台比较空闲的MySQL服务器的CPU利用率比较高，在100%以上，
但是并没有什么连接，非常奇怪。


## 找耗CPU的线程

由于没有什么连接，所以processlist里面没什么内容。那么就只能看线程了。

```
top -p `pidof mysqld`
```
然后输入H，这样可以看到每个线程占用的CPU.

```
- 17476  0.0 
    - 17478  0.0 
    - 17479  0.0 
    - 17480  0.0 
    - 17481  0.0 
    - 17482  0.0 
    - 17483  0.0 
    - 17484  0.0 
    - 17485  0.0 
    - 17486  0.0 
    - 17487  0.0 
    - 17489 61.6 
    - 17490 74.0 
    - 17491  0.0 
    - 17492  0.0 
    - 17495  0.0 
    - 17519  0.0 
    - 25731  0.0 
    - 25732  0.0 
    - 25733  0.0 
    - 25739  0.0 
    - 25741  0.0 
    - 25742  0.0 
    - 25750  0.0 
    - 25751  0.0 
    - 25752  0.0 
    - 25861  0.0 
```

此时，看到有两个线程占用CPU比较高，这时候需要pstack一下mysqld进程，
打印当前进程的堆栈信息，发现了两个占用CPU的线程：

```
Thread 17 (Thread 0x7f4c566b7700 (LWP 17489)): 
#0  0x00000000007e7ed4 in srv_lock_timeout_thread (arg=<optimized out>) at /pb2/build/sb_0-12740893-1405751465.66/mysql-5.5.39/storage/innobase/srv/srv0srv.c:2347 
#1  0x00007f4cb63bb806 in start_thread () from /lib64/libpthread.so.0 
#2  0x00007f4cb565359d in clone () from /lib64/libc.so.6 
#3  0x0000000000000000 in ?? () 
Thread 16 (Thread 0x7f4c564b6700 (LWP 17490)): 
#0  sync_array_print_long_waits (waiter=0x7f4c564b5e98, sema=0x7f4c564b5e90) at /pb2/build/sb_0-12740893-1405751465.66/mysql-5.5.39/storage/innobase/sync/sync0arr.c:955 
#1  0x00000000007e7b51 in srv_error_monitor_thread (arg=<optimized out>) at /pb2/build/sb_0-12740893-1405751465.66/mysql-5.5.39/storage/innobase/srv/srv0srv.c:2490 
#2  0x00007f4cb63bb806 in start_thread () from /lib64/libpthread.so.0 
#3  0x00007f4cb565359d in clone () from /lib64/libc.so.6 
#4  0x0000000000000000 in ?? () 
```


## 找bug

这两个消耗CPU的线程都是后台线程，按理说不应该这么消耗CPU的。找了一下，
在mysql bug里面找到了几乎一样的bug报告：

https://bugs.mysql.com/bug.php?id=65778

官方说这是linux内核bug，由于闰秒导致的，同时也找到了相应的解决方法：

https://blog.mozilla.org/it/2012/06/30/mysql-and-the-leap-second-high-cpu-and-the-fix/

1. 重启机器
2. date -s "`date`"

## 闰秒为什么导致CPU飙高？

网上大多给出来的是解决方法，基本上没有人解释为什么会导致CPU飙高。

首先在linux内核上确实是一个bug：Potential fix for leapsecond caused futex related load spikes

http://marc.info/?l=linux-kernel&m=134113577921904

由于对linux内核不了解，这个bug的具体原因我也解释不好，大概的原因是闰秒发生时，CLOCK_REALTIME
相关的变量没有设置正确，导致一些对于CLOCK_REALTIME的调用会出现偏差。
这个bug可以导致的问题就是如果存在基于CLOCK_REALTIME的计时器（timer），
那么计时器就会提前结束。对于次秒级别（小于1s）的计时器，会立刻返回。
这个CLOCK_REALTIME是本地时间，你说的通过NTP同步时间，NTP是会修改这个CLOCK_REALTIME的，
最终系统调用还是调用linux里面的CLOCK_REALTIME。

下面再结合MySQL的代码说明，具体的两个消耗CPU的线程分别是：

```
srv_error_monitor_thread和srv_lock_timeout_thread。
```

这两个函数都是后台死循环函数，每隔1s做一次检查。
这就用到了timer，前面提到了，如果timer是次秒级别的，会立刻返回，
就相当于timer的阻塞功能没起到作用。

上面两个函数都调用了如下函数：

```
os_event_wait_time_low(xxx_event, 1000000, sigcount);
```

这个函数具体到系统函数调用就是：pthread_cond_timedwait （基于CLOCK_REALTIME）
即等待event 1000000微秒（即1s）返回。刚好针对这个内核bug，对于次秒级的timer，会直接返回。
也就是说，原本每隔1s执行一次的while循环，现在变成了无须等待的直接执行了。

