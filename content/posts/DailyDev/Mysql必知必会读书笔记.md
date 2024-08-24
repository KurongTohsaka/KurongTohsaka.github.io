---
title: "SQL"
date: 2024-06-26
tags: ["MySQL"]
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
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
---



## 第一章：数据库基础

----------------

数据库：保存有组织的数据的容器（通常是一个文件或一组文件）

数据库软件（DBMS）：MySql，Oracle，MongoDB之类。人们通常用数据库来代替数据库软件的名称

表（table）：某种特定类型数据的结构化清单

模式：关于数据库和表的布局及特性的信息

列（column）：表中的一个字段（该列由字段来唯一标识），所有表都是由一个或多个列组成的，每一列都有自己的数据类型

行（row）：表中的一条数据是由行来存储的

主键（primary key）：唯一标识表中每行的这个列就是主键，应该总是定义主键

> 关于主键：
>
> - 任意两行都不具有相同的主键值
> - 每个行都必须有一个主键值

SQL：结构化查询语言



## 第二章：MySQL简介

--------

略



## 第三章：使用MySQL

-------

1. 登录数据库

   默认主机名：localhost

   默认端口：3306

   ```shell
   mysql -u root -p
   # 然后输入密码
   ```

2. 选择数据库

   ```mysql
   USE database_name;
   ```

3. 了解数据库和表

   - 查看所有数据库

     ```mysql
     SHOW DATABASES;
     ```

   - 查看一个数据库中的所有表

     ```mysql
     SHOW TABLES;
     ```

   - 查看一个表的所有字段

     ```mysql
     SHOW COLUMNS FROM table;
     ```

     还有一种快捷写法

     ```mysql
     DESCRIBE table;
     ```

     在返回的列表中可以看到一些建表信息，如字段名，数据类型，键类型，是否为NULL，默认值，其他类型。

   - 其他的show语句

     ```mysql
     SHOW GRANTS; # 显示授予用户的安全权限
     SHOW ERRORS; # 显示服务器错误
     SHOW WARNINGS; # 显示服务器警告信息
     SHOW STATUS; # 显示服务器的状态信息
     HELP SHOW; # 显示mysql允许的show语句
     ```

     >有一个书写规则：select这样的关键词要大写，表名、列名、数据库名要小写

## 第四章：检索数据

-----------

1. 检索单个列，多个列，全部的列

    ```mysql
   SELECT column FROM table; # 检索单列
   SELECT column1, column2 FROM table; # 检索多列
   SELECT * FROM table; # 检索全部列，*是通配符，匹配所有列
   ```

2. 检索不同的行（所谓‘查重’）

   ```mysql
   SELECT DISTINCT column FROM table; # 关键词DISTINCT就是查重
   ```

3. 限制结果

   ```mysql
   SELECT column FROM table LIMIT number; # number处为任意正整数，结果受到限制，将返回前number行（从第零行开始算起）
   ```

   还可以指定从多少行开始检索，检索多少行

   ```mysql
   SELECT column FROM table LIMIT number1, number2; # 第一个数字是开始位置，第二个是要检索的行数，开始行包括在内
   ```

4. 限定表名

   ```mysql
   SELECT table1.column FROM table2.column;
   ```

   

## 第五章：排序检索数据

------------

子句：SQL语句由子句构成，一个子句通常由一个关键字和所提供的数据组成，如SELECT语句的FROM子句

1. 排序数据

   ```mysql
   SELECT column FROM table ORDER BY column; # 按照字母数字顺序排列，ORDER BY后的column不一定与前面的column相同，可以是其他的列。ORDER BY后的column表示联系此列进行排序
   ```

   多个列的排序

   ```mysql
   SELECT column1, column2 FROM table ORDER BY column1, column2;
   SELECT column1, column2, column3 FROM table ORDER BY column1, column2;
   # 需求不同，形式不同
   ```

2. 指定排序方向

   ```mysql
   # 顺序排序（升序）
   SELECT column FROM table ORDER BY column;
   # 逆序排序（降序）
   SELECT column FROM table ORDER BY column DESC;
   ```

3. 一个小例子

   ```mysql
   SELECT column1 FROM table ORDER BY column2 DESC LIMIT 1; # 表示column1按照column2进行降序排序，并返回第一行的结果
   ```



## 第六章：过滤数据

-------

1. 使用WHERE子句

   只检索所需数据需指定搜索条件，而搜索条件放在WHERE后

   >ORDER BY 需放在 WHERE 后面

2. WHERE的子句操作符

   | 操作符  | 说明               |
   | ------- | ------------------ |
   | =       | 等于               |
   | <>      | 不等于             |
   | !=      | 不等于             |
   | <       | 小于               |
   | <=      | 小于等于           |
   | >       | 大于               |
   | >=      | 大于等于           |
   | BETWEEN | 在指定的两个值之间 |

   - 检测单个值

     ```mysql
     SELECT colmun FROM table WHERE colmun = value;
     SELECT colmun FROM table WHERE colmun < value;
     SELECT colmun FROM table WHERE colmun != value;
     # 基本用法有很多种，大体都是这样
     ```

   - 范围值检测

     ```mysql
     SELECT column FROM table WHERE column BETWEEN number1 AND number2;
     # 检测的范围相当于 number1 <= x <= number2
     ```

   - 空值NULL检测

     ```mysql
     SELECT column FROM table WHERE column IS NULL;
     ```

     

