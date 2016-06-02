# Keepalived Vrrp异常退出


## 问题描述

用户使用keepalived漂VIP的方式来做MySQL高可用。
keepalived的检测脚本逻辑是检测到MySQL不存活就执行killall -s SIGTERM keepalived.

用户在测试的时候，发现keepalived的日志有时不正常，
正常情况下vrrp进程会广播一个消息（告知其它节点我要退出了）后正常退出。
但是日志发现vrrp进程有时候不广播消息，导致其它节点切换成master变得比较慢。
也就是vrrp进程在处理SIGTERM的时候概率性的不广播消息。


## 问题原因

这是1.2.17的一个bug，触发vrrp进程无法正常退出的逻辑如下：

1. 外部程序发送KILLALL -s SIG_TERM信号
2. keepalived父进程和vrrp子进程收到TERM信号
3. vrrp子进程进入stop_vrrp函数，并执行了signal_handler_destroy函数，相当于清除了所有的信号处理逻辑。
4. 父进程执行sigend函数，里面会执行kill命令，即给vrrp子进程发TERM信号， 
而此时vrrp子进程已经清除了所有信号处理，所以收到TERM信号后，会直接退出。

由于父进程和子进程是相互独立的，所以必须按照上面的时序才会产生异常的情况，这也是为什么用户测试的时候
概率性的出现这个问题。

