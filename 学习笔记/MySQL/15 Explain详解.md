# 15 Explain详解

## 表结构

```sql
create table single_table (
    id           int auto_increment primary key,
    key1         varchar(100) null,
    key2         int          null,
    key3         varchar(100) null,
    key_part1    varchar(100) null,
    key_part2    varchar(100) null,
    key_part3    varchar(100) null,
    common_field varchar(100) null,
    constraint uk_key2
        unique (key2)
);

create index idx_key1
    on single_table (key1);

create index idx_key3
    on single_table (key3);

create index idx_part
    on single_table (key_part1, key_part2, key_part3);
```



​		MySQL优化器基于成本、规则对查询语句进行优化后，就会生成一个执行计划，使用`explain`语句可以查看某个查询语句的执行计划

```sql
explain select 1;
```

| id   | select\_type | table | partitions | type | possible\_keys | key  | key\_len | ref  | rows | filtered | Extra          |
| :--- | :----------- | :---- | :--------- | :--- | :------------- | :--- | :------- | :--- | :--- | :------- | :------------- |
| 1    | SIMPLE       | NULL  | NULL       | NULL | NULL           | NULL | NULL     | NULL | NULL | NULL     | No tables used |



## explain表格解释

| 列名               | 描述                                                       |
| :----------------- | :--------------------------------------------------------- |
| **id**             | 每个`select`关键字对应一个唯一Id                           |
| **select\_type**   | select关键字对应的查询类型                                 |
| **table**          | 对应单表访问的表名                                         |
| **partitions**     | 匹配的分区信息，不介绍                                     |
| **type**           | 对单表进行访问的`访问方法`                                 |
| **possible\_keys** | 查询可能用到的索引                                         |
| **key**            | 实际使用的索引                                             |
| **key\_len**       | 实际使用索引的长度                                         |
| **ref**            | 使用索引列进行等值匹配时，与索引列进行匹配的信息           |
| **rows**           | 预估读取记录的条数                                         |
| **filtered**       | 对预计读取的记录，进行查询条件过滤后，满足条件的记录的比值 |
| **Extra**          | 执行计划的额外信息                                         |



## id

> 查询语句可能出现多个`select`子句，每出现一个`select`子句，MySQL都会为他分配一个唯一的Id。



> 对于连接查询来说，一个`select`子句后面的`from`子句可以跟着多个表，在`连接查询的执行计划中`，每个表都对应一条记录，但这些记录的`id`是相同的，出现在前面的记录对应的表是`驱动表`

```sql
explain
select *
from single_table s1
         join single_table s2;
```

|                    |        |                                         |
| :----------------- | :----- | :-------------------------------------- |
| **id**             | `1`    | `1`                                     |
| **select\_type**   | SIMPLE | SIMPLE                                  |
| **table**          | s1     | s2                                      |
| **partitions**     | NULL   | NULL                                    |
| **type**           | ALL    | ALL                                     |
| **possible\_keys** | NULL   | NULL                                    |
| **key**            | NULL   | NULL                                    |
| **key\_len**       | NULL   | NULL                                    |
| **ref**            | NULL   | NULL                                    |
| **rows**           | 9937   | 9937                                    |
| **filtered**       | 100    | 100                                     |
| **Extra**          | NULL   | Using join buffer \(Block Nested Loop\) |



> 包含子查询的查询语句，就可能涉及到多个select关键字，所以在包含子查询的查询语句的执行计划中，每个`select`子句都对应一个唯一id。

```sql
explain
select *
from single_table s1
where key1 in (select key1 from single_table s2)
   or key3 = 'a';
```

|                    |             |             |
| :----------------- | :---------- | :---------- |
| **id**             | `1`         | `2`         |
| **select\_type**   | PRIMARY     | SUBQUERY    |
| **table**          | s1          | s2          |
| **partitions**     | NULL        | NULL        |
| **type**           | ALL         | index       |
| **possible\_keys** | idx\_key3   | idx\_key1   |
| **key**            | NULL        | idx\_key1   |
| **key\_len**       | NULL        | 303         |
| **ref**            | NULL        | NULL        |
| **rows**           | 9937        | 9937        |
| **filtered**       | 100         | 100         |
| **Extra**          | Using where | Using index |



> 优化器可能会对涉及子查询的查询语句进行重写，从而转换为`半连接`

```sql
explain
select *
from single_table s1
where key1 in (select common_field from single_table s2 where s1.key3 = 'a');
```

|                    |                              |                             |
| :----------------- | :--------------------------- | :-------------------------- |
| **id**             | `1`                          | `1`                         |
| **select\_type**   | SIMPLE                       | SIMPLE                      |
| **table**          | s2                           | s1                          |
| **partitions**     | NULL                         | NULL                        |
| **type**           | ALL                          | ref                         |
| **possible\_keys** | NULL                         | idx\_key1,idx\_key3         |
| **key**            | NULL                         | idx\_key1                   |
| **key\_len**       | NULL                         | 303                         |
| **ref**            | NULL                         | mysql\_run.s2.common\_field |
| **rows**           | 9937                         | 2                           |
| **filtered**       | 100                          | 2.1                         |
| **Extra**          | Using where; Start temporary | Using where; End temporary  |



