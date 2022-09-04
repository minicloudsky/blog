---
title: hive 数据倾斜
date: 2020-10-22 18:25:10
tags:
---


## 什么是数据倾斜？
数据分布不均匀，造成数据大量的集中到一点，造成数据热点。
### Hadoop 特性
- 不怕数据量大，怕数据倾斜
- Job 数比较多的作业运行效率相对比较低，如很多子查询
- sum,count,max,min 等聚集函数，通常不会有数据倾斜问题
|  关键字   | 情形  | 后果 |
|  ----  | ----  |  ---  |
| join  | 其中一个表较小，但是key集中 | 分发到某一个或者几个Reduce上的数据远高于平均值  |
|    |   大表与大表，但是分桶的判断字段0值或者空值太多|  这些空值 都由一个reduce处理，非常慢|
| group by order by | 维度过小，某值的数量过多| 处理某值的 reduce 非常耗时|
| count ,distinct | 某特殊值过多 | 处理此特殊值的reduce 耗时|

<!-- more -->
1. group by 不和聚集函数搭配使用的时候
2. count(distinct) 在数据量大的情况下，容易数据倾斜，因为count(distinct)是按 group by 字段分组，按 distinct 字段排序。
3. 小表关联超大表 join
### 产生数据倾斜原因
- key 分布不均匀
- 业务数据本身的特性
- 建表考虑不周全
- 某些 HQL 语句本身就存在数据倾斜

### 业务场景
1. 空值产生的数据倾斜
在日志中，常常会有信息丢失的问题，比如日志中的 user_id，如果取其中的 user_id 和用户表的 user_id 关联，就会碰到数据倾斜问题。
解决方案1： user_id 为空的不参与关联
```sql
select * from log a join user b on a.user_id is not null and a.user_id = b.user_id union all 
select * from log c where c.user_id is null;
```
解决方案2： 赋予空值新的 key 值
```sql
select * from log a left outer join user b on
case when a.user_id is null then concat('hive_null_value',rand()) else a.user_id end =b.user_id
```
方法 2 比方法 1 效率更好，不但 IO 少了，而且作业数也少了，方案 1 中，log 表 读了两次，jobs 肯定是 2，而方案 2 是 1。这个优化适合无效 id（比如-99，’’，null）产 生的数据倾斜，把空值的 key 变成一个字符串加上一个随机数，就能把造成数据倾斜的 数据分到不同的 reduce 上解决数据倾斜的问题。

改变之处：使本身为 null 的所有记录不会拥挤在同一个 reduceTask 了，会由于有替代的 随机字符串值，而分散到了多个 reduceTask 中了，由于 null 值关联不上，处理后并不影响最终结果。
2. 不同数据类型关联产生数据倾斜
场景
用户表 user_id 字段类 int，log 表中 user_id 既有 string 也有 int 的类型，当按照两个表的 user_id 进行 join 操作的时候，默认的 hash 操作会按照 int 类型的 id进行分配，这样就会导致所有的 string类型的 id 就被分到同一个 reducer 中
解决方案
把数字类型 id 转换成 string 类型的 id
```sql
select * from user a left outer join log b onb.user_id = cast(a.user_id as string)
```
3. 大小表关联查询产生数据倾斜
注意: 使用 map join 解决小表关联大表造成的数据倾斜。
map join 概念: 将其中做连接的小表(全量数据) 分发到所有MapTask 端进行 join，从而避免了 reduceTask，前提要求是内存足以装下该全量数据
![](hive.png)
以大表 a 和小表 b 为例，所有的 maptask 节点都装载小表 b 的所有数据，然后大表 a 的 一个数据块数据比如说是 a1 去跟 b 全量数据做链接，就省去了 reduce 做汇总的过程。 所以相对来说，在内存允许的条件下使用 map join 比直接使用 MapReduce 效率还高些， 当然这只限于做 join 查询的时候。

在 hive 中，直接提供了能够在 HQL 语句指定该次查询使用 map join，map join 的用法是 在查询/子查询的SELECT关键字后面添加/*+ MAPJOIN(tablelist) */提示优化器转化为map join（早期的 Hive 版本的优化器是不能自动优化 map join 的）。其中 tablelist 可以是一个 表，或以逗号连接的表的列表。tablelist 中的表将会读入内存，通常应该是将小表写在 这里。
MapJoin 具体用法
```sql
select /* +mapjoin(a) */ a.id aid, name, age from a join b on a.id = b.id;
select /* +mapjoin(movies) */ a.title, b.rating from movies a join ratings b on a.movieid =
b.movieid;
```
hive0.11 版本以后会自动开启 map join 优化，由两个参数控制：
```sql
set hive.auto.convert.join=true; //设置 MapJoin 优化自动开启

set hive.mapjoin.smalltable.filesize=25000000 //设置小表不超过多大时开启 mapjoin 优化
```
如果是大大表关联呢？那就大事化小，小事化了。把大表切分成小表，然后分别 map join
那么如果小表不大不小，那该如何处理呢？？？
使用 map join 解决小表(记录数少)关联大表的数据倾斜问题，这个方法使用的频率非常高，但如果小表很大，大到 map join 会出现 bug 或异常，这时就需要特别的处理
举一例：日志表和用户表做连接
```sql
select * from log a left outer join users b on a.user_id = b.user_id;
```
users 表有 1000w+的记录，把 users 分发到所有的 map 上也是个不小的开销，而且 map join 不支持这么大的小表。如果用普通的 join，又会碰到数据倾斜的问题。
```sql
select /*+mapjoin(x)*/* from log a
left outer join (
 select /*+mapjoin(c)*/ d.*
 from ( select distinct user_id from log ) c join users d on c.user_id = d.user_id
) x
on a.user_id = x.user_id;
```
假如，log 里 user_id 有上百万个，这就又回到原来 map join 问题,不过目前我们业务因为行业原因，用户量不会特别特别大，因此通过map join 是可以暂时解决的。