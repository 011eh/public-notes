# 19 redo日志

## 关键字

> 1. lsn：日志序列号
> 2. flushed_to_disk_lsn：redo日志刷入磁盘的序列号
> 3. checkpoint_lsn：检查点序列号
> 4. 崩溃的恢复
> 5. log_block



> 数据库的事务有一个特性是`持久性`，对于一个已提交的事务，事务对数据库的修改应该是永久的，即使数据库发生故障，这个事务对数据库的修改也不应该丢失。
>
> 我们知道在对数据库的数据进行访问（增删改查）时，需要将对应的页面加载到缓冲区中，对数据进行的修改也是在内存中对页面进行修改，这个修改过的页面不会立刻被刷新到磁盘中。
>
> **持久性**
>
> 为了让提交的事务对数据库的修改是永久的（即使系统发生故障，也可以在重启后进行恢复），可以把`修改的内容记录写入磁盘`（记录某个页面的某个偏移量进行了某个修改，刷新到磁盘），这么做，可以保证持久性。
>
> **优点**
>
> 1. redo日志占用的空间非常小
> 2. redo日志是顺序写入的，执行事务时可能产生多个redo日志，这些日志是`按照产生的顺序写入磁盘`的。



## redo日志的格式

> redo日志只是记录了事务对数据库进行了哪些修改，InnoDB有多种redo日志的格式，大部分的redo日志有一下通用的结构



### 通用的结构

1. type：日志的类型
2. space Id：表空间
3. page nubmer：页号
4. data：redo日志的内容



## 简单的redo日志

> InnoDB会为没有显式定义主键且表中也没有不允许为Null的唯一键的表添加一个`row_id隐藏列作为主键`，服务器会维护一个全局变量，当向某个包含`row_id隐藏列`的表插入记录时，会把这个全局变量的值作为`row_id`。
>
> 这个全局变量为256的倍数时，会被刷新的系统表空间页号为7的页面中（刷新到Max Row Id中），刷新的操作是`在缓冲区完成的`，所以需要将这次对这个页面的修改以redo日志的形式记录下来，保证持久性，  
> `Max Row Id`占用8字节，使用的redo日志的格式是`mlog_8byte`，redo日志`按照写入数据分`有1、2、4、8字节、的格式，还有` mlog_write_string`格式，它需要有`另一部分`记录数据占用的字节数。
>
> InnoDB把这种简单的redo日志称为物理日志



## 复杂的redo日志

> 有些时候，执行一条语句需要修改的页面非常多，可能修改的页面有用户数据页面、系统数据页面，这种情况有：
>
> 1. 表中有多个索引，进行了插入语句后，需要对每个索引进行更新
> 2. 在更新用户记录后，需要对索引进行修改，可能更新叶子节点页面、内节点页面，数据页面还有`File Header`、`Page Directory`等部分
> 3. 插入记录时，需要按照索引列排序，此时需要将上一条记录的`next_record`进行更新



### 记录修改

> 如果使用简单的redo日志记录这些修改，有2种方法
>
> 1. 在每个修改的地方都记录一条redo日志
> 2. 将页面第一个被修改的字节到最后一个被修改的字节之间的数据记录到redo日志中



### 新的redo日志的类型

> redo日志类型与创建对应类型的情景
>
> 1. ` mlog_rec_insert`：插入一条`非紧凑`的行格式的记录
> 2. ` mlog_comp_rec_insert`插入一条`紧凑`的行格式的记录
> 3. ` mlog_comp_page_create`：创建一个存储紧凑行格式记录的页面
> 4. ` mlog_comp_rec_delete`：删除一条`紧凑`的行格式的记录
> 5. ` mlog_comp_list_start_delete`：从某一条记录开始删除页面的一系列紧凑行格式的记录
> 6. ` mlog_comp_end_delete`：与上面对应，删除到这个日志中对应的记录
> 7. ` mlog_zip_page_compress`：在压缩一个数据页时
>
> 
>
> **redo日志物理层面和逻辑层面**
>
> - 物理层面：日志记录了对哪些页面的修改
> - 逻辑层面：在使用redo日志进行恢复时，对于页面的一些数据，不能直接根据日志对其进行修改，而是需要执行函数进行处理