> 使用union时，会把多个查询得到的结果进行去重操作，MySQL会在内部创建一个`临时表`。
>
> id为`Null`，表明这个临时表是为了合并两个查询结果而创建的。

```sql
explain
select *
from single_table s1
union
select *
from single_table s2;
```

|                    |         |       |                    |
| :----------------- | :------ | :---- | :----------------- |
| **id**             | 1       | 2     | `NULL`             |
| **select\_type**   | PRIMARY | UNION | UNION RESULT       |
| **table**          | s1      | s2    | &lt;`union1,2`&gt; |
| **partitions**     | NULL    | NULL  | NULL               |
| **type**           | ALL     | ALL   | ALL                |
| **possible\_keys** | NULL    | NULL  | NULL               |
| **key**            | NULL    | NULL  | NULL               |
| **key\_len**       | NULL    | NULL  | NULL               |
| **ref**            | NULL    | NULL  | NULL               |
| **rows**           | 9937    | 9937  | NULL               |
| **filtered**       | 100     | 100   | NULL               |
| **Extra**          | NULL    | NULL  | Using temporary    |



> 使用`union all`，不会对结果集进行去重，不需要创建临时表

```sql
explain
select *
from single_table s1
union all
select *
from single_table s2;
```

|                    |         |       |
| :----------------- | :------ | :---- |
| **id**             | 1       | 2     |
| **select\_type**   | PRIMARY | UNION |
| **table**          | s1      | s2    |
| **partitions**     | NULL    | NULL  |
| **type**           | ALL     | ALL   |
| **possible\_keys** | NULL    | NULL  |
| **key**            | NULL    | NULL  |
| **key\_len**       | NULL    | NULL  |
| **ref**            | NULL    | NULL  |
| **rows**           | 9937    | 9937  |
| **filtered**       | 100     | 100   |
| **Extra**          | NULL    | NULL  |



## table

> 不管查询语句有多复杂，查询中有多少个表，最终也是对`每个表`进行单表访问



## type

> 对表进行查询的访问方法,下方的SQL语句中，对表进行的方法是`ref`

```sql
explain
select *
from single_table
where key1 = 'a';
```

|                    |               |
| :----------------- | :------------ |
| **id**             | 1             |
| **select\_type**   | SIMPLE        |
| **table**          | single\_table |
| **partitions**     | NULL          |
| **type**           | `ref`         |
| **possible\_keys** | idx\_key1     |
| **key**            | idx\_key1     |
| **key\_len**       | 303           |
| **ref**            | const         |
| **rows**           | 138           |
| **filtered**       | 100           |
| **Extra**          | NULL          |



### 访问方法

1. system
2. const
3. eq_ref
4. ref
5. fulltext
6. ref_or_null
7. index_merge
8. unique_subquery
9. index_subquery
10. range
11. index
12. ALL



#### system

>表中只有`一条记录`，并且该表使用的存储引擎的`统计数据是精确`的（MyISAM等）



#### const

>对表进行访问时，使用到了`主键索引`或`唯一索引`与常数进行`等值匹配`



#### eq_ref

>执行连接查询时，被驱动表是通过`主键`或`不允许为Null的唯一索引`（联合索引要求所有列都进行等值匹配）进行等值匹配的方式进行访问

```sql
explain
select *
from single_table s1
         join single_table s2 on s1.id = s2.id;
```

|                    |         |                   |
| :----------------- | :------ | :---------------- |
| **id**             | 1       | 1                 |
| **select\_type**   | SIMPLE  | SIMPLE            |
| **table**          | s1      | s2                |
| **partitions**     | NULL    | NULL              |
| **type**           | ALL     | `eq_ref`          |
| **possible\_keys** | PRIMARY | PRIMARY           |
| **key**            | NULL    | `PRIMARY`         |
| **key\_len**       | NULL    | 4                 |
| **ref**            | NULL    | `mysql_run.s1.id` |
| **rows**           | 9937    | 1                 |
| **filtered**       | 100     | 100               |
| **Extra**          | NULL    | NULL              |



#### ref

>使用普通索引与常量进行等值匹配的方式查询表。
>
>如果执行的是连接查询，被驱动表上的普通索引与驱动表的某一个列进行等值匹配时，也可能使用到`ref`访问方法
>
>ref比range更容易使用索引，因为ref是索引与常量比较，对于符合条件的记录，他们的主键值是有序的，所以:
>
>1. 前一次回表的Id和下一次回表的Id很可能在同一个页面
>2. 即使主键值不在同一个页面，单主键值是递增的，也很可能通过顺序IO找到下一个数据页，这将减少随机IO带来的开销，回表的性能开销小

```sql
explain
select *
from single_table s1
         join single_table s2 on s1.key1 = s2.key1;
```

