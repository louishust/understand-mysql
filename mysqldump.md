# mysqldump的风险与控制

mysqldump作为比较常用的逻辑备份工具，是每个DBA需要掌握的，但是你知道
mysqldump带来的风险么？


## 风险

偶然发现mysqldump的过程中，发现processlist里面很多线程的状态与下面类似
（自己做的实验达到的效果）：

```
| 2587 | root | localhost | NULL | Query      |   52 | Waiting for table flush | flush tables                       |
| 2588 | root | localhost | test | Query      |   55 | User sleep              | select sleep(1000) from t1 limit 1 |
| 2589 | root | localhost | test | Field List |   42 | Waiting for table flush |                                    |
| 2591 | root | localhost | test | Query      |   22 | Waiting for table flush | select * from t1 limit 1
Waiting for table flush
```


flush tables等待一个慢查询，然后后续其他线程的语句等到flush tables结束，
导致线程堆积。结果就是MySQL不能对外提供服务了。

解决方法是kill掉flush tables等待的慢查询。


## 控制

为什么mysqldump会执行flush tables呢？代码片段如下：

```
 if ((opt_lock_all_tables || opt_master_data ||
           (opt_single_transaction && flush_logs)) &&
          do_flush_tables_read_lock(mysql))
```

即如果满足如下三个条件任意一个，就会执行FLUSH TABLES:

1. lock-all-tables
2. master-data
3. single-transaction && flush-logs

### lock-all-tables

此参数不能与single-transaction同时设置。如果设置了此参数，lock-tables参数无效，因为已经FLUSH TABLES WITH READ LOCK了，不需要再针对具体的表进行LOCK TABLES xxx read 操作了。

执行流程如下：

1. connect mysql
2. FLUSH TABLES;
3. FLUSH TABLES WITH READ LOCK;
4. 查询表导出数据
5. disconnect



### master-data

用于记录binary log的当前位置。可以设置成1或2，区别是是否对输出语句添加注释。

如果设置了master-data同时没有设置single-transaction，那么就默认设置lock-all-tables. master-data也会导致dump之前的flush操作。与lock-all-tables类似，只是多了个master binlog的信息。


### single-transaction

这个只针对具有MVCC的引擎才有效，比如InnoDB。此选项的效果就是能尽快的进行UNLOCK TABLES来释放锁。

如果仅仅有此选项，则只能进行一个快照备份，无法和binlog联系到一起，所以用处不大。所以可以和master-data结合起来使用。

这样就能得到binlog信息，还能尽快释放锁。

执行流程如下：

1. connect mysql
2. FLUSH TABLES;
3. FLUSH TABLES WITH READ LOCK;
4. SHOW MASTER STATUS；
5. SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ
6. START TRANSACTION /*!40100 WITH CONSISTENT SNAPSHOT */
7. UNLOCK TABLES;
8. 查询表导出数据
9. disconnect


可以看出由于可以进行快照读，那么占用锁的时间就会非常短，考虑到目前InnoDB是默认的储存引擎，那么single-transaction + master-data的方式就非常高效了。



### 如何解决慢查询对FLUSH TABLES的影响？

这个确实无法解决，如果mysqldump做成定时任务，而慢查询是不定时的，
那么很可能就悲剧了，只能认为干预了，或者把慢查询放到离线库执行。

所以保证一个系统无慢查询状态是多么的重要啊！！！
所以保证一个系统无慢查询状态是多么的重要啊！！！
所以保证一个系统无慢查询状态是多么的重要啊！！！


当然还有一个工具叫做Percona Xtrabackup，可以实现物理热备份。

