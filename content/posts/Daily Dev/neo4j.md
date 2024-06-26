---
title: "neo4j常用命令"
date: 2024-06-26
tags: ["neo4j"]
categories: ["Daily Dev"]
author: "Kurong"
showToc: true
TocOpen: true
draft: false
hidemeta: false
comments: false
disableHLJS: false # to disable highlightjs
disableShare: true
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
---

## neo4j启动与访问

启动neo4j

```bash
docker start test_neo4j
docker exec -it test_neo4j /bin/bash
```

访问browser

```
http://localhost:7474/browser/
```

访问database

```
neo4j://localhost:7687
auth: neo4j
pw: 5225400599
```



## CQL语法

### create 创建节点

```cypher
CREATE (
   <node-name>:<label-name>
   { 	
      <Property1-name>:<Property1-Value>
      ........
      <Propertyn-name>:<Propertyn-Value>
   }
)
```



### match 查询节点或属性

```cypher
# 查询Dept下的内容
MATCH (dept:Dept) return dept

# 查询Employee标签下 id=123，name="Lokesh"的节点
MATCH (p:Employee {id:123,name:"Lokesh"}) RETURN p

## 查询Employee标签下name="Lokesh"的节点，使用（where命令）
MATCH (p:Employee)
WHERE p.name = "Lokesh"
RETURN p

## 返回一个table
MATCH (dept: Dept)
RETURN dept.deptno,dept.dname,dept.location
```

match 要绑定return使用，而return不能单独使用。



### create 创建关系

```cypher
# 使用现有节点创建没有属性的关系. 关系标签为“r”,关系名称为“DO_SHOPPING_WITH”
MATCH (e:Customer),(cc:CreditCard) 
CREATE (e)-[r:DO_SHOPPING_WITH ]->(cc)

# 查询关系                                   
MATCH (e)-[r:DO_SHOPPING_WITH ]->(cc) 
RETURN r 

# 使用现有节点创建有属性的关系
MATCH (cust:Customer),(cc:CreditCard) 
CREATE (cust)-[r:DO_SHOPPING_WITH{shopdate:"12/12/2014",price:55000}]->(cc) 
RETURN r
                                   
# 使用新节点创建没有属性的关系
CREATE (fb1:FaceBookProfile1)-[like:LIKES]->(fb2:FaceBookProfile2) 
RETURN like
                                   
# 使用新节点创建有属性的关系    
CREATE (video1:YoutubeVideo1{title:"Action Movie1",updated_by:"Abc",uploaded_date:"10/10/2010"})
-[movie:ACTION_MOVIES{rating:1}]->
(video2:YoutubeVideo2{title:"Action Movie2",updated_by:"Xyz",uploaded_date:"12/12/2012"})                                    
```



### create 创建标签

Label是Neo4j数据库中的节点或关系的名称或标识符。

```cypher
# 单个标签
CREATE (google1:GooglePlusProfile)
CREATE (p1:Profile1)-[r1:LIKES]->(p2:Profile2)

# 多个标签        
CREATE (m:Movie:Cinema:Film:Picture)
```

注意不可以为关系创建多个标签。



### where 条件过滤

```cypher
MATCH (emp:Employee) 
WHERE emp.name = 'Abc'
RETURN emp

MATCH (emp:Employee) 
WHERE emp.name = 'Abc' OR emp.name = 'Xyz'
RETURN emp

MATCH (cust:Customer),(cc:CreditCard) 
WHERE cust.id = "1001" AND cc.id= "5001" 
CREATE (cust)-[r:DO_SHOPPING_WITH{shopdate:"12/12/2014",price:55000}]->(cc) 
RETURN r
```



### delete 删除节点和其对应关系

```cypher
MATCH (e: Employee) DELETE e

MATCH (cc: CreditCard)-[rel]-(c:Customer) 
DELETE cc,c,rel
```



### remove 移除标签和属性

```cypher
# 移除属性
MATCH (book { id:122 })
REMOVE book.price
RETURN book

# 移除标签
MATCH (m:Movie) 
REMOVE m:Picture
```



### set 添加或更新属性

```cypher
MATCH (book:Book)
SET book.title = 'superstar'
RETURN book
```



### order by 排序

```cypher
MATCH (emp:Employee)
RETURN emp.empid,emp.name,emp.salary,emp.deptno
ORDER BY emp.name

MATCH (emp:Employee)
RETURN emp.empid,emp.name,emp.salary,emp.deptno
ORDER BY emp.name DESC
```



### union 结果集合并

union 将两组结果中的公共行组合并返回到一组结果中。 它不从两个节点返回重复的行。

结果列类型和来自两组结果的名称必须匹配，这意味着列名称应该相同，列的数据类型应该相同。