|                    |             |                     |
| :----------------- | :---------- | :------------------ |
| **id**             | 1           | 1                   |
| **select\_type**   | SIMPLE      | SIMPLE              |
| **table**          | s1          | s2                  |
| **partitions**     | NULL        | NULL                |
| **type**           | ALL         | ref                 |
| **possible\_keys** | idx\_key1   | idx\_key1           |
| **key**            | NULL        | `idx_key1`          |
| **key\_len**       | NULL        | 303                 |
| **ref**            | NULL        | `mysql_run.s1.key1` |
| **rows**           | 9937        | 2                   |
| **filtered**       | 100         | 100                 |
| **Extra**          | Using where | NULL                |



#### ref_or_null

>使用普通索引进行等值匹配且索引列允许为`null`

```sql
explain
select *
from single_table
where key1 = 'a'
   or key1 is null;
```

|                    |                       |
| :----------------- | :-------------------- |
| **id**             | 1                     |
| **select\_type**   | SIMPLE                |
| **table**          | single\_table         |
| **partitions**     | NULL                  |
| **type**           | `ref_or_null`         |
| **possible\_keys** | idx\_key1             |
| **key**            | idx\_key1             |
| **key\_len**       | 303                   |
| **ref**            | const                 |
| **rows**           | 1144                  |
| **filtered**       | 100                   |
| **Extra**          | Using index condition |



#### index_merge

>索引合并，索引合并有交集、并集、排序并集

```sql
explain
select *
from single_table
where key1 = 'a'
   or key3 = 'a';
```

|                    |                                               |
| :----------------- | :-------------------------------------------- |
| **id**             | 1                                             |
| **select\_type**   | SIMPLE                                        |
| **table**          | single\_table                                 |
| **partitions**     | NULL                                          |
| **type**           | `index_merge`                                 |
| **possible\_keys** | idx\_key1,idx\_key3                           |
| **key**            | `idx_key1,idx_key3`                           |
| **key\_len**       | 303,303                                       |
| **ref**            | NULL                                          |
| **rows**           | 295                                           |
| **filtered**       | 100                                           |
| **Extra**          | `Using union(idx_key1,idx_key3)`; Using where |



#### unique_subquery（转换成 exists使用唯一索引）

>类似两表查询中被驱动表的eq_ref访问方法，它针对的是包含`in子查询`的查询语句，优化器决定将`in`子查询转换为`exists`子查询，并且子查询转换之后，可以使用`主键`或`不允许为null`的唯一索引

```sql
explain
select *
from single_table s1
where common_field in (select id from single_table s2 where s1.common_field = s2.common_field)
   or key3 = 'a';
   
# 转换为exists子查询
explain
select *
from single_table s1
where exists(select id from single_table s2 where s1.common_field = s2.common_field and s1.common_field = s2.id)
   or key3 = 'a';
```

|                    |             |                    |
| :----------------- | :---------- | :----------------- |
| **id**             | 1           | 2                  |
| **select\_type**   | PRIMARY     | DEPENDENT SUBQUERY |
| **table**          | s1          | s2                 |
| **partitions**     | NULL        | NULL               |
| **type**           | ALL         | `unique_subquery`  |
| **possible\_keys** | idx\_key3   | PRIMARY            |
| **key**            | NULL        | PRIMARY            |
| **key\_len**       | NULL        | 4                  |
| **ref**            | NULL        | func               |
| **rows**           | 9937        | 1                  |
| **filtered**       | 100         | 10                 |
| **Extra**          | Using where | Using where        |



#### index_subquery（转换成 exists使用二级索引）

>访问子查询的表时，使用的是普通索引，转换为`exists`

```sql
explain
select *
from single_table s1
where common_field in (select key3 from single_table s2 where s1.common_field = s2.common_field)
   or key3 = 'a';
```

|                    |             |                    |
| :----------------- | :---------- | :----------------- |
| **id**             | 1           | 2                  |
| **select\_type**   | PRIMARY     | DEPENDENT SUBQUERY |
| **table**          | s1          | s2                 |
| **partitions**     | NULL        | NULL               |
| **type**           | ALL         | `index_subquery`   |
| **possible\_keys** | idx\_key3   | idx\_key3          |
| **key**            | NULL        | `idx_key3`         |
| **key\_len**       | NULL        | 303                |
| **ref**            | NULL        | func               |
| **rows**           | 9937        | 2                  |
| **filtered**       | 100         | 10                 |
| **Extra**          | Using where | Using where        |



#### range

>使用索引进行查询，扫描区间是一些单点扫描区间或范围区间

```sql
explain select *
from single_table
where key1 in ('a', 'b');
```

|                    |                       |
| :----------------- | :-------------------- |
| **id**             | 1                     |
| **select\_type**   | SIMPLE                |
| **table**          | single\_table         |
| **partitions**     | NULL                  |
| **type**           | `range`               |
| **possible\_keys** | idx\_key1             |
| **key**            | idx\_key1             |
| **key\_len**       | 303                   |
| **ref**            | NULL                  |
| **rows**           | 274                   |
| **filtered**       | 100                   |
| **Extra**          | Using index condition |



#### index

>使用到`索引覆盖`，且`扫描所有的索引`
>
>因为普通索引的记录只包含索引列、主键值，聚簇索引记录包含用户定义的所有列和隐藏字段，扫描所有普通索引的成本比扫描全部聚簇索引的成本低一些