### mlog_comp_rec_insert日志类型

> **结构**
>
> 1. type
> 2. space Id
> 3. page number
> 4. n_fields：该记录的字段数
> 5. n_uniques：能表示该记录唯一的字段数（对于普通索引，这个值是索引列数+主键列数）
> 6. `field 1_len`~`field n_len`：各个字段占用的空间大小（就算是可变长类型的数据也要记录）
> 7. offset：前一条记录的偏移位置，需要修改上一条记录的`next_record`
> 8. end_seg_len：可以计算出当前记录占用空间的大小
> 9. 记录头信息
> 10. extra_size：额外信息占用的空间大小
> 11. mismatch：能节省redo日志的大小
> 12. 记录真实的数据
>
> 
>
> **关于redo日志的逻辑层面**
>
> 以上的redo日志中，并没有关于`page_n_dir_slots`等值进行修改，而只是将页面要插入的记录的必要信息记录下来，当使用redo日志进行恢复时，服务器就要调用向某个页面插入记录的函数，完成这些相关数据的修改。



## Mini-Transaction

### 以组的形式写入redo日志

> 一条语句可能会修改多个页面，对页面的修改是在缓冲区进行的，修改页面后，就需要记录redo日志，在执行语句的过程中产生的redo日志，被InnoDB划分为`不可分割的组`，划分成一个组的情况有：
>
> 1. 更新系统表空间页面为7的属性`Max Row ID`产生的redo日志
> 2. 对索引页面插入一条记录时产生的redo日志
> 3. 其他
>
> 
>
> **例子**
>
> 在向某个索引插入一条记录时，需要先`定位`到应该将这条记录插入的`叶子节点对于的数据页`，在插入时可能有2中情况
>
> - 数据页还有剩余的空间，这样只需要插入这条记录
>
> - 数据剩余的空间不足这时就需要进行页分裂操作
>
>     1. 新建一个叶子节点，将原页面的部分数据复制到这个新的页面
>     2. 将这条记录插入到相应位置
>     3. 将新页面插入到页面的链表中
>     4. 在内节点添加一个目录项记录指向新页面
>
>     - 这个过程会对`多个页面进行修改`，会产生`多条redo日志`,还会修改段、区的统计数据、链表信息等



> InnoDB认为在向某个索引对应的`B+树插入一条记录`的过程必须是`原子的`，在执行这些需要保证原子性的操作时，必须以`组的形式`记录redo日志。



### 实现

> 对于那些需要保证原子性操作产生的多个redo日志，需要以组的形式记录redo日志，InnoDB在一个组的最后一条redo日志`后面`添加一个`特殊类型的redo日志`，只有一个`type`。
>
> 这样在进行系统恢复时，只有解析到这个类型的redo日志，才进行恢复，否则放弃恢复。
>
> 
>
> 对于需要保证原子性操作只产生一个redo日志，InnoDB会在redo日志中`type的第一个比特位标识`是否是`单一的日志`



## Mini-Transaction概念

> MySQL把对底层页面进行一次原子访问的过程称为MTR。
>
> - 一个事务包含多个语句
> - 一个语句包含多个MTR
> - 一个MTR包含多个redo日志



## redo日志的写入过程

### redo日志页结构

> InnoDB把通过MTR生成的redo日志放到了页（大小512B）中进行管理，存储redo日志的页的结构是
>
> 1. log block header，12字节，存储管理信息
> 2. log body，496字节，存储redo日志信息
> 3. log block trailer，4字节，存储管理信息



