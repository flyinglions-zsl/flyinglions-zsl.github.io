---
title: mysql索引学习
date: 2021-05-19 22:54:33
categories:
  - Mysql学习
tags:
  - 索引
---

https://www.cs.usfca.edu/~galles/visualization/Algorithms.html



# 索引

**索引就是一种快速检索的、排好序的数据机构。**

本质：数据结构+算法

## 类别及数据结构

### Hash表

能够快速查找O(1)，但是会发生hash冲突，且不支持范围查询、排序，只支持=、in。

### 二叉树

一颗相对平衡的有序二叉树，可进行插入、查找、删除。O(logn)

缺点：存在致命的倾斜性问题，如 

果数据往大从小插入，或是由小从大插入 ，都会**导致倾斜**。即最大化查询。

### 红黑树

相对二叉树更加平衡，查询的次数也相对可以接受，但是红黑树自旋后还是会发生倾斜性的问题。

### B-Tree

叶节点具有相同的深度，叶节点的指针为空；

所有索引元素不重复；

节点中的数据索引从左到右**递增排列**

### B+Tree(B-Tree加强)

非叶子节点不存储data，只存储索引(冗余)，可以放更多的索引

叶子节点包含所有索引字段

每个节点都有指定的degree，相邻节点靠指针关联(链表)，**值存在叶子节点上**。



## 引擎下实现

**mysql索引时如何存储的： 引擎是表级别的，持久的，是一个文件，存储在磁盘上** 

### MyISAM（非聚集）

不支持事务、也不支持外键，优势是访问速度快，对事务完整性没有 要求或者以select，insert为主的应用基本上可 以用这个引擎来创建表，支持FULLTEXT索引，并且只为CHAR、VARCHAR和TEXT列。 

frm：创建表的文件 

myd：表的数据文件 

myi：表的索引文件，存的地址 

索引查找： 以id为索引的B+tree（myi文件），得到**id对应的物理地址(叶子结点上)**，然后再去myd文件查找对应的数据。以字符型，中文的为索引，存的就是ASCII，重复度很高不建议用索引(男女)，磁盘IO还是会发生，浪费资源。



### InnoDB（聚集）

该存储引擎提供了具有提交、回滚和崩溃恢复能力的事务安全。但是对比MyISAM引擎，写的处理效率会差一些，并 

且会占用更多的磁盘空间以保留数据和索引。 

InnoDB存储引擎的特点：支持自动增长列，支持外键约束 

聚集索引 ：插入的时候会按主键排序 

frm：创建表的文件 

ibd：索引+数据 

索引查找:以id为索引的B+tree（myi文件），叶子节点存的是id+数据（如1，a） 

​       以字符型，中文的(username)为索引，叶子节点存的是id；降低存储空间，减少索引创建

叶子结点每页默认是16k数据。



为什么建议InnoDB表必须建主键，并且推荐使用整型的自增主键？

答：mysql在建表时，会主动寻找某一个没有相同数据的列，没有则会创建一个列，相当于rowid，会浪费时间，而且在索引查找的时候，B+Tree存的是记录的ID，如果是rowid不利于查找。

自增主键，涉及了B+Tree数叶子结点的裂变，当叶子结点存不了数据时，会新开一个节点，这时候使用自增id，效率更高，如果用uuid等，还需要每次都去比较，而且uuid字符比较可能会消耗更多的时间。

为什么非主键索引结构叶子节点存储的是主键值？(一致性和节省存储空间)

答：非主键索引，比如二级索引，字符列name加的索引，叶子结点上key是name值，value是对应的记录ID，因为存的是ID，可以进行回表查询操作，更精确地查找记录，保证数据的一致性。而且不用像唯一索引一样去维护整个数据，可以节省内存空间。（没新增一个索引就多一个B+Tree的结构）



## 联合索引

多个列进行的组合索引，查询需符合最左前缀原则。

例：现有一个索引 a_b_c -->以下组合可用 ： a,b,c ; a,b ; a 三种情况



底层数据结构：

也是B+Tree实现，不过非叶子结点存的是组合的值，如name、age，Alice 12， Bob 14，先对name进行排序，再对age进行排序。







# Explain详解与索引