```sql
explain
select key_part1, key_part2
from single_table
where key_part3 = 'b';
```

|                    |                          |
| :----------------- | :----------------------- |
| **id**             | 1                        |
| **select\_type**   | SIMPLE                   |
| **table**          | single\_table            |
| **partitions**     | NULL                     |
| **type**           | `index`                  |
| **possible\_keys** | NULL                     |
| **key**            | `idx_part`               |
| **key\_len**       | `909`                    |
| **ref**            | NULL                     |
| **rows**           | 9937                     |
| **filtered**       | 10                       |
| **Extra**          | Using where; Using index |



## select_type

> MySQL为每一个select关键字代表的小查询都定义了`select_type`，根据select_type，我们可以知道这个select关键字的`小查询的类型`

### 类型

1. simple
2. primary
3. union
4. union result
5. subquery
6. dependent subquery
7. dependent union
8. derived
9. materialized
10. uncacheable subquery
11. uncacheable union



### simple

> 不包含子查询或union的查询、连接查询的select_type是simple



### primary

> 对于包含子查询、union、union all的查询语句，最左边的查询就是primary



### union

> 对于包含union、union all的查询语句，除了最左边的查询，其余都是小查询的类型都是union

```sql
explain
select *
from single_table s1
union
select *
from single_table s2;
```

|                    |           |         |                  |
| :----------------- | :-------- | :------ | :--------------- |
| **id**             | 1         | 2       | NULL             |
| **select\_type**   | `PRIMARY` | `UNION` | UNION RESULT     |
| **table**          | s1        | s2      | &lt;union1,2&gt; |
| **partitions**     | NULL      | NULL    | NULL             |
| **type**           | ALL       | ALL     | ALL              |
| **possible\_keys** | NULL      | NULL    | NULL             |
| **key**            | NULL      | NULL    | NULL             |
| **key\_len**       | NULL      | NULL    | NULL             |
| **ref**            | NULL      | NULL    | NULL             |
| **rows**           | 9937      | 9937    | NULL             |
| **filtered**       | 100       | 100     | NULL             |
| **Extra**          | NULL      | NULL    | Using temporary  |



### union result

> MySQL使用临时表完成union子句的去重操作

```sql
explain
select *
from single_table s1
union
select *
from single_table s2;
```

|                    |         |       |                  |
| :----------------- | :------ | :---- | :--------------- |
| **id**             | 1       | 2     | NULL             |
| **select\_type**   | PRIMARY | UNION | `UNION RESULT`   |
| **table**          | s1      | s2    | &lt;union1,2&gt; |
| **partitions**     | NULL    | NULL  | NULL             |
| **type**           | ALL     | ALL   | ALL              |
| **possible\_keys** | NULL    | NULL  | NULL             |
| **key**            | NULL    | NULL  | NULL             |
| **key\_len**       | NULL    | NULL  | NULL             |
| **ref**            | NULL    | NULL  | NULL             |
| **rows**           | 9937    | 9937  | NULL             |
| **filtered**       | 100     | 100   | NULL             |
| **Extra**          | NULL    | NULL  | Using temporary  |



### subquery

> 包含子查询的查询语句`不能转换为半连接`，并且子查询是`不相关子查询`，优化器决定采用`将该子查询物化`的方案来执行该子查询时，该子查询的select_type是subquery

```sql
explain
select *
from single_table s1
where key1 in (select key1 from single_table s2)
   or key3 = 'a';
```

|                    |             |             |
| :----------------- | :---------- | :---------- |
| **id**             | 1           | 2           |
| **select\_type**   | PRIMARY     | `SUBQUERY`  |
| **table**          | s1          | s2          |
| **partitions**     | NULL        | NULL        |
| **type**           | ALL         | index       |
| **possible\_keys** | idx\_key3   | idx\_key1   |
| **key**            | NULL        | idx\_key1   |
| **key\_len**       | NULL        | 303         |
| **ref**            | NULL        | NULL        |
| **rows**           | 9937        | 9937        |
| **filtered**       | 100         | 100         |
| **Extra**          | Using where | Using index |



### dependent subquery

> 包含子查询的查询语句`不能转换为半连接`，并且子查询是`相关子查询`，子查询的第一个`select`关键字代表的查询的select_type是dependent subquery
>
> select_type为dependent subquery的子查询可能会被`执行多次`

```sql
explain
select *
from single_table s1
where key1 in (select key1 from single_table s2 where s1.key2 = s2.key2)
   or key3 = 'a';
```

|                    |             |                      |
| :----------------- | :---------- | :------------------- |
| **id**             | 1           | 2                    |
| **select\_type**   | PRIMARY     | `DEPENDENT SUBQUERY` |
| **table**          | s1          | s2                   |
| **partitions**     | NULL        | NULL                 |
| **type**           | ALL         | ref                  |
| **possible\_keys** | idx\_key3   | uk\_key2,idx\_key1   |
| **key**            | NULL        | uk\_key2             |
| **key\_len**       | NULL        | 5                    |
| **ref**            | NULL        | mysql\_run.s1.key2   |
| **rows**           | 9937        | 1                    |
| **filtered**       | 100         | 10                   |
| **Extra**          | Using where | Using where          |