## 第七章：数据过滤

----------

1. AND，OR操作符

   ```mysql
   SELECT column FROM table WHERE column=value1 AND column2=value2; # 交集
   SELECT column FROM table WHERE column=value1 OR column2=value2; # 并集
   # 以上为两个简单实例，仅供参考
   # 当过滤条件中出现多个AND和OR，可以加括号来调整运算优先级
   ```

2. IN，NOT操作符

   ```mysql
   SELECT column FROM table WHERE column IN (value1, value2); # IN用来指示条件范围
   SELECT column FROM table WHERE column NOT IN (value1, value2); # 上面的补集
   
   # 下面一个小例子
   SELECT column FROM table WHERE column IN (value1, value2); # value2-value1=1
   SELECT column FROM table WHERE column=value1 OR column2=value2;
   # 上面的两个SQL语句效果是相同的
   ```

   

## 第八章：用通配符进行过滤

-----------

1. LIKE操作符

   使用通配符必须使用LIKE操作符

2. 部分常用通配符

   - %

     %表示任意字符出现任意次数

     %不匹配NULL

     ```mysql
     SELECT column FROM table WHERE column LIKE 'value%'; # 匹配以value开头的任意字符串
     SELECT column FROM table WHERE column LIKE '%value%'; # 匹配包含value的任意字符串
     ```

   - *

     匹配任意内容

     ```mysql
     SELECT * FROM table; # 返回表中的所有行
     ```

   - _

     匹配任意单一字符

     ```mysql
     SELECT column FROM table WHERE column LIKE 'value_';
     ```

     

## 第九章：用正则表达式进行搜索

------------

1. REGEXP操作符

   使用类似LIKE

2. 正则表达式具体使用

   ```mysql
   SELECT column FROM table WHERE column REGEXP '\w{5}';
   # 一个小例子，所有的使用都与此相似，不多赘述
   ```

   

## 第十章：创建计算字段

---------------

1. 拼接字段

   Concat() 需要一个或多个指定的串

   ```mysql
   SELECT Concat(column1, column2) FROM table; # 将两列的值拼接在一起
   SELECT Concat(column1, '(', column2, ')') FROM table; # 大概是这样 column1(column2)
   ```

2. 去除空白

   Trim(), Rtrim(), LTrim() 分别为去除两侧空白，右侧空白，左侧空白

   ```mysql
   SELECT Concat(Trim(column1), '(', column2, ')') FROM table;
   ```

3. 使用别名

   直接上例子，一看就懂

   ```mysql
   SELECT Concat(column1, '(', column2, ')') AS column3 FROM table;
   ```

4. 执行算术运算

   | 操作符 | 说明 |
   | ------ | ---- |
   | +      | 加   |
   | -      | 减   |
   | *      | 乘   |
   | /      | 除   |

   ```mysql
   SELECT column1, column2, column1*colmun2 AS column3 FROM table; # 挺好懂的，具体使用都差不多
   ```

   

## 第十一章：使用数据处理函数

---------

1. 文本处理函数

   | 函数        | 说明             |
   | ----------- | ---------------- |
   | Left()      | 返回串左侧字符   |
   | Length()    | 返回串的长度     |
   | Locate()    | 找出串的一个子串 |
   | Lower()     | 将串转换为小写   |
   | LTrim()     | 去掉左侧空格     |
   | Right()     | 返回串右侧字符   |
   | RTrim()     | 去掉右侧空格     |
   | SubString() | 返回子串的字符   |
   | Upper()     | 将串转换为大写   |

2. 日期和时间处理函数

   | 函数          | 说明                         |
   | ------------- | ---------------------------- |
   | AddDate()     | 增加一个日期（天，周等）     |
   | AddTime()     | 增加一个时间（时，分等）     |
   | CurTime()     | 返回当前时间                 |
   | Date()        | 返回日期时间的日期部分       |
   | CurDate()     | 返回当前日期                 |
   | DateDiff()    | 计算两个日期之差             |
   | Date_Add()    | 日期运算函数                 |
   | Date_Format() | 返回一个格式化的日期或时间串 |
   | Day()         | 返回一个日期的天数部分       |
   | DayOfWeek()   | 对于一个日期，返回星期几     |
   | Hour()        | 返回一个时间的小时部分       |
   | Minute()      | 返回一个时间的分钟部分       |
   | Month()       | 返回一个日期的月份部分       |
   | Now()         | 返回当前时间和日期           |
   | Second()      | 返回一个时间的秒部分         |
   | Time()        | 返回一个日期时间的时间部分   |
   | Year()        | 返回一个日期的年份部分       |

   一些例子

   ```mysql
   SELECT column FROM table WHERE Date(coulmn) BETWEEN '2008-09-01' AND '2010-09-20';
   SELECT column FROM table WHERE Year(coulmn) BETWEEN '2008' AND '2010';
   # 大体的使用都与这类似
   ```

   

