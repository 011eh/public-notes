# 21 事务隔离级别与MVCC

## 事务并发执行的问题

> 事务的并发执行会出现一致性问题，解决方法有让事务串行执行，这种方式会严重影响到系统吞吐量、资源利用率，还有可串行化执行，对访问相同数据的事务进行限制，多个事务对同一个数据进行`读写`、`写写`情况的访问时，才会出现一致性问题。
>
> **一致性问题**
>
> - 脏写：事务修改了另一个未提交事务中的数据
> - 脏读：事务t1修改了数据，t2访问了未提交t1修改的数据后，t1终止，t2访问的数据是不正确的
> - 不可重复读：t1读取数据后，t2修改了未提交事务读取的数据，t1再次读取的数据与第一次读取的数据不一致
> - 幻读：t1读取了符合查询条件的记录，t2写入了符合查询条件的数据，t1再次读取数据不一致。



> **提示**
>
> 对MySQL来说，幻读强调的是事务`读取了之前没有读取的记录`，对于“事务多次读取的数据不一致”的情况，MySQL把它归类为`脏读`。



### 4种隔离级别

> - 读未提交
> - 读已提交
> - 可重复读
> - 序列化
>
> MySQL支持4种隔离级别，默认的隔离级别是`可重复读`，MySQL在可重复读的级别下，很大程度解决了幻读



### 设置隔离级别


> 如果没有没有指明`global`、`session`关键字，则只对下一个事务有效，可以通过启动选项`transaction-isolation`来指定服务器启动时的隔离级别

```sql
set [global|session] transaction isolation level 级别;

# 级别
read uncommitted
read committed
repeatable read
serializable
```

```sql
# 查看事务隔离级别
show variables like 'transaction_isolation';
```



## MVCC原理

> 对于使用InnoDB存储引擎的表，聚簇索引记录有`2个必要的隐藏列`——`trx_id`、`roll_pointer`，每对记录进行1次改动，都会记录一条undo日志，`roll_pointer`这个属性可以将undo日志串成一个链表，这个链表称为`版本链`，链表的头节点是记录的`最新值`，`trx_id`属性记录了修改记录的事务Id。
>
> 利用这个版本链，在并发事务中，可以控制事务访问记录的行为，这种机制就是MVCC（Multi-Version Concurrency Control）。



> **提示**
>
> insert undo日志只在事务回滚时有作用，事务提交后，这类的undo日志就没用了，它占用的`Undo Log Segment`会被系统回收，但记录的`roll_pointer`的值不会被清除。
>
> undo日志中只会记录`索引列、被更新列`的信息，如果在undo日志中没有某列的数据，就表示这列的数据和上版本的数据相同。



### Read View

> 为了控制事务对记录的可见性，InnoDB提出了`Read View`，它的结构是：
>
> 1. `m_ids`：生成Read View时，系统活跃事务的Id列表
> 2. `min_trx_id`：生成Read View时，活跃的`读写事务`中最小的事务Id
> 3. `max_trx_id`：生成Read View时，下一个事务Id
> 4. `creator_id`：生成Read View的事务的Id
>
> `读已提交`、`可重复读`需要保证读取的是已提交的数据，这2个隔离级别会使用到Read View，且它们生成Read View的时机不同：
>
> - `可重复读`：第一次读取数据前
> - `读已提交`：每次读取数据前



> **提示**
>
> 只有事务对表的记录进行`第一次改动`时，才会为这个事务分配事务Id，否则事务Id为0



#### 根据Read View判断记录可见性

> 根据版本中`trx_id`和Read View进行判断
>
> - `trx_id = creator_id`，该版本对当前事务可见
> - `trx_id < min_trx_id`，修改记录的事务`已提交`，该版本对当前事务`可见`
> - `trx_id ≥ max_trx_id`，修改记录的事务在`生成Read View后才开启`，该版本对当前事务`不可见`
> - `min_trx_id < trx_id < max_trx_id`
>     - `trx_id在m_ids中`：修改记录的事务`未提交`，该版本对当前事务`不可见`
>     - `trx_id不在m_ids中`：修改的事务`已提交`，`trx_id在m_ids中`该版本对当前事务`可见`
> - 如果版本对事务不可见，那就根据`roll_pointer`找到上一个版本，继续判断