### dependent union

> 包含union或union all的查询语句，如果`各个小查询都依赖于外层查询`，那么`除了最左边`的小查询之外，其余的小查询的select_type都是dependent union
>
> 下方的查询语句中，`select key1 from single_table s2 where key1 = 'a'`是最左边的小查询，它的select_type是dependent subquery,
>
> 而`select key1 from single_table s1 where key3 = 'a'`的select_type是dependent union

```sql
explain
select *
from single_table s1
where key1 in
      (select key1 from single_table s2 where key1 = 'a' union select key1 from single_table s1 where key3 = 'a');
```

|                    |             |                          |                          |                  |
| :----------------- | :---------- | :----------------------- | :----------------------- | :--------------- |
| **id**             | 1           | 2                        | 3                        | NULL             |
| **select\_type**   | PRIMARY     | DEPENDENT SUBQUERY       | `DEPENDENT UNION`        | UNION RESULT     |
| **table**          | s1          | s2                       | single\_table            | &lt;union2,3&gt; |
| **partitions**     | NULL        | NULL                     | NULL                     | NULL             |
| **type**           | ALL         | ref                      | ref                      | ALL              |
| **possible\_keys** | NULL        | idx\_key1                | idx\_key1                | NULL             |
| **key**            | NULL        | idx\_key1                | idx\_key1                | NULL             |
| **key\_len**       | NULL        | 303                      | 303                      | NULL             |
| **ref**            | NULL        | const                    | func                     | NULL             |
| **rows**           | 9937        | 138                      | 2                        | NULL             |
| **filtered**       | 100         | 100                      | 100                      | NULL             |
| **Extra**          | Using where | Using where; Using index | Using where; Using index | Using temporary  |



### derived

> 包含派生表的查询中，以物化表的方式进行查询，派生表对应的子查询的select_type是dervied

```sql
explain
select *
from (select key1, count(*) from single_table group by key1) t;
```

|                    |                  |               |
| :----------------- | :--------------- | :------------ |
| **id**             | 1                | 2             |
| **select\_type**   | PRIMARY          | `DERIVED`     |
| **table**          | &lt;derived2&gt; | single\_table |
| **partitions**     | NULL             | NULL          |
| **type**           | ALL              | index         |
| **possible\_keys** | NULL             | idx\_key1     |
| **key**            | NULL             | idx\_key1     |
| **key\_len**       | NULL             | 303           |
| **ref**            | NULL             | NULL          |
| **rows**           | 9937             | 9937          |
| **filtered**       | 100              | 100           |
| **Extra**          | NULL             | Using index   |



### materialized

> 对于包含子查询的查询语句，优化器选择将子查询物化后与外层查询进行连接，子查询的`select_type`就是materialized
>
> 下面的执行计划中，查询优化器把子查询转换为物化表，第二条记录中，的table是`<subquery2>`，这个表是id为2的子查询进行查询后产生的物化表，然后在与s1表进行连接

```sql
explain
select *
from single_table s1
where key1 in (select key1 from single_table s2);
```

|                    |             |                    |                |
| :----------------- | :---------- | :----------------- | :------------- |
| **id**             | 1           | 1                  | 2              |
| **select\_type**   | SIMPLE      | SIMPLE             | `MATERIALIZED` |
| **table**          | s1          | &lt;subquery2&gt;  | s2             |
| **partitions**     | NULL        | NULL               | NULL           |
| **type**           | ALL         | eq\_ref            | index          |
| **possible\_keys** | idx\_key1   | &lt;auto\_key&gt;  | idx\_key1      |
| **key**            | NULL        | &lt;auto\_key&gt;  | idx\_key1      |
| **key\_len**       | NULL        | 303                | 303            |
| **ref**            | NULL        | mysql\_run.s1.key1 | NULL           |
| **rows**           | 9937        | 1                  | 9937           |
| **filtered**       | 100         | 100                | 100            |
| **Extra**          | Using where | NULL               | Using index    |



## possible_keys和key

> possiable_keys列是对表进行查询时，可能使用到的索引
>
> key列是对表进行查询时，实际使用的索引
>
> 优化器进行成本计算后，选择使用`uk_key2`索引对表进行查询
>
> possiable_keys并不是越多越好，在执行查询前，优化器需要对索引进行成本，索引越多，计算成本时的花费时间也就越长

```sql
explain
select *
from single_table
where key1 > 'a'
  and key2 = 13467;
```

|                    |                    |
| :----------------- | :----------------- |
| **id**             | 1                  |
| **select\_type**   | SIMPLE             |
| **table**          | single\_table      |
| **partitions**     | NULL               |
| **type**           | const              |
| **possible\_keys** | `uk_key2,idx_key1` |
| **key**            | `uk_key2`          |
| **key\_len**       | 5                  |
| **ref**            | const              |
| **rows**           | 1                  |
| **filtered**       | 100                |
| **Extra**          | NULL               |



## key_len

> 当使用索引来执行查询时，要弄清楚查询条件形成的扫描区间，以及扫描区间的边界条件
>
> 我们可以根据执行计划的key_len了解边界条件有关`索引列的相关信息`