## 第十二章：汇总数据

1. 聚集函数

   概念：运行在行组上，就算和返回单个值的函数。

   | 函数    | 说明     |
   | ------- | -------- |
   | AVG()   | 取平均值 |
   | MAX()   | 求最大值 |
   | MIN()   | 求最小值 |
   | COUNT() | 计数     |
   | SUM()   | 求和     |

   - AVG()

     使用略，注意，该函数会忽略空行。

   - MAX(), MIN()

     使用略。

   - COUNT()

     使用略，该函数会忽略空行，但是COUNT(*)会计算所有行数。

   - SUM()

     使用略。

2. 聚集不同值

   - 聚集所有行，指定ALL参数或不给参数。如AVG(ALL column)
   - 聚集不同行，指定DISTINCT参数。如AVG(DISTINCT column)



## 第十三章：分组数据

------------------

1. 创建分组

   GROUP BY子句。按照逻辑进行结果分组。

   使用了聚集函数，且涉及多行，就必须使用GROUP BY。

   例：

   ```sql
   SELECT vend_id, COUNT(*) AS num_prods
   FROM products
   GROUP BY vend_id;
   ```

   - GROUP BY子句可以包含任意数量的列。
   - GROUP BY子句中列出的每个列都必须是检索列或有效的表达式（不能是聚集函数）。如果在SELECT中使用了表达式，则GROUP BY子句中也必须用同样的表达式，不能用别名。
   - 除聚集计算语句外，每个列都必须在GROUP BY子句中给出。
   - GROUP BY子句出现在WHERE之后，ORDER BY之前。

2. 过滤分组

   HAVING子句。和WHERE子句类似，不过HAVING子句是过滤组的，而WHERE子句是过滤行的。

   用到了分组，涉及到分组的行的WHERE都变成了HAVING。

   例：

   ```sql
   SELECT vend_id, COUNT(*) AS num_prods
   FROM products
   HAVING vend_id > 0
   GROUP BY vend_id;
   ```

   例：

   ```sql
   SELECT vend_id, COUNT(*) AS num_prods
   FROM products
   WHERE price >= 10
   HAVING vend_id > 0
   GROUP BY vend_id;
   ```

3. 分组与排序

   GROUP BY子句的结果无序，常常和ORDER BY一起使用。

   ```sql
   SELECT order_num, SUM(quanlity * item_price) AS ordertotal
   FROM orderitems
   GROUP BY order_num
   HAVING SUM(quanlity * item_price) >= 50
   ORDER BY ordertoal;
   ```



## 第十四章：使用子查询

-------------

其实就是嵌套查询。

看一个例子，后面略：

```sql
SELECT (SELECT vend_id
FROM products
WHERE vend_id > 0), COUNT(*) AS num_prods
FROM products
GROUP BY vend_id;
```



## 第十五章：联结表

-------------------------

1. 联结

   联结是一种机制，用在SELECT中关联表。

2. 创建联结

   - 例：

     ```sql
     SELECT vend.vend_id, COUNT(*) AS num_prods
     FROM products
     WHERE price >= 10;
     ```

   - 内联接

     内联接，又称等值联接，基于两个表间的相等测试。

     ```sql
     SELECT vend_name, prod_name, prod_price
     FROM cendors INNER JOIN products
     ON vendors.vend_id = products.vend_id;
     ```

     

 ## 第十六章：创建高级联结

-------------

1. 使用表别名

   ```sql
   SELECT vend.vend_id, COUNT(*) AS num_prods
   FROM products AS pro
   WHERE price >= 10;
   ```

2. 使用不同类型的联结

   - 自联结

     ```sql
     SELECT p1.prod_id, p1.prod_name
     FROM products AS p1, products AS p2
     WHERE p1.vend_id = p2.vend_id
     AND p2.prod_id = 'DTNTR';
     ```

     使用联结往往比子查询性能更佳。

   - 自然联结

     相对于内联结返回所有数据，甚至有重复行，自然联结没有重复行。

     但是实际上，用到的所有内部联结都是自然联结，所以重复行的情况可能很难遇到。

   - 外部联结

     外连接操作以指定表为连接主体，将主体表中不满足连接条件的行一并输出。

     如果左表的某行在右表中没有匹配行，则在相关联的结果集行中右表的所有选择列表列均为空值。    

     ```sql
     SELECT student.sno, sname, ssex, sage, sdept, cno, grade
     FROM student left outer JOIN sc 
     ON student.sno = sc.sno;
     ```

     - 左外连接

       以左表为连接主体，返回左表所有匹配行。

     - 右外连接

       以右表为连接主体，返回右表所有匹配行。

