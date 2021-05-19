原文 https://dev.mysql.com/doc/refman/8.0/en/innodb-multi-versioning.html
## 什么是undo log
定义：是与单个读写事务关联的undo log records的集合。

## 什么是undo log records
undo log records包含有关如何撤消事务对聚簇索引记录的最新更改的信息。如果另一个事务需要读取原始数据（一致性读），则将从undo log records中检索未修改的数据

## undo log records 存在哪？
undo tablespaces 里的 rollback segment 的 undo log segments中。

## Undo log 主要有2类：insert 和 update
一个事务最多可以分配四个撤消日志，以下每种操作类型都可以分配一个：
INSERT operations on user-defined tables
UPDATE and DELETE operations on user-defined tables
INSERT operations on user-defined temporary tables
UPDATE and DELETE operations on user-defined temporary tables

所以同一个事物，会有2套 undo log，（不考虑临时表的情况）
* insert 插入撤消日志仅在事务回滚时才需要，并且在事务提交后可以立即将其丢弃。
* update undo log 除了用于回滚，也用于一致性读。它也是可以被丢弃的，在一定的条件下。


## innodb支持并发事物数
支持的事物数，取决于：
 * number of undo slots in the rollback segment . undo slot的大小 = InnoDB Page Size / 16
 * the number of undo logs required by each transaction. 
e.g.
如果每个事物都执行一个INSERT 和一个 UPDATE or DELETE操作, innodb可以支持的最大并行读写事物数为：
```
(innodb_page_size / 16 / 2) * innodb_rollback_segments * number of undo tablespaces
```
 
## innodb 每行 额外的3个字段：
* DB_TRX_ID 
字段表示插入或更新该行的最后一笔交易的交易标识符。此外，删除在内部被视为更新，在该更新中，该行中的特殊位被设置为将其标记为已删除。
* DB_ROLL_PTR
称为滚动指针的字段。回滚指针指向写入 rollback segment 的 undo log。如果该行已更新，则undo log将包含在更新该行之前重建该行的内容所必需的信息。
* DB_ROW_ID
默认的主键

## 什么是purge
被标记删除的 row 和index，何时会被物理删除？在其对应的update undo log被丢弃的时候执行。
purge操作可能会由于一直批量的insert+delete而滞后，这样，标记删除的row会导致表空间越来越大。
