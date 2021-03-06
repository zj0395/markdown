# Mysql

## 锁
(https://blog.csdn.net/zhanghongzheng3213/article/details/53436240)
### 行锁
- 记录锁(Record Lock)：锁Unique索引或主键索引
- 间隙锁(Gap Lock)：锁住一个索引区间，或在某条索引记录之前后之后加锁
- next-key Lcok：在部分设置下（REPEATABLE READ transaction isolation level and with the innodb_locks_unsafe_for_binlog system variable disabled），使用next-key lock进行搜索和索引扫描，阻止幻读
- 插入意向锁(Insert Intention Lock)：在某索引间隙中插入记录的锁，不同事务插入此间隙中不同位置时不会互斥

### Update锁
- 是行锁
- 如果where中用到了主键索引，mysql会锁定主键索引
- 如果where中用到了非主键索引，msql会先锁定非主键索引，再锁定主键索引
- Set中的字段如果参与了某些索引的话，也会锁定这些索引
- 如果无索引可用，则锁定所有行，扫描全表后释放掉未使用到的行锁
- 避免冲突：使用主键索引进行更新

### 自增锁
- 特殊的表级别的锁，Insert时会首先获取该锁来获取主键id

### Insert锁
- 获取自增索引后，在插入的对应记录上加一个排他行锁
- 自增索引的获取方式由`innodb_autoinc_lock_mode`控制，为0时加自增锁

## InnoDB引擎的索引
- 大大减少服务器需扫描的数据量，可以有效提升查询速度
- 帮组服务器避免排序和临时表
- 将随机I/O变为顺序I/O
- 但索引数量太多会导致插入、删除、更新效率变低

### 聚簇索引
优点：  
1. 可以把相关数据保存在一起。如根据用户id来聚集数据，可以快速查找某用户的全部数据
2. 数据访问较快。索引和数据放在了一个B-Tree里
3. 使用索引查询可直接使用主键值。
缺点：  
1. 聚簇索引提高了IO密集应用的性能。数据全放内存中时，聚簇索引无优势
2. 插入速度依赖于插入顺序。顺序插入是最快的
3. 更新索引代价很高。会移动每个被更新的行
4. 页分裂问题。插入到已满的分页时，会将该页分成两页
5. 全表扫描变慢。尤其是行较稀疏或分裂导致的不连续
6. 二级索引变大。二级索引中存放了主键列
7. 二级索引需要两次索引查找。二级索引中获取主键值，再根据值去聚簇索引中查找数据

### B-Tree+索引
- 所有值按顺序存储
- ***索引的顺序很重要***
- 相比二叉树减少I/O次数，每一层包含更多节点
- 相比B-Tree减少I/O次数，因为只有叶子节点包含数据，索引节点不包含数据，一次可以读取更多的索引节点
- 相比B-Tree范围查找方便，不用再去上一层找指针，因为节点之间是链表

有效查询：
- 全值匹配：查询索引中全部字段
- 匹配最左前缀：查询索引中最左边几个列
- 匹配列前缀：匹配某一列的值的开头部分
- 匹配范围值：范围查询
- 精确匹配某列并范围匹配另一列
- 只访问索引的查询：覆盖索引，索引中字段满足查询

限制：
- 不是从索引最左列开始，则无法使用此索引
- 不能跳过索引中的列
- 如果查询中有某个列的范围查询，则右边的所有列都无法使用索引来优化查找

## 事务隔离级别
- 脏读：事务A读取到了事务B未提交的修改
- 不可重复读：事务A对同一条记录第一次和第二次读取到了不同的值，因为穿插了事务B的修改
- 幻读：事务A第一次读取的记录条数和第二次读取到的记录条数不一样，因为穿插了事务B的新增/删除

- 当前读：`Update, Insert, Delete`使用当前读
- 快照读：`Select`使用的快照读，在可重复读及以上隔离级别时，只读取一次MVCC下的当前版本；在可重复读以下隔离级别时，每次都读取

不同隔离级别下会出现的情况：
| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
| :-: | :-: | :-: | :-: |
| 读未提交 | √ | √ | √ |
| 读已提交 | × | √ | √ |
| 可重复读 | × | × | √ |
| 串行 | × | × | × |


## RedoLog 和 BinLog
### RedoLog
引擎层需要读取磁盘数据到内存中，并在修改后写回磁盘。写盘是随机磁盘I/O，每次都写太低效，数据积累一定量以后再写盘效率更高。但如果写盘之前数据库崩溃，则数据会丢失。RedoLog解决写盘之前数据丢失的问题

### BinLog
异常重启时根据BinLog进行数据恢复

完整事务流程：
1. Client向Server发送Sql
2. Server向引擎层发送Prepare，引擎层写RedoLog
3. Server写BinLog
4. Server向引擎层发送Commit，引擎层写RedoLog记下Commit
异常恢复：
- 读取BinLog进行恢复
- 在步骤2及之前异常的事务，丢弃此事务
- 在步骤3时异常，丢弃此事务
- 在步骤4异常，重做此事务
