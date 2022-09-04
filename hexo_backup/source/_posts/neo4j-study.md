---
title: neo4j 图数据库学习
date: 2020-07-01 12:59:55
categories: 图数据库 大数据
tags:
- 图数据库
- 大数据

---

##### neo4j 图数据库学习

##### 节点

##### 属性

##### 关系

##### 标签

1. create 创建节点

   ```sql
   create(node_name:label_name {attribute_key:value})
   create(张小龙:微信 {age:45,company:"腾讯",position:"微信创始人"})
   ```

2. match 从数据库获取有关节点和属性的数据

   ```sql
   从数据库获取有关节点，关系和属性的数据
   match(node_name:label_name)
   ```

3. 3 return 检索节点的某些属性

   ```sql
   检索节点的所有属性
   检索节点和关联关系的某些属性
   检索节点和关联关系的所有属性
   return node.attribute
   ```

4.   MATCH + RETURN命令

   ```sql
   match 和return语句不能单独使用，需要结合使用 
   检索节点的某些属性
   检索节点的所有属性
   检索节点和关联关系的某些属性
   检索节点和关联关系的所有属性
   MATCH (cc:CreditCard)
   RETURN cc.id,cc.number,cc.cvv,cc.expiredate
   MATCH (cc:CreditCard)
   RETURN cccreate 创建标签
   ```

   ```sql
   CREATE (张三:程序员)-[rel:LOVE]->(李四:砖家)
   6 where子句
   简单 where 子句
   where <condition>
   复杂WHERE子句语法
   WHERE <condition> <boolean-operator> <condition>
   condition 语法
   <property-name> <comparison-operator> <value>
   ```

>```
>布尔运算符
>AND OR NOT XOR
>比较运算符
>= 
><>
><
>
>><=
>>=
>>where 查询
>>MATCH (公司:财务部) 
>>WHERE 公司.name = '田七' OR 公司.age < 30
>>RETURN 公司
>```
>
>
>

##### 5. 如何使用WHERE子句创建两个现有节点之间的关系。

```sql
MATCH (company:huawei),(bank:ccb) 
WHERE company.age = 45 AND bank.name="建设银行" 
CREATE (bank)-[r:mortage{date:"24/06/2020",price:550000}]->(company) 
RETURN r
```

##### 删除节点

```sql
match(company:tencent ) delete company
DELETE节点和关系子句语法
DELETE <node1-name>,<node2-name>,<relationship-name>
```

##### Neo4j CQL REMOVE命令用于

```bash
删除节点或关系的标签
删除节点或关系的属性
```

##### Neo4j CQL DELETE和REMOVE命令之间的主要区别 - 

`DELETE操作用于删除节点和关联关系。`
`REMOVE操作用于删除标签和属性。`

##### Neo4j CQL DELETE和REMOVE命令之间的相似性 - 

`这两个命令不应单独使用。`
`两个命令都应该与MATCH命令一起使用。`