简述: explain为mysql提供语句的执行计划信息。可以应用在select、delete、insert、update和place语句上。explain的执行计划，只是作为语句执行过程的一个参考，实际执行的过程不一定和计划完全一致，但是执行计划中透露出的讯息却可以帮助选择更好的索引和写出更优化的查询语句。





官方文档：https://dev.mysql.com/doc/refman/5.7/en/explain-output.html 



## Explain的列说明

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1621523649211-b302f4a2-bc10-465c-af40-3fc945147c66.png)

### id列

id列的编号是select的序列号，一般有多少个select查询就有多少个id，且id是按照select出现的顺序增长的，id越大则select执行的优先级就越高。

id列也可以为null，当查询为union时，id为null。在这种情况下，table列会显示为形如<union M,N>，表示它是id为M和N的查询行的联合结果。



### select_type列

顾名思义，此列表示的是查询的种类。

1. simple：简单的查询，不使用uinon或者子查询。如下示例

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1621567776325-ac254778-daf6-4c43-b317-de924c28f88b.png)

**示例1-1**

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1621576969959-92750360-d476-4128-aeee-bcb8a9781ddb.png)

1. primary：一个需要union操作或者含有子查询的select，位于最外层的单位查询的select_type即为primary。且只有一个。如示例1-1中 第一个出现的select，即id=1的
2. derived：派生表SELECT（FROM子句中的子查询）。如示例1-1中 FROM 后面的（select * from ...) emp

1. subquery：除了from子句中包含的子查询外，其他地方出现的子查询都可能是subquery。如示例1-1中 FROM前面出现的(select 1 ....)
2. union：union连接的select查询，除了第一个表外，第二个及以后的表select_type都是union。

1. union result：包含union的结果集，在union和union all语句中,因为它不需要参与查询，所以id字段为null



### table列

输出行所引用的表的名称。显示的查询表名，如果查询使用了别名，那么这里显示的是别名，如果不涉及对数据表的操作，那么这显示为null。

<unionM，N>：该行引用ID值为M和N的行的并集。
<derivedN>：该行引用ID值为N的行的派生表结果。派生表可能来自例如FROM子句中的子查询。
<subqueryN>：该行引用ID值为N的行的实例化子查询的结果。

如示例1-1中id=1的table列为 <derived3>派生表(临时表),代表其使用的是id为3的结果集，即from后的子查询。



### partitions列

版本5.7以前，该项是explain partitions显示的选项，5.7以后成为了默认选项。该列显示的为分区表命中的分区情况。非分区表该字段为空（null）。



### type列

描述如何联接表，即如何查找表中的行，以及数据范围等。

依次从好到差：system>const>eq_ref>ref>fulltext>ref_or_null

​      	>index_merge>unique_subquery>index_subquery>range>index>ALL

除了all之外，其他的type都可以使用到索引，除了index_merge之外，其他的type只可以用到一个索引。

1.system:表中只有一行数据时，且其是const的一个特例。 EXPLAIN EXTENDED 结合show warnings时可以看出，mysql会将查询某些部分进行优化-将其转换为一个常量查询

```
EXPLAIN EXTENDED SELECT * FROM (SELECT * FROM employee WHERE id = 1) emp;
SHOW WARNINGS;
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1621579604732-6edc4d9b-a2b7-44a0-acf6-ddf90b0eee1a.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1621579620332-39d19758-9d31-4e9f-b802-ea90d2b090d3.png)

2.const:如上面的示例，id=2时，使用唯一索引(unique)或者主键(primary)，返回记录一定是1行记录的等值where条件时，通常type是const，也可称之为唯一索引扫描。

3.eq_ref:当连接使用索引的所有部分并且索引是PRIMARY KEY或UNIQUE NOT NULL索引时，将使用它。

驱动表只返回一行数据，且这行数据是第二个表的主键或者唯一索引，且必须为不为空，当唯一索引和主键多列时，只有**所有的列都用作比较**时才会出现eq_ref。

```
EXPLAIN SELECT * FROM employee emp 
LEFT JOIN department dept ON emp.departmentId = dept.id;

EXPLAIN SELECT * FROM employee emp ,department dept 
WHERE emp.departmentId = dept.id;
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1621580342967-c1b46499-f12d-433a-a8f7-0947c02adf30.png)