3. 使用带聚集函数的联结

   ```sql
   SELECT vend_name, COUNT(prod_name), prod_price
   FROM cendors INNER JOIN products
   ON vendors.vend_id = products.vend_id;
   ```

   

## 第十七章：组合查询

-------------

1. 组合查询

   执行多个查询，并将结果作为单个查询结果集返回。

2. 使用Union（并）

   ```sql
   SELECT vend_id, prod_id
   FROM products
   WHERE prod_price <= 5
   UNION
   SELECT vend_id, prod_id
   FROM products
   WHERE vend_id IN (1001, 1002);
   ```

   > 将多个查询结果并在一起

   - Union自动去重。如果不想去重可以用UNION ALL。
   - 每个查询中，字段名、表达式、聚集函数必须相同。

3. 对结果排序

   ```sql
   SELECT vend_id, prod_id
   FROM products
   WHERE prod_price <= 5
   UNION
   SELECT vend_id, prod_id
   FROM products
   WHERE vend_id IN (1001, 1002)
   ORDER BY vend_id;
   ```

   

## 第十八章：全文本搜索

暂略

## 第十九章：插入数据

1. 插入单条记录

   ```sql
   INSERT INTO Student(id, name, sex, grade, age)
   VALUES ('95020'，'陈冬'，'男'，'IS'，18)；
   ```
   
   ```sql
   INSERT
   INTO Student
   VALUES ('95020'，'陈冬'，'男'，'IS'，18)；
   ```
```
   
2. 插入多条记录

   ```sql
   INSERT
   INTO Student
   VALUES ('95020'，'陈冬'，'男'，'IS'，18)
   ('95030'，'陈冬'，'男'，'IS'，19);