> log block header的结构
>
> 1. log_block_hdr_no：每个block都有大于0的唯一编号
> 2. log_block_hdr_data_len：block已经使用的字节数，初始值为12（header部分）
> 3. log_block_first_rec_group：block中的第一个MTR生成的第一个redo日志记录的偏移量（一个block可能被多个MTR生成的redo日志占用）
> 4. log_block_checkpoint_no：checkpoint序号
>
> 
>
> log block trailer的结构
>
> 1. log_block_checksum：block的校验值



## redo日志缓冲区

> 写入redo日志也一样不能直接写入磁盘，服务器在启动时会向操作系统申请连续的内存空间作为redo日志的缓冲区（log buffer，默认大小16M），这些内存存储了redo日志 block。



### redo日志写入日志缓冲区

> 向日志缓冲区写入redo日志是顺序写入的，会先向缓冲区前面的block中，当前的block空间使用完后，再往下一个block中写，InnoDB提供了`buf_free`全局变量，指明后续写入的redo日志应该写入到缓冲区的哪个位置。
>
> 一个MTR执行过程可能产生多个redo日志，这些日志是一个不可分割的组，所以在MTR执行完成前，这些日志是暂存在一个地方，当MTR结束后，这个过程产生的一组redo日志，会被`复制到redo日志缓冲区`。
>
> 
>
> **事务的并发**
>
> 不同事务可能是并发执行的，`事务中的多个MTR`生成的对于的`redo日志组`可能是被`交替写入`到日志缓冲区中。（事务中的MTR生成的日志组可能在缓冲区中不连续）



## redo日志文件

### redo日志刷盘时机

> **日志缓冲区空间不足**
>
> redo日志占用日志缓冲区总空间的50%时
>
> 
>
> **事务提交**
>
> 为了保证事务的持久性，需要在`事务提交前`把页面修改对应的redo日志写入磁盘
>
> 
>
> **脏页被刷新前**
>
> redo日志是`顺序刷新`的，它会保证将`其之前产生的redo日志`也刷新到磁盘
>
> 
>
> **后台的线程**
>
> 后台的线程大约每秒将日志缓冲区的redo日志写入磁盘
>
> 
>
> **服务器关闭**
>
> 服务器正常关闭
>
> 
>
> **做checkpoint前**



### redo日志文件组

> MySQL的数据目录默认有2个文件`ib_logfile0`、`ib_logfile1`，日志缓冲区在默认情况下是刷新到这2个磁盘文件中，可以通过启动选项进行相关的修改
>
> - `innodb_log_group_home_dir`：redo日志文件的目录
> - ` innodb_log_file_size`：redo日志文件的大小，默认48M
> - `innodb_log_files_in_group`：redo日志文件的个数，默认为2，最大值为100
> - redo日志文件的总大小：单个文件大小×文件个数
>
> redo日志文件不止1个，在将redo日志写入文件时，会从`ib_logfile0`开始，然后依次向后续的文件写入，当文件都写满后，会`回到ib_logfile0`继续写入。



### redo日志文件的格式

> 每个redo日志文件的大小、格式都一样，文件由2部分组成：
>
> - 前2048字节（前4个block的格式）用来存储一些管理信息
> - 第2048字节往后的字节用来存储log buffer的block镜像
> - 对文件的写入，是从每个文件的第2048个字节开始的



#### 前4个block的格式

> redo日志文件的前4个block的格式（前2048字节）
>
> 1. log file header
> 2. checkpoint1
> 3. 没用
> 4. checkpoint2
>
> 
>
> log file header的格式
>
> 1. log_header_format：redo日志版本，为1
> 2. log_header_pad1：字符填充
> 3. log_header_start_lsn：标记本redo日志文件偏移量为2048字节处的lsn值
> 4. log_header_creator：日志文件的创建者，正常情况下是MySQL的版本号
> 5. log_block_checksum：block的校验和
>
> 
>
> checkpoint1、2的格式
>
> 1. log_checkpoint_no：服务器执行checkpoint的编号，每执行一次checkpoint，就+1
> 2. log_checkpoint_lsn：服务器在结束checkpoint时对应的lsn值；系统在崩溃后恢复将从该值开始
> 3. log_checkpoint_offset：lsn值在redo日志文件组中的偏移量
> 4. log_checkpoint_log_buf_size：执行checkpoint操作时对应的日志缓冲区的大小
> 5. log_block_checksum：block的校验和