```sql
explain
select *
from single_table
where key1 > 'a'
  and key1 < 'd';
```

|                    |                       |
| :----------------- | :-------------------- |
| **id**             | 1                     |
| **select\_type**   | SIMPLE                |
| **table**          | single\_table         |
| **partitions**     | NULL                  |
| **type**           | range                 |
| **possible\_keys** | idx\_key1             |
| **key**            | idx\_key1             |
| **key\_len**       | `303`                 |
| **ref**            | NULL                  |
| **rows**           | 899                   |
| **filtered**       | 100                   |
| **Extra**          | Using index condition |

> key_len的值为303
>
> 
>
> key_len的值由3部分组成：
>
> 1. 该列`数据实际最多占用的字节数`
> 2. 如果该列可以存储`Null`，需要有1字节来维护
> 3. 如果该列数据是变长的，需要2字节来维护
>
> 
>
> 在这个表中，key1列数据类型是`varchar(100)`，使用的是utf8字符集，且它允许为空，它的key_len组成是
>
> 1. 3×100=300（utf8字符最多占用3字节，key1的类型长度是100）
> 2. 1字节维护存储Null值
> 3. 2字节维护变长存储
>
> 
>
> 在InnoDB的记录行格式中，存储变长字段的部分可能是1字节或2字节，但执行计划是server层的功能,server层与存储引擎表示记录的方式是不一样的





> 对于Id列，它是int型，占用4字节，且不是变长、不允许存储Null

```sql
explain
select *
from single_table
where id > 10
  and id < 100;
```

|                    |               |
| :----------------- | :------------ |
| **id**             | 1             |
| **select\_type**   | SIMPLE        |
| **table**          | single\_table |
| **partitions**     | NULL          |
| **type**           | range         |
| **possible\_keys** | PRIMARY       |
| **key**            | PRIMARY       |
| **key\_len**       | `4`           |
| **ref**            | NULL          |
| **rows**           | 89            |
| **filtered**       | 100           |
| **Extra**          | Using where   |



### 联合索引与key_len

> key_len主要是为了让我们在使用联合索引进行查询时，了解到优化器使用索引`涉及到哪些列`

```sql
explain
select *
from single_table
where key_part1 = 'a'
  and key_part2 > 'c';
```

|                    |                       |
| :----------------- | :-------------------- |
| **id**             | 1                     |
| **select\_type**   | SIMPLE                |
| **table**          | single\_table         |
| **partitions**     | NULL                  |
| **type**           | range                 |
| **possible\_keys** | idx\_part             |
| **key**            | idx\_part             |
| **key\_len**       | `606`                 |
| **ref**            | NULL                  |
| **rows**           | 106                   |
| **filtered**       | 100                   |
| **Extra**          | Using index condition |

> key_len为606，MySQL在使用上面的查询语句时，会使用到key_part1、key_part2这两个列来形成扫描区间



## ref

> ref列的内容表示与索引列进行等值匹配的内容（常数、表的一个列）

```sql
explain
select *
from single_table
where key1 = 'a';
```

|                    |               |
| :----------------- | :------------ |
| **id**             | 1             |
| **select\_type**   | SIMPLE        |
| **table**          | single\_table |
| **partitions**     | NULL          |
| **type**           | ref           |
| **possible\_keys** | idx\_key1     |
| **key**            | idx\_key1     |
| **key\_len**       | 303           |
| **ref**            | `const`       |
| **rows**           | 138           |
| **filtered**       | 100           |
| **Extra**          | NULL          |



```sql
explain
select *
from single_table s1
         join single_table s2
where s1.key1 = s2.common_field;
```

|                    |             |                             |
| :----------------- | :---------- | :-------------------------- |
| **id**             | 1           | 1                           |
| **select\_type**   | SIMPLE      | SIMPLE                      |
| **table**          | s2          | s1                          |
| **partitions**     | NULL        | NULL                        |
| **type**           | ALL         | ref                         |
| **possible\_keys** | NULL        | idx\_key1                   |
| **key**            | NULL        | idx\_key1                   |
| **key\_len**       | NULL        | 303                         |
| **ref**            | NULL        | `mysql_run.s2.common_field` |
| **rows**           | 9937        | 2                           |
| **filtered**       | 100         | 100                         |
| **Extra**          | Using where | NULL                        |

> 执行计划中的第2条记录中的ref是 `mysql_run.s2.common_field`，表示在对s1表进行访问时，与s1表的key1列的等值匹配的是s2.common_field的列



```sql
explain
select *
from single_table s1
         join single_table s2 on s2.key1 = lower(s1.key3);
```

|                    |        |                       |
| :----------------- | :----- | :-------------------- |
| **id**             | 1      | 1                     |
| **select\_type**   | SIMPLE | SIMPLE                |
| **table**          | s1     | s2                    |
| **partitions**     | NULL   | NULL                  |
| **type**           | ALL    | ref                   |
| **possible\_keys** | NULL   | idx\_key1             |
| **key**            | NULL   | idx\_key1             |
| **key\_len**       | NULL   | 303                   |
| **ref**            | NULL   | `func`                |
| **rows**           | 9937   | 2                     |
| **filtered**       | 100    | 100                   |
| **Extra**          | NULL   | Using index condition |