```cypher
MATCH (cc:CreditCard) RETURN cc.id,cc.number
UNION
MATCH (dc:DebitCard) RETURN dc.id,dc.number

MATCH (cc:CreditCard)
RETURN cc.id as id,cc.number as number,cc.name as name,
   cc.valid_from as valid_from,cc.valid_to as valid_to
UNION
MATCH (dc:DebitCard)
RETURN dc.id as id,dc.number as number,dc.name as name,
   dc.valid_from as valid_from,dc.valid_to as valid_to
```

union all 结合并返回两个结果集的所有行成一个单一的结果集。它还返回由两个节点重复行。

```cypher
MATCH (cc:CreditCard)
RETURN cc.id as id,cc.number as number,cc.name as name,
   cc.valid_from as valid_from,cc.valid_to as valid_to
UNION ALL
MATCH (dc:DebitCard)
RETURN dc.id as id,dc.number as number,dc.name as name,
   dc.valid_from as valid_from,dc.valid_to as valid_to
```



### limit和skip

Neo4j CQL已提供“LIMIT”子句来过滤或限制查询返回的行数。 它修剪CQL查询结果集底部的结果。

```cypher
MATCH (emp:Employee) 
RETURN emp
LIMIT 2
```

Neo4j CQL已提供“SKIP”子句来过滤或限制查询返回的行数。 它修整了CQL查询结果集顶部的结果。

```cypher
MATCH (emp:Employee) 
RETURN emp
SKIP 2
```



### merge 合并

MERGE命令是CREATE命令和MATCH命令的组合。

Neo4j CQL MERGE命令在图中搜索给定模式，如果存在，则返回结果。如果它不存在于图中，则它创建新的节点/关系并返回结果。

```cypher
MERGE (gp2:GoogleProfile2{ Id: 201402,Name:"Nokia"})
```



### NULL

Neo4j CQL将空值视为对节点或关系的属性的缺失值或未定义值。

当我们创建一个具有现有节点标签名称但未指定其属性值的节点时，它将创建一个具有NULL属性值的新节点。

```cypher
MATCH (e:Employee) 
WHERE e.id IS NOT NULL
RETURN e.id,e.name,e.sal,e.deptno
```



### in

```cypher
MATCH (e:Employee) 
WHERE e.id IN [123,124]
RETURN e.id,e.name,e.sal,e.deptno
```



### 字符串函数

| **S.No.** | **功能**  | 描述                             |
| --------- | --------- | -------------------------------- |
| 1         | UPPER     | 它用于将所有字母更改为大写字母。 |
| 2         | LOWER     | 它用于将所有字母改为小写字母。   |
| 3         | SUBSTRING | 它用于获取给定String的子字符串。 |
| 4         | REPLACE   | 它用于替换一个字符串的子字符串。 |

```cypher
MATCH (e:Employee) 
RETURN e.id,UPPER(e.name),e.sal,e.deptno
                  
MATCH (e:Employee) 
RETURN e.id,SUBSTRING(e.name,0,2),e.sal,e.deptno            
```



### 聚合函数

| **S.No.** | **功能** | 描述                                    |
| --------- | -------- | --------------------------------------- |
| 1         | COUNT    | 它返回由MATCH命令返回的行数。           |
| 2         | MAX      | 它从MATCH命令返回的一组行返回最大值。   |
| 3         | MIN      | 它返回由MATCH命令返回的一组行的最小值。 |
| 4         | SUM      | 它返回由MATCH命令返回的所有行的求和值。 |
| 5         | AVG      | 它返回由MATCH命令返回的所有行的平均值。 |

```cypher
MATCH (e:Employee) RETURN COUNT(*)
                                
MATCH (e:Employee) 
RETURN MAX(e.sal),MIN(e.sal)
                      
MATCH (e:Employee) 
RETURN SUM(e.sal),AVG(e.sal)                      
```



### 关系函数

| **S.No.** | **功能**  | 描述                                     |
| --------- | --------- | ---------------------------------------- |
| 1         | STARTNODE | 它用于知道关系的开始节点。               |
| 2         | ENDNODE   | 它用于知道关系的结束节点。               |
| 3         | ID        | 它用于知道关系的ID。                     |
| 4         | TYPE      | 它用于知道字符串表示中的一个关系的TYPE。 |

```cypher
MATCH (a)-[movie:ACTION_MOVIES]->(b) 
RETURN STARTNODE(movie)
                 
MATCH (a)-[movie:ACTION_MOVIES]->(b) 
RETURN ENDNODE(movie)     
               
MATCH (a)-[movie:ACTION_MOVIES]->(b) 
RETURN ID(movie),TYPE(movie)               
```









## neo4j常识

### id

在Neo4j中，“Id”是节点和关系的默认内部属性。 这意味着，当我们创建一个新的节点或关系时，Neo4j数据库服务器将为内部使用分配一个数字。 它会自动递增。