## log sequence number

> 系统运行后，会不断修改页面，不断产生redo日志，InnoDB设计了全局变量`lsn`来记录已经写入的redo日志量，`lsn`的初始值是8704，在统计lsn的`增长量`时，是按照3部分计算的：
>
> 1. redo日志量
> 2. header
> 3. trailer
>
> 在系统第一次启动时，全局变量`buf_free`指向第一个block偏移12字节的位置（header占用12字节，即指向block body开始的位置），此时`lsn会增加12`。
>
> MTR产生的redo日志组占用的空间可能会占用一部分block body或多个block，lsn增长量可能会包括block的header、trailer字节数。



### flush_to_disk_lsn

> redo日志是写在日志缓冲区的，之后才会被刷新到磁盘，InnoDB提出了`buf_next_to_write`的全局变量，标记日志缓冲区中已经被写入磁盘的redo日志。
>
> InnoDB提出了`flush_to_disk_lsn`表示已经刷新到磁盘的redo日志量（系统第一次启动时，是8704），随着日志被写入日志缓冲区，`lsn`就与`flush_to_disk_lsn`拉开了差距。如果它们的值相等，就表示日志缓冲区的日志都被写入磁盘中。



### lsn与redo日志文件组中的偏移量关系

> lsn的值代表系统写入redo日志量的总和，redo日志被刷新到磁盘后，可以计算出`lsn`在redo日志文件组中的偏移量（lsn为8704，就是在日志文件组的偏移量2048，即偏移4个管理信息的block，记录真正日志block的开始）。



### flush链表的lsn

> MTR结束后，不仅需要将redo日志写入缓冲区，还需要将`被修改的页面`加入到`flush链表`。
>
> 第一次修改缓冲区的页面时，会将页面对应的`控制块`加入到flush链表头部（之后修改不会移动到头部），在控制块中记录页面的修改：
>
> 1. oldest_modification：第一修改页面时，记录MTR`开始`时的`lsn`
> 2. newest_modification：每次修改页面，记录MTR`结束`时的`lsn`
>
> 在flush链表中，第一次修改比较晚的页面的`控制块`在链表前面



### checkpoint

> redo日志文件组的容量时是有限的，我们需要循环利用redo日志文件组中的文件（覆盖日志文件组中的文件），redo日志文件是为了在系统崩溃后恢复，如果对应的`脏页已经刷新到磁盘`，崩溃恢复也就不需要用到这些相关的redo日志，所以这些redo日志是可以被覆盖的。
>
> InnoDB提出了`checkpoint_lsn`表示当前系统`可以覆盖`的redo日志总量
>
> 如果某个脏页被刷新到磁盘后，与它对应的redo日志也就可以被覆盖了，所以可以进行一次增加checkpoint_lsn的操作
>
> 一般来说，刷新脏页、进行checkpoint操作在不同的线程执行的，并不是每次执行脏页刷新都进行checkpoint操作。



#### checkpoint过程

> 1. 计算当前系统可以覆盖的redo日志的`lsn`：将flush链表`尾节点`的`oldest_modification`复制给`checkpoint_lsn`（在这一刻，尾节点是最早被修改的脏页对应的控制块）。
> 2. 将管理信息写入到`redo日志文件组`（这些信息只会写入日志文件组的第一个文件中）
>     - `checkpoint_no`：记录进行了多少次checkpoint
>     - `checkpoint_lsn`
>     - `checkpoint_offset`：redo日志文件组的偏移量
>
> checkpoint为偶数就记录到文件关联信息中的checkpoint1，否则记录到checkpoint2。