> 对被驱动表s2进行查询时，与s2表的key1进行等值匹配的对象是一个函数



## rows

> 如果优化器决定采用`全表扫描`的方式进行查询，那么rows的值就是对表进行全表扫描时预估的行数。
>
> 如果优化器决定采用`索引`来执行查询，那么rows的值就代表预计扫描索引的记录行数。



## filterd

> 在分析连接扫描的成本时，会进行`条件过滤`。
>
> 如果MySQL采用全表扫描的方式来进行单表查询，那么在计算驱动表扇出时，需要估计满足`全部查询条件`的记录的行数。
>
> 如果MySQL采用索引来执行单表扫描，那么计算驱动表扇出时，需要估计`满足索引扫描区间外，还满足其他查询条件`的记录行数

```sql
explain
select *
from single_table s1
         join single_table s2 on s1.key1 = s2.key3 and s1.common_field < 'b';
```

|                    |             |                    |
| :----------------- | :---------- | :----------------- |
| **id**             | 1           | 1                  |
| **select\_type**   | SIMPLE      | SIMPLE             |
| **table**          | s1          | s2                 |
| **partitions**     | NULL        | NULL               |
| **type**           | ALL         | ref                |
| **possible\_keys** | idx\_key1   | idx\_key3          |
| **key**            | NULL        | idx\_key3          |
| **key\_len**       | NULL        | 303                |
| **ref**            | NULL        | mysql\_run.s1.key1 |
| **rows**           | 9937        | 2                  |
| **filtered**       | `33.33`     | `100`              |
| **Extra**          | Using where | NULL               |

> 在单表查询里，filtered没有什么意义，在上面的执行计划中我们可以了解到，优化器在执行上面的查询语句时，选择将s1表作为驱动表，s2表作为被驱动表，在执行计划的第一条记录中，filtered的值是1/3，9937÷3≈3312，也就是说，还需要对被驱动表执行大约3312次



## Extra

> 执行计划的额外信息



### No tables used

> 没有from子句

```sql
explain select 1；
```



### Impossible WHERE

> where子句永远为false

```sql
explain
select *
from single_table
where false = true;
```



### No matching min/max row

> 查询语句有min、max，但记录不满足where的条件

```sql
explain
select min(key2)
from single_table
where key2 = 10;
```



### Using index

> 使用覆盖索引进行查询



### Using index condition（减少回表）

> `索引条件下推`，搜索条件出现了索引列，但`没有利用它`形成扫描区间（不能使用这个条件来减少记录扫描的数量）

```sql
explain
select *
from single_table
where key_part1 < 'd' # 条件1
  and key_part2 < 'd'; # 条件2
```

|                    |                         |
| :----------------- | :---------------------- |
| **id**             | 1                       |
| **select\_type**   | SIMPLE                  |
| **table**          | single\_table           |
| **partitions**     | NULL                    |
| **type**           | range                   |
| **possible\_keys** | idx\_part               |
| **key**            | idx\_part               |
| **key\_len**       | `303`                   |
| **ref**            | NULL                    |
| **rows**           | 1080                    |
| **filtered**       | 33.33                   |
| **Extra**          | `Using index condition` |

> key_len为303，上面的查询，只使用了key_part1列来形成扫描区间。
>
> 对于`条件2 key_part2 < 'd'`，MySQL使用了`索引条件下推`，存储引擎判断
> `key_part1 < 'd' and key_part2 < 'd'`是否成立
> 
>如果没有索引下推，在扫描到符合`条件1 key_part1 < 'd'`的记录后，MySQL会根据记录的主键进行回表操作，将完整的用户记录返回给客户端，再server层判断该记录是否符合`条件2`。
> 
>如果查询的条件中`只有条件1`，在执行计划额外信息中也会包含`Using index condition`，对于非精确匹配的扫描区间来说，形成扫描区间的边界条件也会被当作ICP条件下推到存储引擎判断
> 
>`InnoDB规定索引条件下推只适用于非聚簇索引`



### Using where

> 某个查询条件需要在server层进行判断



### Using join buffer (Block Nested Loop)

> 在连接查询中，对被驱动访问`不能有效利用索引`加快访问速度时，MySQL会为其分配一块`连接缓冲区`，它是`基于块的嵌套循环算法`来执行连接查询的。

```sql
explain
select *
from single_table s1
         join single_table s2 on s1.common_field = s2.common_field;
```

|                    |        |                                                      |
| :----------------- | :----- | :--------------------------------------------------- |
| **id**             | 1      | 1                                                    |
| **select\_type**   | SIMPLE | SIMPLE                                               |
| **table**          | s1     | s2                                                   |
| **partitions**     | NULL   | NULL                                                 |
| **type**           | ALL    | ALL                                                  |
| **possible\_keys** | NULL   | NULL                                                 |
| **key**            | NULL   | NULL                                                 |
| **key\_len**       | NULL   | NULL                                                 |
| **ref**            | NULL   | NULL                                                 |
| **rows**           | 9937   | 9937                                                 |
| **filtered**       | 100    | 10                                                   |
| **Extra**          | NULL   | Using where; `Using join buffer (Block Nested Loop)` |



