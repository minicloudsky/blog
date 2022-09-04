---
title: hbase 基础
date: 2020-06-01 15:59:14
tags:
---


## hbase 是一个基于列存储的数据库，底层存储可以用hdfs，也可以用本地文件系统，这里记录下常用操作。

### 数据模型
在 HBase 中，数据存储在由行列构成的二维表中，这是与一个关系数据库(RDBMS)重叠的术语，但这不是一个有帮助的比喻，相反，将hbase 表视为多表映射会很有帮助。
- 表
HBase 表由多行组成
- 行
HBase 中的一行由一个行键和一个或多个具有与之关联的值的列组成，存储时候按照行键排序，因此，行键设计非常重要，其目标是以相关行彼此靠近的方式存储数据。。常见的行键模式是网站域名。如果您的行键是域名，则应该反向存储它们（org.apache.www，org.apache.mail，org.apache.jira）。这样，所有 Apache 域名都在表中彼此靠近，而不是基于子域名的第一个字母展开。
- 列
HBase 中的列由列族和列限定符组成，它们由 : (冒号) 字符分隔
- 列族
出于性能原因，列族通常在物理上拥有一簇列及其值。每个列族都有一组存储属性，例如是否应该将其值缓存在内存中，如何压缩其数据或对其行间进行编码等，表中的每一行都有相同的列族，到给定的行肯呢个不会再给定的列族中存储任何内容。
- 列限定符
列限定符被添加到列族中，以提供给定数据段的索引，给定`列族` content，列限定符可能是 `content:html`,另一个可能是 `content:pdf`，列族在创建表时候是固定的，但是列限定符是可变的，并且行之间可能有很大差异。
- 单元格
单元格是行，列族和列限定符的组合，它包含一个值和一个时间戳，时间戳表示值的版本。
- 时间戳
时间戳与每个值一起写入，它是该值给定版本的标识符，默认情况下，时间戳表示写入数据时 RegionServer 上的时间，但是当年将数据放入单元格时候，可以指定不同的时间戳值。