### 二级索引与MVCC

> 只有聚簇索引中才有`trx_id`、`roll_pointer`，使用`二级索引查询数据`保证可见性的方式是：
>
> 1. 二级索引页面中有属性`page_max_trx_id`记录了对页面进行改动的最大事务Id（事务对页面的记录进行改动时，如果事务的Id大于这个属性，则将属性设置为执行改动的事务Id），如果Read View的这个属性`page_max_trx_id < min_trx_id`，则`页面中的所有记录`对事务可见，否则执行2。
> 2. 根据二级索引主键值进行`回表操作`，在聚簇索引中找到可见的`第一个`版本，在版本中判断`索引列的值`是否`符合查询条件`，如果符合则返回给客户端（还有其他查询条件需继续判断条件是否成立），否则跳过记录。



> **提示**
>
> MVCC是使用隔离级别`可重复读`、`读已提交`的事务执行`普通select`操作时，对版本链的访问的过程。
>
> 在执行`delete`、`update`语句时，并不会立刻把记录从页面中删除，而是进行了`delete mark`操作，这是为了MVCC服务。



## 关于Purge

> 因为update undo日志需要为MVCC服务，所以在事务提交后，不可立刻删除。
>
> undo页面有`Undo Log Header`部分，它有`trx_undo_history_node`属性，是一个名为`History链表`的节点，事务提交后，就会把事务生成的`update undo`日志插入到History链表`头部`。
>
> 在`Rollback Segment Header`的页面中有属性`trx_rseg_history`History链表的基节点、`trx_rseg_history_size`链表占用的页面数量。
>
> 每个回滚段有一个History链表，一个事务在某个回滚段中写入的一组`update undo`日志，在事务提交后，就会被加入到`History链表`，系统有很多回滚段，意味着可能有很多的History链表。
>
> 
>
> 为了支持MVCC，`delete mark`操作不会将记录真正删除，在undo日志中`Undo Log Header`有`trx_undo_del_marks`属性标记本组undo日志是否有`delete mark`操作而产生的undo日志。在合适的时候，就会对`update undo`日志 、`delete mark`产生的undo日志进行`purge`操作。
>
> 只要系统中`最早`产生的ReadView不再访问这些日志，就可以清理这些undo日志。只要保证生成的Read View的事务已经提交了，ReadView就肯定不需要访问该事务生成的undo日志（该事务改动的记录的最新版本均对该Read View可见）。
>
> 
>
> InnoDB做了2件事，事务提交时，会为这个事务生成`事务no`，表示事务提交的顺序。undo日志的`Undo Log Header`有`trx_undo_trx_no`属性，事务提交时，就会将事务no填入这个属性中。History链表是按照事务提交的顺序来排列各组undo日志的（即按照事务no排序的）。
>
> 
>
> Read View还有`事务no`属性，生成Read View时，这个值设置为当前系统`最大事务no+1`。InnoDB还把系统中所有Read View按照创建时间练成一个链表，执行`purge`操作时，会把系统最早的Read View取出来（不存在则新创建一个），然后从回滚段的history链表中取出`事务no`较小的各组undo日志，如果一组undo日志的`事务no`小于最早的Read View的`事务no`，就说明该组undo日志没用了，也就可以在History链表中移除，释放空间，如果该组undo日志有`delete mark`生成的undo日志，就需要彻底删除。
>
> 在`可重复读`的隔离级别下，事务会一直复用最初的Read View，如果事务运行很久，那Read View会一直不释放，这就导致`update undo`日志、`delete mark`生成的undo日志越来越多，表空间对应的文件越来越大，记录的版本链也越来越长，影响系统的性能。