```

3. 插入检索结果

   ```sql
   INSERT 
   INTO <表名>  [(<属性列1> [，<属性列2>…  )]
   子查询;
   ```
   
   ```sql
   INSERT
   INTO Deptage(Sdept，Avgage)
   SELECT Sdept，AVG(Sage)
   FROM Student
   GROUP BY Sdept;
   ```
   
   

## 第二十章：更新和删除数据

1. 更新数据

   - 修改某一个元组的值

     ```sql
     UPDATE Student
     SET Sage = 22
     WHERE Sno = '95001';
     ```

   - 修改多个元组的值

     ```sql
     UPDATE Student
     SET Sage = Sage+1
     WHERE Sdept = 'IS';
     ```

   - 带子查询的修改语句

     ```sql
     UPDATE SC
     SET Grade = 0
     WHERE 'CS' =
     (select Sdept
     FROM Student
     WHERE Student.Sno = SC.Sno);
     ```

2. 删除数据

   - 删除某一个元组的值

     ```sql
     DELETE
     FROM Student
     WHERE Sno = '95019';
     ```

   - 删除多个元组的值

     ```sql
     DELETE
     FROM SC
     WHERE Cno = '2';
     ```

   

## 第二十一章：创建和操纵表，创建约束

1. 创建表

   ```sql
   CREATE TABLE Student
   (
       Sno CHAR(5) AUTO_INCREMENT,
       Sname CHAR(20) NULL,       
       Ssex CHAR(1) NULL,
       Sage INT NULL,
       Sdept CHAR(15) NULL,
       PRIMARY KEY(Sno)
   ); 
   ```

   主键可多个，叫做联合主键。

   > AUTO_INCREMENT:自动填充

   在建表语句的末尾其实还有一句``ENGINE = ENGINE_NAME``，代表了所选择的引擎。

   MYSQL默认选择MyISAM，性能极佳，支持全文本搜索，但是不支持事务处理。

   还有其他的几个引擎，如InnoDB，支持事务管理，但是不支持全文本搜索；MYMORY，存储在内存，性能极佳，适合做临时表。

   > 外键不能跨引擎。

2. 创建约束

   - 外键

     一个表中的 FOREIGN KEY 指向另一个表中的 PRIMARY KEY。

     FOREIGN KEY 约束用于预防破坏表之间连接的动作。

     FOREIGN KEY 约束也能防止非法数据插入外键列，因为它必须是它指向的那个表中的值之一。

     ```sql
     CREATE TABLE Orders
     (
     Id_O int NOT NULL,
     OrderNo int NOT NULL,
     Id_P int,
     PRIMARY KEY (Id_O),
     CONSTRAINT fk_PerOrders FOREIGN KEY (Id_P)
     REFERENCES Persons(Id_P)
     )
     ```

     如果在 "Orders" 表已存在的情况下为 "Id_P" 列创建 FOREIGN KEY 约束，请使用下面的 SQL：

     ```sql
     ALTER TABLE Orders
     ADD FOREIGN KEY (Id_P)
     REFERENCES Persons(Id_P)
     ```

     如果需要命名 FOREIGN KEY 约束，以及为多个列定义 FOREIGN KEY 约束，请使用下面的 SQL 语法：

     ```sql
     ALTER TABLE Orders
     ADD CONSTRAINT fk_PerOrders
     FOREIGN KEY (Id_P)
     REFERENCES Persons(Id_P)
     ```

     如需撤销 FOREIGN KEY 约束，请使用下面的 SQL：

     ```sql
     ALTER TABLE Orders
     DROP CONSTRAINT fk_PerOrders
     ```

   - Check

     CHECK 约束用于限制列中的值的范围。

     如果对单个列定义 CHECK 约束，那么该列只允许特定的值。

     如果对一个表定义 CHECK 约束，那么此约束会在特定的列中对值进行限制。

     ```sql
     CREATE TABLE Persons
     (
     Id_P int NOT NULL CHECK (Id_P>0),
     LastName varchar(255) NOT NULL,
     FirstName varchar(255),
     Address varchar(255),
     City varchar(255)
     )
     ```

     - IN

       ```sql
       Create table test(
       Ccity char(15) constraint checcity check( ccity in (‘beijing’,’changchun’)
       )
           
       Alter table test 
       add constraint checcity check(ccity in (‘beijing’,’changchun’))
       ```

     - LIKE

       ```sql
       Create table test(
       Cardid char(20) constraint checard check(cardid like(‘[0-9][a-z][0-9][0-9]’)
       )
       ```

     - BETWEEN AND

       ```sql
       Create table test(
       cardid int constraint checard check(cardid between 0 and 200)
       )
       ```

     如果需要命名 CHECK 约束，以及为多个列定义 CHECK 约束，请使用下面的 SQL 语法：

     ```sql
     CREATE TABLE Persons
     (
     Id_P int NOT NULL,
     LastName varchar(255) NOT NULL,
     FirstName varchar(255),
     Address varchar(255),
     City varchar(255),
     CONSTRAINT chk_Person CHECK (Id_P>0 AND City='Sandnes')
     )
     ```

     如果在表已存在的情况下为 "Id_P" 列创建 CHECK 约束，请使用下面的 SQL：

     ```sql
     ALTER TABLE Persons
     ADD CHECK (Id_P>0)
     ```

     如果需要命名 CHECK 约束，以及为多个列定义 CHECK 约束，请使用下面的 SQL 语法：

     ```sql
     ALTER TABLE Persons
     ADD CONSTRAINT chk_Person CHECK (Id_P>0 AND City='Sandnes')
     ```

     如需撤销 CHECK 约束，请使用下面的 SQL：

     ```sql
     ALTER TABLE Persons
     DROP CONSTRAINT chk_Person
     ```

   - Default

     - DEFAULT 约束用于向列中插入默认值。

       如果没有规定其他的值，那么会将默认值添加到所有的新记录。

       ```sql
       CREATE TABLE Persons
       (
       Id_P int NOT NULL,
       LastName varchar(255) NOT NULL,
       FirstName varchar(255),
       Address varchar(255),
       City varchar(255) DEFAULT 'Sandnes'
       )
       ```

       通过使用类似 GETDATE() 这样的函数，DEFAULT 约束也可以用于插入系统值：

       ```sql
       CREATE TABLE Orders
       (
       Id_O int NOT NULL,
       OrderNo int NOT NULL,
       Id_P int,
       OrderDate date DEFAULT GETDATE()
       )
       ```

       如果在表已存在的情况下为 "City" 列创建 DEFAULT 约束，请使用下面的 SQL：

       ```sql
       ALTER TABLE Persons
       ALTER COLUMN City SET DEFAULT 'SANDNES'
       ```

       如需撤销 DEFAULT 约束，请使用下面的 SQL：

       ```sql
       ALTER TABLE Persons
       ALTER COLUMN City DROP DEFAULT
       ```

     

3. 更新表

   例1：向Student表增加“入学时间”列，其数据类型为日期型。

   ```sql
   ALTER TABLE Student 
   ADD Scome DATE;
   ```

   例2：删除属性列。

   ```sql
   Alter table table_name 
   drop column column_name;
   ```

   例3：将年龄的数据类型改为整数。

   ```sql
   ALTER TABLE Student 
   MODIFY Sage INT;
   ```

   例4：删除学生姓名必须取唯一值的约束。

   ```sql
   ALTER TABLE Student DROP unisname;
   ```

4. 删除表

   ```sql
   DROP TABLE table_name
   ```

   DROP TABLE 不能用于除去由 FOREIGN KEY 约束引用的表。必须先除去引用的 FOREIGN KEY 约束或引用的表。

   表所有者可以除去任何数据库内的表。除去表时，表上的规则或默认值将解除绑定，任何与表关联的约束或触发器将自动除去。如果重新创建表，必须重新绑定适当的规则和默认值，重新创建任何触发器并添加必要的约束。

5. 重命名表

   ```sql
   RENAEM TABLE test1 TO test2;
   ```

   

## 第二十二章：使用视图

1. 视图

   视图是一个虚拟的表，是一个表中的数据经过某种筛选后的显示方式，视图由一个预定义的查询select语句组成。

   视图的特点：

   - 视图中的数据并不属于视图本身，而是属于基本的表，对视图可以像表一样进行insert,update,delete操作。
   - 视图不能被修改，表修改或者删除后应该删除视图再重建。
   - 视图的数量没有限制，但是命名不能和视图以及表重复，具有唯一性。
   - 视图可以被嵌套，一个视图中可以嵌套另一个视图。
   - 视图不能索引，不能有相关联的触发器和默认值。
   - 视图可以和表一起使用。

2. 使用视图

   - 创建视图
   
     ```sql
     CREATE  VIEW <视图名>  [(<列名>  [，<列名>]…)] AS  <子查询>
     ```
     
   
   例：
   
     ```sql
     CREATE VIEW IS_Student AS 
     SELECT Sno，Sname，Sage
     FROM Student
   WHERE Sdept= 'IS';
     ```
   
     - 基于视图的视图
     
       ```sql
       CREATE VIEW IS_S2 AS
       SELECT Sno，Sname，Grade
       --这是一个视图
       FROM IS_S1
       WHERE Grade >= 90;
       ```
     ```
     
   - 查询视图
   
     ```sql
     SELECT Sno，Sage
     FROM IS_Student
     WHERE Sage < 20;
     ```
   
   - 更新视图
   
     ```sql
     UPDATE IS_Student
     SET Sname = '刘辰'
     WHERE Sno = '95002';
     ```
   
   - 删除视图
   
     ```sql
     DROP VIEW <视图名>;
     ```
   
     

## 第二十三章：使用存储过程

1. 存储过程

   将常用的或很复杂的工作，预先用SQL语句写好并用一个指定的名称存储起来, 那么以后要叫数据库提供与已定义好的存储过程的功能相同的服务时,只需调用execute,即可自动完成命令。

   优点：

   - 存储过程只在创造时进行编译，以后每次执行存储过程都不需再重新编译，而一般SQL语句每执行一次就编译一次,所以使用存储过程可提高数据库执行速度。
   - 当对数据库进行复杂操作时(如对多个表进行Update,Insert,Query,Delete时)，可将此复杂操作用存储过程封装起来与数据库提供的事务处理结合一起使用。
   - 存储过程可以重复使用,可减少数据库开发人员的工作量。
   - 安全性高,可设定只有某此用户才具有对指定存储过程的使用权。

2. 使用存储过程

   注意每一句sql都要以；结尾。
   
   - 简单的创建存储过程
   
     ```sql
     CREATE PROCEDURE prcPrintRecruitmentAgencyList()
     BEGIN
     		SELECT cName, vAddress, cCity, cZip, cPhone, cFax 
     		FROM RecruitmentAgencies;
     END;
     ```
   
   - 运行存储过程
   
     ```sql
     CALL processing_name(@variable);
     ```
   
   - 创建带输入参数的存储过程
   
     ```sql
     CREATE PROCEDURE productpricing(
         OUT pl DECIMAL(8, 2),
         OUT ph DECIMAL(8, 2),
         OUT pa DECIMAL(8, 2),
     )
     BEGIN
     SELECT Min(prod_price)
     INTO pl
     FROM products;
     SELECT Max(prod_price)
     INTO ph
     FROM products;
     SELECT AVG(prod_price)
     INTO pa
     FROM products;
     END;
     ```
   
     关键字OUT指出相应的参数用来从存储过程传出一个值返回给调用者。MySQL支持IN（传递给存储过程）、OUT、INOUT（对存储过程传入传出）。
   
     通过INTO将值传入参数。
   
     > `DECIMAL`数据类型用于在数据库中存储精确的数值。我们经常将`DECIMAL`数据类型用于保留准确精确度的列，例如会计系统中的货币数据。
     >
     > 要定义数据类型为`DECIMAL`的列，请使用以下语法：
     >
     > ```sql
     > column_name ``DECIMAL``(P,D);
     > ```
     >
     > 在上面的语法中：
     >
     > - `P`是表示有效数字数的精度。 `P`范围为`1〜65`。
     > - `D`是表示小数点后的位数。 `D`的范围是`0`~`30`。MySQL要求`D`小于或等于(`<=`)`P`。
     >
     > `DECIMAL(P，D)`表示列可以存储`D`位小数的`P`位数。十进制列的实际范围取决于精度和刻度。
   
     运行上面的例子：
   
     ```sql
     CALL productpricing(
         @pricelow,
         @pricehigh,
         @priceaverage
         );
     ```
   
     如果想检索变量的值：
   
     ```sql
     SELECT @pricelow, @pricehigh, @priceaverage;
     ```
   
     另一个例子：
   
     ```sql
     CREATE PROCEDURE ordertotal(
         IN onumber INT,
         OUT ototal DECIMAL(8, 2)
     )
     BEGIN
     SELECT Sum(item_price * quanlity)
     FROM orderitems
     WHERE order_num = onumber
     INTO ototal;
     END;
     ```
   
     运行：
   
     ```sql
     CALL ordertotal(20005, @total);
     ```
   
     结果：
   
     ```sql
     SELECT @total;
     ```
   
   - 删除存储过程
   
     ```sql
     DROP PROCEDURE processing_name;
     ```
   
   - 建立智能存储过程
   
     ```sql
     DELIMITER $$
     CREATE PROCEDURE ordertotal(
         IN onumber INT,
         IN taxable BOOLEAN,
         OUT ototal DECIMAL(8, 2)
     )
     BEGIN
     DECLARE total DECIMAL(8, 2);
     DECLARE taxrate INT DEFAULT 6;
     SELECT Sum(item_price * quanlity)
     FROM orderitems
     WHERE order_num = onumber
     INTO ototal;
     --Is this taxable?
     IF taxable THEN
     SELECT total + (tatal / 100 * taxrate);
     END IF;
     SELECT total INTO ototal;
     END $$
     DELIMITER;
     ```
   
     > DELIMITER $$ 声明分隔符为$，而在语句中；还是语句的结尾，而且是存储过程的一部分。要是不加的话，应用会认为；不是存储过程的一部分，会报错。
   
   - 检查存储过程
   
     获得包括何时、有谁创建等详细信息的存储过程列表
   
     ```sql
     SHOW PROCEDURE STATUS;
     SHOW PROCEDURE STATUS LIKE 'SAD';--获得特定的值
     ```
   
     

## 第二十四章：语句的分割，流程控制，变量

1. 语句分块

   BEGIN, END用来限定存储过程体，相当于java里的'{ }'

2. 流程控制

   - if，else

     ```sql
     CREATE PROCEDURE PRC()
     BEGIN
     DECLARE NUMBER INT;
     SET NUMBER = 10;
     IF NUMBER > 5 THEN
     SELECT NUMBER;
     ELSE
     SELECT NUMBER;
     END IF;
     END
     ```

   - while

     使用方法与其他语言类似。

     ```sql
     DECLARE i int;
     SET i = 1
     WHILE i < 30
     INSERT INTO test (userid) 
     VALUES(i);
     SET i = i+1;
     END WHILE
     ```

   - go

     go是用来分开多条sql语句的，一个执行成功后才会执行下一个。

   - return

     返回值，或结束程序。

3. 变量的定义与赋值

   - 定义

     ```sql
     DECLARE local_variable data_type;
     ```

   - 赋值

     ```sql
     SET local_variable = variavle_value;
     ```

     ```sql
     SELECT variavle_value INTO local_variable;
     ```

     

## 第二十五章：使用游标

1. 游标

   有时需要在检索出来的行中前进或后退一行或多行，这就是使用游标的原因。而游标，就是一个存储在服务器上的数据库查询，它不是一条SELECT语句，而是被检索出来的结果集。

   在MySQL中，游标只能用于存储过程和存储函数。

2. 使用游标

   几个步骤：
   
   - 先定义游标。
   - 打开游标以供使用。
   - 对于有数据的游标，根据需要取出数据。
   - 结束游标时，关闭游标。
   
   
   
   - 创建游标
   
     ```sql
     CREATE PROCEDURE processorders()
     BEGIN
     	DECLARE ordernumbers CURSOR
     	FOR
     	--对下面的查询建立游标
     	SELECT order_num FROM ordeRs;
     END;
     declare asd cursor 
     for select * from table;
     ```
   
     用DECLARE来声明游标。
   
   - 打开和关闭游标
   
     ```sql
     --打开游标
     OPEN CURSOR;
     --关闭游标
     CLOSE CURSOR;
     ```
   
     和上一条的例子结合：
   
     ```sql
     CREATE PROCEDURE processorders()
     BEGIN
     	DECLARE ordernumbers CURSOR
     	FOR
     	SELECT order_num FROM ordeRs;
     	OPEN ordernumbers;
     	CLOSE ordernumbers;
     END;
     ```
   
   - 使用游标数据
   
     - 检索单个行
   
       ```sql
       CREATE PROCEDURE processorders()
       BEGIN
       	DECLARE o INT;
       	DECLARE ordernumbers CURSOR
       	FOR
       	SELECT order_num FROM ordeRs;
       	OPEN ordernumbers;
       	FETCH ordernumbers INTO o;
       	CLOSE ordernumbers;
       END;
       ```
   
       用fetch将第一行数据放入变量o中。
   
     - 循环检索行（全部行）
   
       ```sql
       CREATE PROCEDURE processorders()
       BEGIN
       	DECLARE o INT;
       	DECLARE done BOOLEAN DEFAULT 0;
       	DECLARE ordernumbers CURSOR
       	FOR
       	SELECT order_num FROM ordeRs;
       	DECLARE CONTINUE HANDLER FOR SQLSTATE '02000' SET done = 1;
       	OPEN ordernumbers;
       	REPEAT
       		FETCH ordernumbers INTO o;
       	UNTIL done END REPEAT;
       	CLOSE ordernumbers;
       END;
       ```
   
       先是用DECLARE声明了一个CONTINUE HANDLER，条件是SQLSTATE 为'02000'时，SET done = 1。
   
       然后在REPEAT区块内，重复执行fetch，直到done = 1才会结束重复。
   
       这样就是循环检索所有行来获得目标记录。
   
     - 综合例子
   
       ```sql
       CREATE PROCEDURE processorders()
       BEGIN
       	DECLARE o INT;
       	DECLARE done BOOLEAN DEFAULT 0;
       	DECLARE t DECIMAL(8, 2);
       	DECLARE ordernumbers CURSOR
       	FOR
       	SELECT order_num FROM ordeRs;
       	DECLARE CONTINUE HANDLER FOR SQLSTATE '02000' SET done = 1;
       	CREATE TABLE IF NOT EXISTS ordertotals
       	(order_num INT, total DECIMAL(8, 2));
       	OPEN ordernumbers;
       	REPEAT
       		FETCH ordernumbers INTO o;
       		--调用存储过程进行计算和检查
       		CALL ordertotal(o, 1, t);
       		INSERT INTO ordertotals(order_num, total) VALUES (o, t);
       	UNTIL done END REPEAT;
       	CLOSE ordernumbers;
       END;
       ```
   
       

## 第二十六章：使用触发器

1. 触发器

   触发器是MySQL响应以下任意语句而自动执行的一条SQL语句：

   - update
   - dalete
   - insert

2. 创建触发器

   ```sql
   --AFTER 代表这是一个后触发器
   CREATE TRIGGER newproduct AFTER INSERT ON products
   FOR EACH ROW SELECT 'PROduct added';
   --FOR EACH ROW 对每一行都起作用
   ```

   还有before 前触发器。

3. 删除触发器

   ```sql
   DROP TRIGGER newproduct;
   ```

4. 使用触发器

   - INSERT触发器

     在INSERT触发器代码内，可以引用一个名为NEW的虚拟表，访问被插入的行。

     在前触发器中，NEW中的值可以被更新。

     对于AUTO_INCREMENT列，NEW在INSERT执行前包含0，执行之后包含新的自动生成值。

     ```sql
     CREATE TRIGGER neworder AFTER INSERT ON orders
     FOR EACH ROW SELECT NEW.order_num;
     ```
  ```
   
  > BEFORE触发器适用于数据验证和净化，AFTER触发器适用于操作后的输出、处理。
   
- DELETE触发器
   
  在DELETE触发器代码内，可以引用一个名为OLD的虚拟表，访问被插入的行。
   
  OLD表中的值为只读，不可更新。
   
     ```sql
     CREATE TRIGGER deleteorder BEFORE DELETE ON orders
     FOR EACH ROW
     BEGIN
     	INSERT INTO archive_orders(order_num, order_date)
     	VALUES(OLD.order_num, OLD.order_date);
     END;
  ```

   - UPDATE触发器
   
     在UPDATE触发器代码内，可以引用一个名为OLD的虚拟表，访问更新前的值。可以引用一个名为NEW的虚拟表，访问更新后的值。
     
     OLD表中的值为只读，不可更新。
     
     在BEFORE触发器中，new中的值也可能会被更新。
     
     ```sql
     CREATE TRIGGER updatevendor BEFORE UPDATE ON vendors
     FOR EACH ROW SET NEW.vend_state = Upper(NEW.vend_state);
     ```

5. 一些事项

   - MySQL中，在触发器内部，无法使用CALL来调用存储过程，只能将代码复制粘贴。
   - 应该用触发器保证数据的一致性。



## 第二十七章：管理事务处理

1. 事务处理

   用一个例子来了解：

   - 检查数据库是否存在相应的客户，如果不存在，添加它。
   - 提交客户信息。
   - 检索客户ID。
   - 添加一行到表中。
   - 如果在添加到表中的过程出现故障，回退。
   - 在表中检索新ID。
   - 对于订购的每一项物品，添加新行到另一个表中。
   - 如果在添加到表中的过程出现故障，回退所有添加的行。
   - 提交订单信息。

   事务（transaction）：一组SQL语句。

   回退（rollback）：撤销SQL语句的过程。

   提交（commit）：将未存储的SQL语句存储到数据库表。

   保留点（savepoint）：事务处理中设置的临时占位符，可以对它发布回退。

2. 控制事务处理

   - ROLLBACK
   
     ```sql
     SELECT * FROM ordertotals;
     START TRANSACTION;
     DELETE FROM ordertotals;
     SELECT * FROM ordertotals;
     ROLLBACK;
     SELECT * FROM ordertotals;
     ```
   
     drop和create语句不能回退。
   
   - COMMIT
   
     ```sql
     START TRANSACTION;
     DELETE FROM orders;
     COMMIT;
     --只有delete执行成功才会对表进行更改
     ```
   
   - SAVEPOINT
   
     ```sql
     SELECT * FROM ordertotals;
     START TRANSACTION;
     DELETE FROM ordertotals;
     SAVEPOINT rollback1
     SELECT * FROM ordertotals;
     SAVEPOINT rollback2
     ROLLBACK TO rollback1;
     SELECT * FROM ordertotals;
     ```
   
   - 更改默认的提交行为
   
     ```sql
     SET autocommit = 0;
     ```
   

