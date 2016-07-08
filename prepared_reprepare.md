# Prepared statement need to be reprepared


## 问题描述

客户的一个存储过程中使用了prepared语句，不定期的会出现如下报错：


```
ERROR 1615 (HY000): Prepared statement needs to be re-prepared
```


## 解决方法

查找解决方法，发现网上一致性说将参数table\_definition\_cache调大即可。

在mysql bug论坛里面也是这么说：

https://bugs.mysql.com/bug.php?id=42041

bug论坛里面的说法是这不是一个bug，手册中也有提到此问题：

http://dev.mysql.com/doc/refman/5.6/en/statement-caching.html


## 问题原因


table\_definition\_cache这个参数是表示表定义缓存的大小，对应TABLE\_SHARE结构体。
这个cache本身是一个淘汰机制，不能说因为cache不够，就导致语句执行失败，这有点说不过去。

要了解原因，就需要知道prepared语句的原理。

### prepared语句执行原理

prepared语句分为两个阶段:

```
PREPARED STMT FOR 'SELECT * FROM T1';
EXECUTE STMT;
```

首先是prepared阶段，进行语句的解析，并缓存起来，这就是为什么对多此要执行的语句
使用prepared，这样可以减少解析的次数。

然后是execute阶段，此阶段使用prepared缓存的解析结果，直接执行。

这里就有一个问题，即prepared阶段和execute阶段是两个时间点，如果在这两个时间点
之间，表的结构发生变化，比如表的列被drop了，那么execute阶段的就不能直接使用
prepared的缓存。

那么如何判断表结构是否变化了呢？这就依赖于TABLE\_SHARE结构体中的一个成员，
用来表示表的版本信息。如果TABLE\_SHARE在prepared阶段后，被table definition cache
淘汰了，那么它的版本就相当于变化了。

如果在execute阶段，发现表结构变化了，那么就会reprepared，并执行。这就相当于重新prepare，
然后execute，MySQL会尝试3此reprepare，如果3次都失败，那么就会报我们提到的错误。

### 原因总结

出现这个报错的原因就是说，prepared语句尝试了3此reprepare都失败了，这说明3此reprepare后，
表定义都被淘汰出去了，为什么被淘汰呢？因为table defination cache有限，如果当前还有其它正在
运行的语句，那么就会占用table defination cache，而每次reprepare之后，都会将表定义释放回
cache，如果cache满了，就直接丢弃了，那么下次再装载进来就是新的version了。

所以问题的原因就是，有一些长时间执行的语句长期占用table definition cache的空间，导致prepared
涉及的TABLE\_SHARE在释放回cache的时候，被丢弃了。

产生此问题的现场，可能是正在执行备份，或者有很多慢语句，或者prepared涉及的表特别多。