4.ref:如果联接仅使用键的最左前缀，或者如果键不是PRIMARY KEY或UNIQUE索引（换句话说，如果联接无法基于键值选择单个行）则为ref。

```
KEY `idx_empId_deptId` (`departmentId`,`id`) #组合索引
KEY `departmentId` (`departmentId`) #普通索引
```

示例1:不是PRIMARY KEY或UNIQUE索引

```
EXPLAIN SELECT * FROM employee WHERE departmentId = 1;
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1621581472214-779d5d96-4a50-4545-9223-19fc2f438dce.png)

示例2:索引的最左前缀

```
EXPLAIN SELECT * FROM department dept 
LEFT JOIN employee emp ON dept.id = emp.departmentId;
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1621582019682-b8279e4f-a11a-4d7f-9583-4735e802a202.png)

5.fulltext：全文索引检索，全文索引的优先级很高，若全文索引和普通索引同时存在时，mysql不管代价，优先选择使用全文索引。

6.ref_or_null：与ref方法类似，只是增加了null值的比较。

7.range：索引范围扫描，常见于使用 =, <>, >, >=, <, <=, IS NULL, <=>, BETWEEN, IN()或者like等运算符的查询中。

```
EXPLAIN SELECT * FROM employee WHERE id > 1;
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1621582275175-523ccf62-2cd2-481e-a0b3-60bb52cd9df7.png)

8.index:

- 如果索引是查询的覆盖索引，并且可用于满足表中所需的所有数据，则仅扫描索引树(不从根节点开始查找，直接对二级索引的叶子结点遍历和扫描)。在这种情况下，“extra”列显示“using index”。仅索引扫描通常比ALL更快，因为索引的大小通常小于表数据。

```
EXPLAIN SELECT departmentId FROM employee ;#departmentId为普通索引
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1621582955239-4b98b668-af40-4dc9-afdc-71d0118ac14b.png)

- 在索引上进行全表扫描，没有Using index的提示

9.ALL:全表扫描

```
EXPLAIN SELECT * FROM employee
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1621583382561-3a644f8f-c117-43e8-9566-88ef4b370e8a.png)

### possibile_keys列

显示查询可能使用哪些索引来查找。

- 该列显示null时：说明当前查询情况，没有索引可以选择。
- 该列不显示null时：但key 显示可能为null，因为可能表中数据不多，mysql直接走了全表查询。



### key列

显示实际使用的哪个索引来进行表查询的。

select_type为index_merge时，这里可能出现两个以上的索引，其他的select_type这里只会出现一个。



### key_len列

显示索引使用的字节数，通过这个值来计算使用了索引中的哪些列。

```
KEY `idx_empId_deptId` (`departmentId`,`id`) #组合索引
EXPLAIN SELECT * FROM employee WHERE departmentId = 1;
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1621581472214-779d5d96-4a50-4545-9223-19fc2f438dce.png)

departmentId是int 4个字节(这里展示5是因为建表时默认给了null值，占了一个字节)，这里本身就有一个索引，没走组合索引。



key_len计算规则如下: 

- 字符串，char(n)和varchar(n)，5.0.3以后版本中，n均代表字符数，而不是字节数，如果是**utf-8**，一个数字 或字母占1个字节，一个汉字占3个字节

- char(n):如果存汉字长度就是 3n 字节 
- varchar(n):如果存汉字则长度是 3n + 2 字节，加的2字节用来存储字符串长度，因为 varchar是变长字符串

- 数值类型 

- tinyint:1字节 
- smallint:2字节

- int:4字节
- bigint:8字节  

- 时间类型 

- date:3字节 
- timestamp:4字节 

- datetime:8字节如果字段允许为 NULL，需要1字节记录是否为 NULL 

索引最大长度是768字节，当字符串过长时，mysql会做一个类似左前缀索引的处理，将前半部分的字符提取出来做索 引。



### ref列

显示了在key列记录的索引中，表查找值所用到的列或常量，常见的有:const(常量)，字段名(例:emp.id) 



### rows列

行计划中估算的扫描行数，不是精确值



### filtered列

