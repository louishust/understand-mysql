# MySQL占用内存持续升高


## 问题描述

用户有两个独立的MySQL实例1和2，每天夜里在主MySQL1上进行mysqldump备份，
然后将备份出来的语句灌到MySQL2上，发现MySQL2的内存每天都会持续增长。

因为每天mysqldump出来的语句是先drop table，然后create table，insert into。
所以占用的内存应该是基本固定的，为什么会一直增长呢？


难道是drop table并没有释放对应的内存么？


## 问题分析

buffer pool 含有几个链表，这里比较重要的是两个链表free, LRU。

buffer pool在初始化的时候，会按照16K作为一个分配单元（称为page），挂载到free链表上。
比如设置的innodb\_buffer\_pool\_size为128M，那么按照16K为单元，会将128M切分为8192个page。
也就是初始化之后free链表上挂在了8192个page页，每个页占用内存大小为16K。

当我们进行数据的读取，写入操作，其实并不是直接写文件，而是先将文件读入到buffer pool中，
这就需要先从free链表上进行获取一个空闲的page，才能将磁盘内容映射到内存中。free链表就
拿出来一个page，然后这个page会挂载到LRU链表上，表示此page正常被使用。

现在结合从库的备份恢复逻辑，解释下为什么物理内存一直上涨。

1. buffer pool初始化的时候只是申请内存，而没有实际使用，所以MySQL初始化后物理内存占用(RES)应该比较少。
2. 创建表，导入数据的时候，会使用buffer pool上的free链表的page，这样free链表的page减少， LRU链表的page增多。
3. DROP TABLE时，并不会清除LRU链表上的page。因为buffer pool的内存分配机制是， 先从free链表上找空闲的page，如果free链表用完了，那么就淘汰LRU链表最后的页，来使用。 所以LRU上的page在drop table时并不会清除。

总结一下此问题的主要原因：

buffer pool申请page的时候是先看free链表，如果free链表为空，就会淘汰LRU链表的page来使用，
同时，LRU链表的page不会随着DROP table而还原到free链表上。

相关函数：

1. 申请空闲page的相关函数
    buf_LRU_get_free_block

2. 删除table的相关函数
    ha_innobase::delete_table
    row_drop_table_for_mysql
    fil_delete_tablespace
    buf_LRU_flush_or_remove_pages
