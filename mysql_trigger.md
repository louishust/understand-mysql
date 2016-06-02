# MySQL触发器目的表rename后导致原始触发器无法使用


## 问题描述

```
mysql> create table tri0(c1 int);
Query OK, 0 rows affected (0.04 sec)

mysql> create table tri1(c1 int);
Query OK, 0 rows affected (0.01 sec)

mysql> create TRIGGER my_trigger AFTER INSERT ON tri0 FOR EACH ROW INSERT into tri1 values (NEW.c1)
    -> ;
Query OK, 0 rows affected (0.02 sec)

mysql> alter table tri1 rename tri2;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into tri0 values(2);
ERROR 1146 (42S02): Table 'test.tri1' doesn't exist
```

虽然创建了相应的触发器，但是触发器并不会约束对目的表的操作，比如我们将目的表更名，
或者删除，触发器仍然存在，但是触发器失效。