explain的extended 扩展能够在原本explain的基础上额外的提供一些查询优化的信息，这些信息可以通过mysql的show warnings命令得到。(5.7之后的版本默认就有这个字段，不需要使用explain extended了。这个字段表示存储引擎返回的数据在server层过滤后，剩下多少满足查询的记录数量的比例，注意是**百分比**，不是具体记录数)

```
EXPLAIN [EXTENDED] SELECT select_options 
```

rows * filtered/100 可以估算出将要和 explain 中前一个表进行连接的行数(前一个表指 explain 中的id值比当前表id值小的 表)

当为100时代表的基本是进行了全表扫描，值越小越好。

### Extra列

显示的是额外的信息。

官网有句话，大概翻译为：如果你想要优化你的查询，那就要注意extra辅助信息中的using filesort和using temporary，这两项非常消耗性能，需要注意。

常见信息：

1.using index：explain结果的key里有使用索引，如果此时select的字段都能通过这个索引(索引树里)来查找到，一般值得就是覆盖索引(一般针对的是辅助索引，不再需要通过辅助索引拿到主键)，即查询时不需要回表查询，直接通过索引就可以获取查询的数据。

2.using where:使用了where子句来查询，并且select查询的列没有被索引覆盖到。

3.Using index condition:查询的列不完全被索引覆盖，where条件中是一个前导列的范围。

```
KEY `idx_workAge_birthday` (`workAge`,`birthday`)
EXPLAIN SELECT * FROM employee WHERE workAge > 1;
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1621586226271-8761a899-e453-4dc7-8bca-382a1493d5d0.png)

4.using temporary：表示使用了临时表存储中间结果。临时表可以是内存临时表和磁盘临时表，执行计划中看不出来，需要查看status变量，used_tmp_table，used_tmp_disk_table才能看出来。

5.using filesort：排序时无法使用到索引时，就会出现这个。常见于order by和group by语句中。数据较小时从内存排序，否则需要在磁盘完成排序。 

a.排序的字段未建立索引

```
EXPLAIN SELECT * FROM employee ORDER BY address;
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1621586513267-219dcd99-7af3-428b-8c22-a2d169b8abd9.png)

此处address字段未建立索引，进行全表扫描，保存address和对应的id，再排序address并检索记录。

b.排序的字段建立了索引

```
KEY `idx_address` (`address`)
EXPLAIN SELECT id,address FROM employee ORDER BY address;
#这里select的字段基本只能包含id和当前索引字段，不然依旧是using filesort
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1621587076006-a87d69ed-9e09-4984-92c3-c699035114f1.png)

1） 如果select 只查询索引字段，order by 索引字段会用到索引，要不然就是全表排列；

2） 如果有where 条件，比如where xx=1 order by 索引字段 asc . 这样order by 也会用到索引！

6.Select tables optimized away:使用某些聚合函数(比如 max、min)来访问存在索引的某个字段。table列为null



## 索引使用及注意事项

1.全值匹配

```
explain select * from employees where name = 'Lucy';
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1621608020058-7c7e6bfe-0a9f-4da5-80de-a42ce2db9a19.png)

ref为const

2.最左前缀法则

当索引是组合索引时，需遵守最左前缀法则，即查询组合索引中的列时，从最左侧开始不能跳过列，否则索引会失效

```
explain select * from employees where name = 'Lucy' and age=23 and position='dev';
explain select * from employees where name = 'Lucy' and age=23;
explain select * from employees where name = 'Lucy';
explain select * from employees where age=23;
explain select * from employees where position='dev';
explain select * from employees where age=23 and position='dev';
```

只有前三种是符合法则的，走了索引

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1621610072108-ab69e7e3-dea0-4fff-8739-234d3d07f471.png)

### 3.不在索引列上做任何操作（计算、函数、（自动or手动）类型转换），会导致索引失效而转向全表扫描 

```
explain select * from employees where left(name,2) = 'Lu';
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1621612237417-cc5614e9-7ad3-42a6-a27b-69bcf354d0f8.png)

### 4.存储引擎不能使用索引中范围条件右边的列 

组合索引时，比如有三个字段组合

```
explain select * from employees where name = 'Lucy' and age>23 and position='dev';
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1621613942922-ebd6f299-8a06-45eb-adc3-bda270fe1591.png)

