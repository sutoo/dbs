## InnoDB Row Structure
DB_TRX_ID
DB_ROLL_PTR
DB_ROW_ID

# undo log

UNDO LOG存储在UNDO PAGE



## ReadView
实现MVCC的关键，用于实现快照读。ReadView是与SQL绑定的。
ReadView这个数据结构包含下面三个成员：
low_trx_id表示该SQL启动时，当前事务链表中最大的事务id编号，也就是最近创建的除自身以外最大事务编号；
up_trx_id表示该SQL启动时，当前事务链表中最小的事务id编号，也就是当前系统中创建最早但还未提交的事务；
trx_ids表示所有事务链表中事务的id集合。

所有数据行上DATA_TRX_ID小于up_trx_id的记录，说明修改该行的事务在当前事务开启之前都已经提交完成，所以对当前事务来说，都是可见的

RR
事务在第一个Read操作(select 语句)时，会建立Read View
RC
事务在每次Read操作(select 语句)时，都会建立Read View

## 两段锁协议
数据库遵循的是两段锁协议，将事务分成两个阶段，加锁阶段和解锁阶段，两个阶段严格保证先后顺序以及不能重叠。
加锁：事务开始后，遇到一个sql，加其需要的锁
解锁：事务commit时，开始一个个释放锁，

## 当前读:
　　select...lock in share mode (共享读锁)
　　select...for update
　　update , delete , insert
## 快照读
　　简单的select操作(不包括 select ... lock in share mode, select ... for update)。　
  
## Repeatable-read isolation violated in UPDATE
  https://bugs.mysql.com/bug.php?id=63870
  DML语句可以看到一些select看不到的改动（由其他的事务并行提交的）