### Using [intersect, union, sort_union]

> 使用了索引合并



### Zero limit

> 出现了 `limt 0`



### Using filesort

> 对结果集的记录进行排序时，但没有使用到索引
>
> MySQL把在内存或磁盘中进行的排序称为`文件排序`

```sql
explain
select *
from single_table
order by common_field limit 10;
```

|                    |                  |
| :----------------- | :--------------- |
| **id**             | 1                |
| **select\_type**   | SIMPLE           |
| **table**          | single\_table    |
| **partitions**     | NULL             |
| **type**           | ALL              |
| **possible\_keys** | NULL             |
| **key**            | NULL             |
| **key\_len**       | NULL             |
| **ref**            | NULL             |
| **rows**           | 9937             |
| **filtered**       | 100              |
| **Extra**          | `Using filesort` |



### Using temporary

> MySQL需要借助临时表来完成一些功能（去重、排序等），如果不能有效利用索引完成这些功能，MySQL很有可能创建临时表来执行一些查询（包含distinct、order by、union等），应尽量避免查询中出现创建临时表的情况

```sql
explain
select  common_field
from single_table group by common_field;
```

|                    |                                   |
| :----------------- | :-------------------------------- |
| **id**             | 1                                 |
| **select\_type**   | SIMPLE                            |
| **table**          | single\_table                     |
| **partitions**     | NULL                              |
| **type**           | ALL                               |
| **possible\_keys** | NULL                              |
| **key**            | NULL                              |
| **key\_len**       | NULL                              |
| **ref**            | NULL                              |
| **rows**           | 9937                              |
| **filtered**       | 100                               |
| **Extra**          | `Using temporary; Using filesort` |

> 使用`group by`时，将`默认使用order by`



### Start temporary与End temporary

> 优化器优先将`in子查询`转换为`半连接`，如果采用的半连接策略是`重复消除（Duplicate Weedout）`，也就是通过建立临时表为外层查询去重时，驱动表的执行计划记录额外信息就包含`Start temporary`，被驱动表包含  `End temporary`

```sql
explain
select *
from single_table s1
where key1 in (select common_field from single_table s2 where s1.key3 = 'a');
```

|                    |                                |                              |
| :----------------- | :----------------------------- | :--------------------------- |
| **id**             | 1                              | 1                            |
| **select\_type**   | SIMPLE                         | SIMPLE                       |
| **table**          | s2                             | s1                           |
| **partitions**     | NULL                           | NULL                         |
| **type**           | ALL                            | ref                          |
| **possible\_keys** | NULL                           | idx\_key1,idx\_key3          |
| **key**            | NULL                           | idx\_key1                    |
| **key\_len**       | NULL                           | 303                          |
| **ref**            | NULL                           | mysql\_run.s2.common\_field  |
| **rows**           | 9937                           | 2                            |
| **filtered**       | 100                            | 2.1                          |
| **Extra**          | Using where; `Start temporary` | Using where; `End temporary` |



### LooseScan

> 半连接的执行策略是LooseScan



### FirstMatch(表名)

> 半连接的执行策略是FirstMatch

```sql
explain
select *
from single_table s1
where key1 in (select common_field from single_table s2 where s1.key3 = s2.key3);
```

|                    |                     |                               |
| :----------------- | :------------------ | :---------------------------- |
| **id**             | 1                   | 1                             |
| **select\_type**   | SIMPLE              | SIMPLE                        |
| **table**          | s1                  | s2                            |
| **partitions**     | NULL                | NULL                          |
| **type**           | ALL                 | ref                           |
| **possible\_keys** | idx\_key1,idx\_key3 | idx\_key3                     |
| **key**            | NULL                | idx\_key3                     |
| **key\_len**       | NULL                | 303                           |
| **ref**            | NULL                | mysql\_run.s1.key3            |
| **rows**           | 9937                | 2                             |
| **filtered**       | 100                 | 10                            |
| **Extra**          | Using where         | Using where; `FirstMatch(s1)` |



## optimizer trace

> MySQL提供了 optimizer trace 功能，它可以让用户查看优化器生成执行计划的整个过程



```sql
show variables like 'optimizer_trace';
set optimizer_trace = 'enabled=on';
```

> 开启optimizer trace功能后，我们进行查询（使用explain语句也可以）后，可以在相关表中查看执行计划生成的过程。
>
> 这个表的结构为：
>
> - query：输入的语句
> - trace：优化过程的JSON文本
> - missing bytes beyond max mem size：被忽略的文本字节数

```sql
select *
from information_schema.OPTIMIZER_TRACE;
```

> 优化的过程大致分为三个阶段：
>
> 1. prepare
> 2. optimize
> 3. execute
>
> 基于成本的优化主要集中在optimize阶段
>
> 对于单表查询，我们关注的是optimize阶段中的`rows_estimation`阶段（预估不同单表访问的成本）
>
> 对于多表连接查询，我们更关注`considered_execution_plans`阶段（分析可能的执行计划）