从key_len可以看出，全部字段使用时是140，目前只有78，得知postion没有走索引。

mysql5.6版本之前没有加入index condition pushdown，所以索引逻辑还是这样的：

即便对于复合索引，从第一列开始先确定第一列索引范围，如果范围带=号，则对于=号情况，确定第二列索引范围加入索引结果集里，每列的处理方式都是一样的。

确定完索引范围后，则回表查询数据，再用剩下的where条件进行过滤判断。

mysql5.6后加入了ICP，对于确定完了索引范围后，会用剩下的where条件对索引范围再进行一次过滤，然后再回表，再用剩下的where条件进行过滤判断。（减少回表记录数量）。

### [ ](https://blog.csdn.net/Mypromise_TFS/article/details/68945293)5.尽量使用覆盖索引

（只访问索引的查询（索引列包含查询列）），减少 select * 语句

### select的字段是索引所相关的字段，不然无法是覆盖索引，只是const级别的查询而已。[  ](https://blog.csdn.net/Mypromise_TFS/article/details/68945293)6.mysql在使用不等于（！=或者<>），not in ，not exists 的时候无法使用索引会导致全表扫描 

< 小于、 > 大于、 <=、>= 这些，mysql内部优化器会根据检索比例、表大小等多个因素整体评估是否使用索引

```
explain select * from employees where name != 'Lucy' ;
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1621656814033-405a96f2-4f01-4e4f-8a2b-7ef08e4b3672.png)

### 7.is null 、 is not null 一般也会使索引失效

```
explain select * from employees where age is not null ;
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1621656901633-510e26df-0a55-46de-8fa2-932e038093c1.png)

### 8.字符串不添加单引号也会使索引失效

```
explain select * from employees where name = 20;
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1621657038075-13948ef6-1c9f-44b1-bd36-3782a28cda12.png)



### 9.模糊查询-like 用通配符开头('%xxx...')会导致索引失效

```
explain select * from employees where name like '%Lu';
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1621657383586-e46adf88-ba8e-4517-ae6b-78a9a425a9c4.png)

```
explain select * from employees where name like '%Lu%';
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1621657464590-55e7479f-7ca5-4eb4-a6e6-5532578c2dca.png)

前后都是通配符时，索引还是失效，需要使用覆盖索引，即select字段是索引相关字段。

```
explain select name,age from employees where name like '%Lu%';
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1621657580063-fc78eea7-4347-4455-a19d-a52d7f3d757b.png)

### 10.查询使用or或者in时

不一定会走索引，mysql内部优化器会根据检索比例、表大小等多个因素整体评 

估是否使用索引



### 11.查询使用范围查询时会失效

```
explain select * from employees where age >1 and age <=1000;
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1621657854665-107616ef-b4c1-4c2f-b5e5-803e671e1486.png)

这里可能是数据量过大或者过小的情况下，直接不走索引，如果数据量太大可以优化范围查询（即分页）

如：

```
explain select * from employees where age >1 and age <=500;
```

### 12.组合索引常见示例

组合索引index_a_b_c(a,b,c)

| where子句                                 | 索引是否有效                               |
| ----------------------------------------- | ------------------------------------------ |
| a=3                                       | 是，使用了最左字段a                        |
| a=3 and b=4                               | 是，使用了a、b                             |
| a=3 and b=4 and c=5                       | 是，都使用了                               |
| b=4 and c=5或者b=4或者c=5                 | 否，没使用最左字段a                        |
| a=3 and c=5                               | a有用，c没用，b被中断了                    |
| a=3 and b>4 and c=5                       | a、b有用，c不在范围内，b是范围查找，中断了 |
| a=3 and b like 'AA%' and c=5              | 是，都使用了                               |
| a=3 and b like '%AA' and c=5              | 是，使用了a                                |
| a=3 and b like '%AA%' and c=5             | 是，使用了a                                |
| a=3 and b like 'A%AA%' and c=5            | 是，都使用了                               |
| like AA%相当于=常量，%AA和%AA% 相当于范围 |                                            |



# 执行一条sql的过程

## 概念图

## 执行过程