### 概念视角 - HBase 模式设计
参考 [HBase 模式设计介绍](https://zhuanlan.zhihu.com/p/36235199)

### 物理视角
在概念级别，表可以看作一组稀疏的行，但在物理意义上它们是按照列族存储的，可以随时将新的列限定符(column_family:column_qualifier) 添加到现有列族中
ColumnFamily website
Row Key | Time Stamp | 列族 anchor
"com.cnn.www" |	t9 |	anchor:cnnsi.com = "CNN"
"com.cnn.www" |	t8 |	anchor:my.look.ca = "CNN.com"

ColumnFamily contents
Row Key | Time Stamp | ColumnFamily contents:
"com.cnn.www"|	t6  |	contents:html = "…​"
"com.cnn.www"|	t5	|contents:html = "…​"
"com.cnn.www"|	t3  |	contents:html = "…​"
概念视角中显示的空单元格不占据物理存储空间，因此，在时间戳 t8 处对 `contents:html` 列的值的请求将不返回任何值。类似地，在时间戳t9处对anchor:my.look.ca值的请求将不返回任何值。但是，如果未提供时间戳，则将返回特定列的最新值。给定多个版本，最新版本也是第一个版本，因为时间戳按降序存储。因此，如果没有指定时间戳，则对行com.cnn.www中所有列的值的请求将是：来自时间戳t6的contents:html的值，来自时间戳t9的anchor:cnnsi.com的值，来自时间戳t8的anchor:my.look.ca。
### 命名空间
命名空间是与关系数据库系统中的数据库类似的表的逻辑分组，这种抽象为即将出现的多租户相关功能奠定了基础
- 配额管理 限制命名空间可以使用的资源数量(即区域、表)
- 命名空间安全管理 - 为用户提供另一级别的安全管理
- 区域服务器组 - 可以将命名空间/表固定到 RegionServers 的子集上，从而保证粗粒度的隔离级别。
### 命名空间管理
一个空间可以被创建，被删除或被更改，通过制定表单的完全限定表名，在表创建期间确定命名空间成员资格
```xml
<table namespace>:<table qualifier>
```
```shell
# Create a namespace
create_namespace 'project'
# create my_table in project namespace
create 'project:my_table' ,'fam'
# drop namespace
drop_namespace 'project'
# alter namespace
alter_namespace 'project',{METHOD => 'set','PROPERTY_NAME' => 'PROPERTY_NAME'}
```
### 预定义的名称空间
有两个预定义的特殊命名空间
- hbase - 系统命名空间，用于包含 HBase 内部表
- default - 没有明确制定名称空间的表将自动落入此名称空间
```shell
#namespace=foo and table qualifier=bar
create 'foo:bar', 'fam'

#namespace=default and table qualifier=bar
create 'bar', 'fam' 
```
### 表
表在模式定义时预先声明
### 行
行键是无解释的字节，行按字典顺序排序，最低顺序首先出现在表中，空字节数组用于表示表的名称空间的开始和结束
### 列族
HBase 中的列分组为列族，列族的所有列成员都具有相同的前缀。例如，列 `courses:history` 和 `courses:math` 都是 `courses` 列族的成员，冒号字符(:) 从列族限定符中分隔列族。列族前缀必须由可打印字符组成，限定符的右部(列族限定符)可以由任意字节组成。列族必须在模式定义时预先定义，且可以在表启动和运行时动态变化。
从物理上讲，所有列族成员都存储在文件系统中。由于调优和存储规范是在列族级别完成的，因此建议所有列族成员具有相同的一般访问模式和大小特征。
### 单元格
{row,column,version} 它是一个元祖并确切地指定了 HBase 的cell，单元格内容是未解释的字节。
### 数据模型操作
四个主要的数据模型操作是 Get,Put,Scan和 Delete，通过表实例应用操作。
### 版本
{row，column，version} 元祖确切地指定了 HBase 中的 cell，可以有一个无限数量的单元格，其中行和列相同但单元格地址仅在其版本纬度上有所不同。

### 排序顺序
HBase 中所有数据模型的操作都将按照排序顺序返回数据，首先是行，然后是列族，后面是列限定符，最后是时间戳(反向排序，因此首先返回最新的时间戳。)
### 列的元数据
列族的内部 KeyValue 实例之外没有列的元数据存储，因此，尽管HBase 不紧可以支持每行的大量列，而且还可以支持行之间的异构列，因此有必要追踪列名。
获取列族存在的完整列集合的唯一方法是处理所有行。
虽然行和列键表示为字节，但版本则使用长整数（long integer）类型指定。通常这个长时间类型包含时间实例，例如java.util.Date.getTime()或System.currentTimeMillis()返回的时间实例，即：当前时间与 UTC 时间 1970 年 1 月 1 日午夜之间的差异（以毫秒为单位）。

HBase 版本维度按递减顺序存储，以便在从存储文件中读取时，首先找到最近的值。
在 HBase 中，cell版本的语义存在很多混淆。特别是：
如果对单元格的多次写入具有相同的版本，则只能读取最后写入的内容。
可以按非增加版本顺序写入单元格。
### HBase 表 Schema 的经验法则
因为存在许多种数据集，不同的访问模式和服务层级的要求。以下经验法则只是概述。
- 目标是把 region 的大小限制在 10-50 GB之间
- 目标是限制 cell 的大小在 10MB 之内，如果使用的是 mob 类型，限制在 50 MB 之内。否则，考虑把 cell 的数据存储在 HDFS中，并在 HBase 中存储指向该数据的指针。
- 典型 schema 每张表包含1-3个列族，HBase 表设计不应该和 RDBMS 表设计类似
- 对于拥有1或者2个列族的表来说，50-100个 region 是比较合适的。region 是列族的连续段。
- 保持列族名称尽可能短。每个值都会存储列族的名称(忽略前缀编码)。它们不应该像典型 RDBMS 那样，是自文档化，描述性的名称。
- 如果你正在存储基于时间的机器数据或者日志信息，并且 row key 是基于设备id 或者服务id + 时间，最终会出现这样一种情况，即更旧的数据 region 永远不会有额外写入。在这种情况下，最终会存在少量的活动 region 和大量不会再有新写入的 region。对于这种情况，可以接受更多的 region 数量，因为资源的消耗只取决于活动 region。
- 如果只有一个列族会频繁写那么只会让这个列族占用内存，当分配资源时候注意写入模式。
### 用 hbase shell连接正在运行的Hbase实例
```shell
[tools@uatgw01 ~]$ hbase shell

HBase Shell
Use "help" to get list of supported commands.
Use "exit" to quit this interactive shell.
For Reference, please visit: http://hbase.apache.org/2.0/book.html#shell
Version 2.1.0-cdh6.3.1, rUnknown, Thu Sep 26 02:56:36 PDT 2019
Took 0.0018 seconds                                                                                
hbase(main):001:0> 
hbase(main):002:0* 
```
### 创建表
使用 create 创建一个表，必须执行一个表名和列族名
```shell
hbase(main):002:0* create 'hbase_test','cf'
Created table hbase_test
Took 2.5584 seconds                                                                                
=> Hbase::Table - hbase_test
hbase(main):003:0>
```
### 表信息 - 使用 `list` 查看存在表
```shell
hbase(main):003:0> list 'hbase_test'
TABLE                                                                                              
hbase_test                                                                                         
1 row(s)
Took 0.0318 seconds                                                                                
=> ["hbase_test"]
```
<!-- more -->
### describe 查看表细节及配置
```shell
hbase(main):004:0> describe 'hbase_test'
Table hbase_test is ENABLED                                                                        
hbase_test                                                                                         
COLUMN FAMILIES DESCRIPTION                                                                        
{NAME => 'cf', VERSIONS => '1', EVICT_BLOCKS_ON_CLOSE => 'false', NEW_VERSION_BEHAVIOR => 'false', 
KEEP_DELETED_CELLS => 'FALSE', CACHE_DATA_ON_WRITE => 'false', DATA_BLOCK_ENCODING => 'NONE', TTL =
> 'FOREVER', MIN_VERSIONS => '0', REPLICATION_SCOPE => '0', BLOOMFILTER => 'ROW', CACHE_INDEX_ON_WR
ITE => 'false', IN_MEMORY => 'false', CACHE_BLOOMS_ON_WRITE => 'false', PREFETCH_BLOCKS_ON_OPEN => 
'false', COMPRESSION => 'NONE', BLOCKCACHE => 'true', BLOCKSIZE => '65536'}                        
1 row(s)
Took 0.2977 seconds
```
### 插入数据，使用 `put` 插入数据
```shell
hbase(main):005:0> put 'hbase_test','row1','cf:aaa','value1'
Took 0.2313 seconds                                                                                
hbase(main):006:0> put 'hbase_test','row2','cf:bbb','value2'
Took 0.0093 seconds                                                                                
hbase(main):007:0> put 'hbase_test','row3','cf:ccc','value3'
Took 0.0066 seconds  
```
这里，我们往 `test` 表插入了三条数据，首先插入了一条 rowky 为 `row1`，列为 `cf:a`、值为`value``的数据。HBase 中的列是包含列族前缀的，在这个例子里，冒号前的未列`cf`,冒号后的未列限定符 `a`
### 扫描全部数据
从 HBase 获取数据的途径之一就是 `scan`，使用 scan 命令扫描表数据，你可以对扫描做限制，这里获取全部的数据。
```shell
hbase(main):008:0> scan 'hbase_test'
ROW                       COLUMN+CELL                                                              
 row1                     column=cf:aaa, timestamp=1612166766975, value=value1                     
 row2                     column=cf:bbb, timestamp=1612166780484, value=value2                     
 row3                     column=cf:ccc, timestamp=1612166789458, value=value3                     
3 row(s)
Took 0.0790 seconds 
```
### 获取一条数据
使用 `get` 一次获取一条数据
```shell
hbase(main):009:0> get 'hbase_test','row1'
COLUMN                    CELL                                                                     
 cf:aaa                   timestamp=1612166766975, value=value1                                    
1 row(s)
Took 0.0278 seconds 
```
### 禁用表
删除一个表或者改变表的设置，以及一些其他的场景，需要首先用 `disable` 禁用表，可以使用`enable`启用表
```shell
hbase(main):010:0> disable 'hbase_test'
Took 1.4835 seconds                                                                                
hbase(main):011:0> enable 'hbase_test'
Took 0.7826 seconds 
```
### 删除表 - 用 `drop` 删除一个表
```shell
hbase(main):013:0> drop 'hbase_test'
Took 0.6806 seconds
```
### 获取表行数 
Count命令返回表中的行数，配置正确的 CACHE 时速度非常快
```shell
hbase(main):001:0> count 'test',CACHE => 1000
4 row(s)
Took 0.0524 seconds                                                                                
=> 4

```
上述计数一次取 1000 行。如果行很大，请将 CACHE 设置得较低。默认是每次读取一行。