## 用户线程批量从flush链表刷出脏页

> 如果系统刷新页面的操作十分频繁，会使得`写redo日志`的操作十分频繁，后台线程不能及时将脏页刷新，系统就不能及时执行`checkpoint`，这可能需要用户线程将flush链表的脏页刷新到磁盘中，这时才会进行checkpoint操作。



## 查看系统lsn

```sql
show engine innodb status\G;
```

```bash
# log部分
---
LOG
---
# lsn
Log sequence number 29406140

# flush_to_disk_lsn
Log flushed up to   29406140 

# flush链表最早修改页面的oldest_modification
Pages flushed up to 29406140 

# 当前系统的checkpoint_lsn
Last checkpoint at  29406131 
0 pending log flushes, 0 pending chkp writes
527 log i/o's done, 0.00 log i/o's/second
```



## 全局变量innodb_flush_log_at_trx_commit

> 指明了redo日志的刷新策略，全局变量的值有：
>
> - `0`：事务提交后，不立刻将redo日志同步到磁盘
> - `1`：事务提交后，将redo日志刷新到磁盘，默认值
> - `2`：将redo日志写到`操作系统缓冲区`（服务器崩溃，但操作系统正常，事务的持久性仍可以保证）



## 崩溃恢复

### 确定恢复的起点

> `lsn`小于`checkpoint_lsn`的redo日志来说，它们是可以被覆盖的（因为与它们对应的脏页被刷新到磁盘中），对于这部分的redo日志，我们不需要使用它们进行恢复。
>
> `lsn`大于`checkpoint_lsn`的redo日志来说，我们`不清楚`与它们对应的脏页`是否被刷新到磁盘中`（磁盘刷新大部分时候是异步的），所以就需要从`lsn`为`checkpoint_lsn`的redo日志开始恢复。
>
> 从redo日志文件的管理信息中（checkpoint1、checkpoint2）找出checkpoint_lsn较大的那个checkpoint，在这个block中得到日志文件组的偏移量`checkpoint_offest`。



### 确定恢复的终点

> 存储redo日志的block header中有`log_block_hdr_data_len`，记录当前block使用了多少字节，被填满的block的值是512，没被填满的block的值小于512。所以结束恢复的block就是对应的`log_block_hdr_data_len`小于512的那一个。



### 恢复的方式

> 按照redo日志的的顺序依次扫描checkpoint_lsn之后的redo日志进行恢复。
>
> InnoDB通过一些措施加快了恢复的过程：
>
> **哈希表**
>
> - 根据redo日志的`space Id`、`page number`作为`哈希值`，将它们相同的redo日志放到同一个槽中，有如果有多个相同的redo日志就`按照生成的顺序`使用链表组织。
> - 对同一个页面的redo日志都放到了一个槽中，可以一次性将一个页面修复好减少了读取页面的随机IO。
>
> 
>
> **跳过已经刷新到页面**
>
> 我们不能判断`lsn`大于`checkpoint_lsn`的redo日志对应的脏页是否被刷新到磁盘中（在执行了checkpoint后，后台线程可能将`LRU链表`、`flush链表`中的脏页刷新到磁盘）。
>
> 可以根据页面中的`File Header`部分的属性`fil_page_lsn`（最近修改页面时的lsn，控制块中的`newest_modification`）判断。



## 关于存储redo日志的block中的header

> log block中的部分`log block header`中有属性`log_block_hdr_no`，这个属性代表block的唯一编号，在初次使用block时进行分配，他的值与当时的lsn有关：`((lsn/512) & 0x3FFFFFFF) + 1`，这个编号数值的范围是0~2<sup>30</sup>（0x3FFFFFFF是2<sup>30</sup>-1）。
>
> InnoDB规定redo日志文件组的所有文件的大小不得超过`512GB`，一个block的大小为`512B`，所以日志文件组的block的个数不超过`1G`个。

