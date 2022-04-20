---
title: mysql sql调优
date: 2022-04-19 11:35:23
tags: 
- mysql
---

# explain语法



explain有两种扩展(<5.7), 一个是`explain extend`, 结果增加了filtered列；另一个是`explain partitions`，结果增加了partitions列；

![图1](http://wyett.github.io/assets/img/mysql_explain/image-20210701165349233.png)

在mysql 5.7版本中，又增加了`format=json`选项，将explain结果通过json格式显示；同时也支持支持对select/update/delete/replace等不同的DML操作进行分析。

![图2](http://wyett.github.io/assets/img/mysql_explain/image-20210701165447077.png)



# explain解析结果

- **id**：子句执行顺序，id大的先执行，id相同的从上往下按顺序执行

- **table**: 表名或者表的alias，`<derivedN>`，`<union m,n>`, `subqueryN`

- **partitions**：分区

- **posible_key**: 查询优化器评估了哪些索引；

- **key**：最终选择的索引



需要计算的信息

**key_len**：

在ref列信息不为const时，通过计算key_len，可以知道用了最左前缀的哪些列

- 计算条件用了索引的哪些前缀列。

- 字符集和字节：utf8(3**n)/utf8mb4(3~4*n)/gbk(2*n)

- null和非null：占1字节/0字节

- 常见的数据类型长度
  - date(3)/datetime(8)和timestamp(4)
  
  | type             | 字节                      |
  | ---------------- | ------------------------- |
  | datetime(非毫秒) | <5.6(8字节)；>=5.6(5字节) |
  | datetime(毫秒)   | 毫秒1字节，微秒2字节      |
  
  - int(4byte)/bigint(8byte)/tinyint(1byte)/smallint(2byte)
  
  - varchar/char
  
    | type       | length |
    | ---------- | ------ |
    | varchar(N) | N+2    |
    | char(N)    | N      |
  



**rows**

- 按key执行这条SQL，需要scan多少条数据
- 这个值不准确，是mysql根据meta信息估计的



**filtered**

- 返回结果的行占需要读到的行(rows列的值)的百分比
- 不准确



*优化建议*

**select_type**

- simple。区别于union和driver
- primary/subquery/drivered

```sql
set session optimizer_switch='derived_merge=off';
explain select (select 1 from employees) from (select name from employees where id>10 limit 100) t;
```

![图3](http://wyett.github.io/assets/img/mysql_explain/image-20210701170949674.png)



- union：union之后的select子句
- materialized：物化子查询，第一次select把子查询结果生成临时表，以后有相同的则访问临时表；
- 还有一些其他的，最常见的是上面这几类

![图4](http://wyett.github.io/assets/img/mysql_explain/image-20210616201847992.png)

**type**

- 关联类型/访问类型
- 从优到差：system > const > eq_ref > ref  >range > index > ALL

- 有几个特殊的
  - ref_or_null<==>ref
  - unique_subquery/index_subquery<==>eq_ref
- 一般需要尽量控制在range，最好是ref

| type值          |            | 简介                                                         |
| --------------- | ---------- | ------------------------------------------------------------ |
| system          | 常量       | 表内只有一条数据，且与条件匹配                               |
| const           | 常量       | 表内最多有一条数据匹配                                       |
| eq_ref          | 连接内表   | 基于primary key 或 unique key 索引，链接内表，对外表的一条数据，内表只有一条数据对应 |
| ref             | 连接内表   | 基于索引的等值关联，链接字段不为NULL，对外表的一条数据，内表可能会找到多个符合条件的记录 |
| ref_or_null     | 链接内表   | 类似ref，但链接字段可为NULL                                  |
| range           | 范围扫描   | 基于索引的范围扫描， in(), between ,> ,<, >= ，like等操作中  |
| index           | 索引扫描   | 扫描整个索引                                                 |
| all             | 全表扫     | 即全表扫描，不使用索引，顺序扫描表上的数据                   |
| unique_subquery | 唯一子查询 | 在子查询种，基于唯一索引进行扫描                             |
| index_subquery  | 子查询     | 在子查询种，基于非唯一索引进行扫描                           |
| index_merge     | 索引合并   | 有多个索引可用时，对结果进行合并，去重等等。应该避免         |
| fulltext        |            | FT，全文检索                                                 |

> type为index和覆盖索引是类似的场景，区别在于：
>
> - type为index，扫描了索引里的每条数据，没有范围判断，效率也不是很高；是覆盖索引中最慢的一类

![图5](http://wyett.github.io/assets/img/mysql_explain/image-20210705110135436.png)





**ref**

- 用了索引哪个前缀列？

> <a,b,c> <a,b> <a>

- const/tt.id/NULL



**Extra**

- Using index ：使用了覆盖索引，覆盖索引一般针对的是辅助索引，整个
  查询结果只通过辅助索引就能拿到结果，不需要通过辅助索引树找到主键，再通过主键去主键索引树里获取其它字段值

```sql
explain select * from employees where name='wyett2'\G
```

![图6](http://wyett.github.io/assets/img/mysql_explain/image-20210701172508950.png)



- Using index condition ：查询的列不完全被索引覆盖，where条件中有前缀列的范围扫描，或者like。

```sql
explain select * from employees where name='wyett2' and age >20\G;
```

![图7](http://wyett.github.io/assets/img/mysql_explain/image-20210701172838319.png)



- Using where：使用 where 语句来处理结果，并且查询的列未被索引覆盖。<font color="red">优化标记</font>

```sql
alter table employees drop index idx_name_age_position;
explain select * from employees where name='wyett2'\G;
```

![图8](http://wyett.github.io/assets/img/mysql_explain/image-20210701173138013.png)



- Using  temporary：需要创建一张临时表来处理查询。<font color="red">优化标记</font>



```sql
alter table employees add index idx_hire_time(hire_time);
alter table employees drop index idx_name_age_position;
explain select distinct(name) from employees where hire_time >= '2021-06-17 9:37:00'\G;
```

![图9](http://wyett.github.io/assets/img/mysql_explain/image-20210701174045514.png)

```sql
explain select distinct(hire_time) from employees where hire_time >= '2021-06-17 9:37:00'\G;
```

![图10](http://wyett.github.io/assets/img/mysql_explain/image-20210701174143153.png)



> 优化方式：临时表列加入索引



- Using filesort：将用外部排序而不是索引排序，数据较小时从内存排序，否则需要在磁盘完成排序。<font color="red">优化标记</font>

```sql
explain select * from employees where hire_time >= '2021-06-17 9:37:00' order by name\G;
```

![图11](http://wyett.github.io/assets/img/mysql_explain/image-20210701174514409.png)





>把排序列加入索引，并且放在索引的后面，一般是最后一列



- Using join buffer：使用join buffer进行关联

```sql
explain select * from t1 inner join t2 on t1.b= t2.b
```

![图12](http://wyett.github.io/assets/img/mysql_explain/image-20210621111433728.png)



> 小结
>
> - 指明优化方向的：extra，type
> - 在ref为NULL时，通过key_len，判断使用了哪些前缀索引列
