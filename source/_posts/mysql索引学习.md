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

explain的extended 扩展能够在原本explain的基础上额外的提供一些查询优化的信息，这些信息可以通过mysql的**show warnings**命令得到。(5.7之后的版本默认就有这个字段，不需要使用explain extended了。这个字段表示存储引擎返回的数据在server层过滤后，剩下多少满足查询的记录数量的比例，注意是**百分比**，不是具体记录数)

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

2） 如果有where 条件，比如where xx=1 order by 索引字段 asc . 这样order by 也会用到索引！**一般情况order by没有按照索引顺序排序**

6.Select tables optimized away:使用某些聚合函数(比如 max、min)来访问存在索引的某个字段。table列为null



## 索引使用及注意事项

### 1.全值匹配

```
explain select * from employees where name = 'Lucy';
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1621608020058-7c7e6bfe-0a9f-4da5-80de-a42ce2db9a19.png)

ref为const

### 2.最左前缀法则

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

（只访问索引的查询（索引列包含查询列）），减少 select * 语句。select的字段是索引所相关的字段，不然无法是覆盖索引，只是const级别的查询而已。

### [ ](https://blog.csdn.net/Mypromise_TFS/article/details/68945293)6.mysql在使用不等于（！=或者<>），not in ，not exists 的时候无法使用索引会导致全表扫描 

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

不一定会走索引，mysql内部优化器会根据检索比例、表大小等多个因素整体评估是否使用索引



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

## 内部组件结构

大体来说，MySQL可以分为 Server 层和存储引擎层两部分。

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1622037742709-593e1d4a-d5fe-4a88-ae6e-4bb807747203.png)

### server层

主要包括连接器、查询缓存、分析器、优化器、执行器等，涵盖 MySQL 的大多数核心服务功能，以及所有的内置函数 （如日期、时间、数学和加密函数等），所有跨存储引擎的功能都在这一层实现，比如存储过程、触发器、视图等。 

### 存储引擎层

存储引擎层负责数据的存储和提取。其架构模式是插件式的，支持 InnoDB、MyISAM、Memory 等多个存储引擎。从MySQL 5.5.5 版本开始默认存储引擎是InnoDB，即建表时没有设置引擎时，默认设置为InnoDB。



## 连接器

1.首先**第一步**，当需要使用Mysql的时候，肯定要与MySQL建立连接才行，现在基本都是用客户端工具进行连接的，比如mysql、navicat、SQLyog等。其实基本都是通过以下连接命令进行连接的：

```
mysql -h host[数据库ip] -u root -p
```

此时客户端面对的就是server层的**连接器。连接器负责跟客户端建立连接、获取权限、维持和管理连接的作用。**

指令中的mysql是客户端用来跟服务端建立连接的命令，在完成经典的TCP握手后，连接器就会开始认证当前用户的身份，此时需要输入账号和密码来完成连接。

- 账号或者密码错误，报错 “Access denied for user”，连接断开
- 认证通过，连接器将会到**系统的权限表（user表）**里查找当前帐号的权限。后续这个连接里的权限判断逻辑，都依赖于此次获取的权限列表。

含义：当一个账号建立连接成功后，即使当管理员**修改了当前账号的权限**时，当前连接的权限也**不会被影响**到，还在这次会话中。只有**新建立**的连接才会被影响。

权限相关指令：

```
grant all privileges on *.* to 'username'@'%'; //赋权限,%表示所有(host) 
flush privileges //刷新数据库
update user set password=password(”123456″) where user=’root’;//更新用户密码
show grants for root@"%"; //查看当前用户的权限
```

2.当建立连接后，一直没有别的操作，这个连接会被置于空闲状态，可通过指令**“show processlist”**查看用户状态。**当长时间没有与Server端交互时，连接器将自动断开**。

自动断开时间由 **wait_timeout**参数控制，**默认时间为8小时**

**查看和修改指令：**

```
show global variables like "wait_timeout"; 
set global wait_timeout=28800; //全局的，单位秒，默认8小时
```



## 查询缓存

**第二步**，当连接建立后，即可开始进行查询操作了。

MySQL在接到select请求之后，会**先去查询缓存**看看之前是不是执行过这条语句。之前执行过语句及其结果会**以key-value存放在内存中**。key是查询的语句，value是查询的结果。如果你的查询在查询缓存中找到key，则直接返回查询缓存的value给客户端。

如果查询语句不在查询缓存中，就会继续后面的执行流程。执行完成后，执行结果会被存入查询缓存中。

**注意：从MySQL8.0开始，直接把整个查询缓存的功能删除掉。**



MySQL实战中提到基本**不建议使用查询缓存，为什么**？

**因为查询缓存往往利大于弊**。查询缓存**非常容易失效且频繁**，只要对一张表进行更新操作，这个表的所有查询缓存都会被清空。因此之前建立的查询缓存还没有使用就要被清空，这对于更新压力大的数据库，查询缓存的命中率更低。



什么时候可以可以使用或者**建议使用查询缓存**？

系统中的静态表，可以使用。如：字典表、系统配置表，这类基本不会怎么变化的表。



查询缓存的配置：在my.cnf中进行修改即可。

```
show global variables like "%query_cache_type%";//查看是否开启缓存
show status like'%Qcache%'; //查看运行的缓存信息

#query_cache_type有3个值 
0->关闭查询缓存OFF
1->开启ON
2->（DEMAND）当sql语句中有SQL_CACHE 关键词时才缓存
//如
select SQL_CACHE from user where name='root';
```



## 分析器

**第三步**，当select查询没有命中缓存时，这时候就要真正的开始执行select语句了。

首先，MySQL需要知道select语句要操作什么，即解析select语句。

1. 通过select关键字判断是查询语句
2. 词法分析，识别各个字符串代表的是什么，如user代表表名、id代表列名等

1. 语法分析，根据其规则，判断整个sql的语法是否正确；错误则会报错



**原理**:

分析器分成6个主要步骤完成对sql语句的分析 

1、词法分析 

2、语法分析 

3、语义分析 

4、构造执行树 

5、生成执行计划 

6、计划的执行



例如：

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1622041602935-708c4036-cf78-4c8b-a09a-72fdf2c7e8f9.png)

语法树文章：https://en.wikipedia.org/wiki/LR_parser。



## 优化器

**第四步**，当分析器分析的包括语法和语义都正确后，MySQL基本就了解了select语句需要做的操作。此时MySQL会经过优化器处理。

优化器是在表里面**有多个索引**的时候，决定使用哪个索引；或者在一个语句有**多表关联**（join）的时候，决定各个表的连接顺序。



优化器的作用就是决定选择使用哪一个更加高效的方案(MySQL数据库使用的是基于成本的优化器，估算成本为CPU代价+IO代价)。当优化器阶段完成后，这个语句的执行方案就确定下来了，然后进入执行器阶段。



## 执行器

**第五步**，优化器选定了执行方案后，进入到执行器来执行sql。

当执行sql的时候，会先进行权限校验(**即连接器时拿到的权限列表**)，如果没有权限则会报错提示。如果权限认证成功，在打开表的时候，执行器会拿到表的引擎设置，用对应引擎提供的接口执行。



```
select * from user where id=7;
```

执行过程为：

1. 调用InnoDB引擎接口取表T的第一行，判断id值是否等于7，如果不是则跳过，如果是则存到结果集中。
2. 遍历整个表，调用引擎接口取下一行，重复以上的相同逻辑，直到取到表T的最后一行。

1. 执行器将上面的遍历过程中所有满足id=7的行组成结果集全部返回给客户端。

具体可通过数据库的慢查询日志中查看 rows_examined 字段(表示这个语句执行过程中扫描了多少行)。

## redo-log

redo log 是 InnoDB 中用来实现**崩溃恢复**的日志。

InnoDB 会将更新操作的内容保存到内存缓冲区中，在合适的时机或不得已的情况下再把缓冲区的数据更新到磁盘中，这样可以减少磁盘的随机读写，提高更新操作的性能。但是一旦程序崩溃，缓冲区中的数据就会丢失，为了实现崩溃恢复，InnoDB引入 redo log，执行更新语句的时候不但需要把更新的数据保存到内存缓冲区，还需要把更新的数据写到redo log，这样在程序崩溃之后，能够根据redo log进行数据恢复，redo log是顺序写入的，因此能够保证更新操作的性能。这就是Mysql中经常说到的WAL技术（Write-Ahead Logging）。

redo log 又被称为 物理日志，记录的是“在某个数据页上做了什么修改”。

redo log 是固定大小的，采用循环写的方式；但是循环写是有条件的，当redo log写满以后，需要把相应的脏页写到磁盘（刷脏页），才能够对redo log进行“循环写”。



## bin-log

binlog 是 Mysql Server 用来实现**备份、复制、数据**恢复等功能的二进制日志。



binlog是Server层实现的二进制日志,他会记录cud操作。Binlog有以下几个特点： 

1、Binlog在MySQL的**Server层实现**（引擎共用） 

2、Binlog为逻辑日志,记录的是一条语句的原始逻辑 (数据更新或潜在发生更新的SQL语句)

3、Binlog不限大小,追加写入,不会覆盖以前的日志



binlog 有以下三种格式：

- Statement： 每一条修改数据的 sql 都会记录在日志中。
- Row：不记录sql语句上下文相关信息，就保存哪条记录被修改。

- Mixed: 以上两种格式的混合使用，一般的语句修改使用statment格式保存；如一些函数，statement无法完成主从复制的操作，则采用row格式保存。



**需要在my.cnf中配置**

```
 mysql> show variables like '%log_bin%'; 查看bin‐log是否开启 
 mysql> flush logs; 会多一个最新的bin‐log日志 
 mysql> show master status; 查看最后一个bin‐log日志的相关信息 
 mysql> reset master; 清空所有的bin‐log日志
```

redo-log 和 bin-log 参考：[https://blog.zhenlanghuo.top/2020/02/06/Mysql%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E2%80%94%E2%80%94%E9%87%8D%E5%81%9A%E6%97%A5%E5%BF%97%E4%B8%8E%E5%BD%92%E6%A1%A3%E6%97%A5%E5%BF%97/](https://blog.zhenlanghuo.top/2020/02/06/Mysql学习笔记——重做日志与归档日志/)





# 索引实战

## 常见具体索引案例

```
#例子：联合索引
KEY `idx_name_age_position` (`name`,`age`,`position`) USING BTREE ;
```

### 第一个字段使用范围查询，索引失效

```
explain select * from employees where name > 'LiLei' and age = 22 
and position = 'manager';
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1622470901483-b9a67bdf-6b18-425b-abe8-9865eb54f1e9.png)

#### 1.强制使用索引

```
explain select * from employees force index(idx_name_age_position) 
where name > 'LiLei' and age = 22 and position = 'manager';
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1622470959865-5a024840-3106-4459-80c2-41b33ae2c9f7.png)

**key_len 为74**，说明第一个字段生效了。

总结：mysql默认是**不走索引**的，当**强行使用**索引时，虽然其rows少了许多，但**不一定效率就会更好**，实际开发中需要对比使用。



#### 2.使用覆盖索引优化

```
explain select `name`,`age`,`position` from employees 
where name > 'LiLei' and age = 22 and position = 'manager';
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1622471392378-c9a2558e-f8f0-457b-9c61-ae081709b74f.png)

### in和or在表数据量大的情况下会走索引，表记录不多时会全表扫描

```
EXPLAIN SELECT * FROM employees WHERE name in ('LiLei','HanMeimei') 
AND age = 22 AND position ='manager';
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1622471569748-fc1fa7c6-06eb-444d-a19b-6e96a124e802.png)

### like 'LL%' 一般都走索引，与表数据大小无关

```
EXPLAIN SELECT * FROM employees WHERE name like  'LiLei%'  
AND age = 22 AND position ='manager';
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1622471715695-548fcbb3-d90d-427f-8fa8-981c06df8a18.png)



like 'LL%' 涉及概念，**索引下推。**

**问题：一般来说，此例子中， 只会有name字段的索引，后两个没用到，因为根据name字段，在索引树上得到的数据中(索引行)，age和position是无序的，没办法更好的使用索引。**

#### 索引下推

索引条件下推优化（Index Condition Pushdown (ICP) ）是MySQL5.6添加的，用于优化数据查询。  

- 不使用索引条件下推优化时，存储引擎通过索引(name like 'LiLei')检索到数据，然后**拿到对应叶子节点上的id簇**，返回给MySQL服务器进行回表查询，服务器然后**根据剩下字段条件**判断符合的数据，返回结果集。 
- 当使用索引条件下推优化时，如果存在某些被索引的列的判断条件时，MySQL服务器将这一部分判断条件传递给存储引擎，然后由存储引擎通过判断索引是否符合MySQL服务器传递的条件，只有当索引符合条件时才会将数据检索出来返回给MySQL服务器。(**即先用索引的字段条件判断，过滤不符合的记录后，再拿到id回表查询，有效的减少回表的次数**)



对于InnDB引擎只适用于二级索引，因为InnDB的聚簇索引会将整行数据读到InnDB的缓冲区(**叶子结点保存了全行数据**)，这样一来索引条件下推的主要目的(减少IO次数)就失去了意义。因为数据已经在内存中了，不再需要去读取了，并不会起到减少查询全行数据的效果。

```
SET optimizer_switch = 'index_condition_pushdown=off'; #默认开启
SET optimizer_switch = 'index_condition_pushdown=on';
```

参考：https://juejin.cn/post/6844904017332535304



### Mysql引擎是如何选择合适的索引的

mysql提供了一个trace工具进行分析。

感觉大概就是sql执行的过程相对应起来看：

- 1分析器进行sql的词义分析、语法分析等
- 2优化器对sql进行相应的优化，mysql估算出表里涉及的各种方案所需要花费的代价--**cost，定下所需要的方案**

- 3执行器执行第二步确认的方案

参考：https://iluoy.com/articles/203



### 常见对于sql的优化场景



MySQL关于索引的一些基础知识要点：

• a、EXPLAIN结果中的key_len只显示了条件检索子句需要的索引长度，但 ORDER BY、GROUP BY 子句用到的索引则不计入 key_len 统计值； 

• b、联合索引(composite index)：多个字段组成的索引，称为联合索引； 例如：ALTER TABLE t ADD INDEX `idx` (col1, col2, col3) 

• c、覆盖索引(covering index)：如果查询需要读取到索引中的一个或多个字段，则可以**从索引树**中直接取得结果集，称为覆盖索引； 例如：SELECT col1, col2 FROM t; 

• d、最左原则(prefix index)：如果查询条件检索时，只需要匹配联合索引中的最左顺序一个或多个字段，称为最左索引原则，或者叫最左前缀； 例如：SELECT * FROM t WHERE col1 = ? AND col2 = ?; 

• e、在老版本（大概是5.5以前，具体版本号未确认核实）中，查询使用联合索引时，可以不区分条件中的字段顺序，在这以前是需要按照联合索引的创建顺序书写SQL条件子句的； 例如：SELECT * FROM t WHERE col3 = ? AND col1 = ? AND col2 = ?; 

• f、MySQL截止目前还只支持多个字段都是正序索引，不支个别字段持倒序索引； 例如：ALTER TABLE t ADD INDEX `idx` (col1, col2, col3 DESC)，这里的DESC只是个预留的关键字，目前还不能真正有作用 

• g、联合索引中，如果查询条件中最左边某个索引列使用范围查找，则只能使用前缀索引，无法使用到整个索引； 例如：SELECT * FROM t WHERE col1 = ? AND col2 >= ? AND col3 = ?; 这时候，只能用到 idx 索引的最左2列进行检索，而col3条件则无法利用索引进行检索 

• h、InnoDB引擎中，二级索引实际上包含了主键索引值；



#### order by和group by优化

相关案例参考：https://www.kancloud.cn/thinkphp/mysql-faq/47431



总结：

1.MySQL支持两种排序方式：filesort和index(**extra列查看**)。

index->获取的数据是：联合索引已经排好序的数据(索引树中)

filesort->文件，磁盘排序的数据(效率低下)

2.覆盖索引优先使用

3.使用索引列排序时，遵循最左原则

4.order by字段非索引字段时，一定会filesort

5.where和order by一起使用时，遵循最左原则



#### filesort 排序算法

定义：当MySQL不能使用索引进行排序时，就会利用自己的排序算法(快速排序算法)在内存(sort buffer)中对数据进行排序，如果内存装载不下，它会将磁盘上的数据进行**分块**(**产生临时文件**)，再对各个数据块进行排序，然后将各个块合并成有序的结果集（实际上就是外排序）。

对于filesort，MySQL有两种排序算法：

1、**两遍扫描算法(Two passes、双路排序、回表排序)**
实现方式是**先将需要排序的字段和可以直接定位**到相关行数据的**指针信息(数据行ID)**取出，然后在设定的内存（**sort_buffer**通过参数 sort_buffer_size 设定）中进行排序，**完成排序之后再次通过行指针信息取出所需的列**。
注：该算法是4.1之前只有这种算法，它需要两次访问数据，尤其是第二次读取操作会导致大量的随机I/O操作。不过，这种方法内存开销较小。

trace中：sort_mode信息里显示< sort_key, rowid >

2、**一次扫描算法(single pass、单路排序)**
该算法**一次性**将所需的列全部取出，在内存中排序后直接将结果输出。

trace中：sort_mode信息里显示< sort_key, additional_fields >或者< sort_key, packed_additional_fields >



- 如果字段的总长度**小于**max_length_for_sort_data ，那么使用 单路排序模式
- 如果字段的总长度**大于**max_length_for_sort_data ，那么使用 双路排序模式



#### 分页查询优化

参考：https://www.cnblogs.com/youyoui/p/7851007.html

**语法**：分页查询使用简单的 limit 子句就可以实现。limit 子句声明如下：

```
SELECT * FROM table LIMIT [offset,] rows | rows OFFSET offset
```

- offset 指定第一个返回记录行的偏移量，注意从0开始，默认为0
- rows   指定返回记录行的最大数目

- 如果只给定一个参数：如limit 10 它表示返回rows



例：(**主键递增的情形，id没有断档**)

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1622796227163-32b20cb0-ce26-43a6-b8e7-8d8947f7c563.png)

```
select * from user limit 100,10;
```

表示**offset**从100开始，取的是101-110的10条数据。

一般默认主键id排序，**等效于 101<=id<=110**

```
select * from user order by id limit 100,10;
```

看似只查询了 10 条记录，实际这条 SQL 是先读取 110 条记录，然后抛弃前 100 条记录，再读到后面 10 条想要的数据。



**现象**：

**验证偏移量变化带来的时间影响**：

```
select * from user order by id limit 10,100;
select * from user order by id limit 100,100;
select * from user order by id limit 1000,100;
select * from user order by id limit 10000,100;
select * from user order by id limit 100000,100;
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1622796510087-c602849a-b50c-41e6-b1cd-75f58d43b067.png)

结论：在偏移量大于10W数据后，时间花费就会比较久。



**验证记录变化带来的时间影响**：

```
select * from user order by id limit 100,10;
select * from user order by id limit 100,100;
select * from user order by id limit 100,1000;
select * from user order by id limit 100,10000;
select * from user order by id limit 100,100000;
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1622796748754-4402100f-bbaa-44b9-a422-20d908a4bdba.png)

结论：当偏移量在差不多时，记录小时基本花费时间差不多。随着记录多而时间久。



**优化**：

1.使用id限定（假设数据表的id是连续递增的）

执行时间对比：

```
select * from user limit 900000,100;
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1622797256474-cdab7aaa-3edb-473e-a8cd-d29711ab8f3a.png)

```
select * from user where id>900000 limit 100;
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1622797468239-bd46cf64-980d-408e-979b-e3b11696a88c.png)

执行计划对比：

```
explain select * from user limit 900000,100;//执行计划
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1622797344734-eee97499-da24-4aab-9f44-a3f0c48c0a65.png)

```
explain select * from user where id>900000 limit 100;//执行计划
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1622797553308-b008e0f5-1c62-4e6f-9d45-435c45bfe91e.png)

结论：使用id远快于直接limit，但是会存在问题，如果id是不连续的时，查询的记录肯定不同。

因此有两个必要条件：

- id为自增的
- 结果按照id排序

注：**当然，这里也有根据非主键字段排序，比如添加了索引的字段，但是不一定会走索引，mysql引擎会判断执行的成本，选择具体的执行方案。**



2.使用子查询或者join

```
select * from user limit 900000,100;
select id from user limit 900000,100;
select * from user where id>=(select id from user limit 900000,1) limit 100;
select * from user a inner join (select id from user limit 900000,100) b
on a.id=b.id ;
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1622803330922-5b54f68c-c9ab-4c7b-8f34-f5b85b72dde9.png)





#### Join查询优化

驱动表概念：

- inner join连接时，小表做驱动表，并不一定是排在前面的表就是驱动表。
- left join连接时，左表是驱动表，右表是被驱动表。

- right join连接时，右表是驱动表，左表是被驱动表。



**mysql的表关联常见有两种算法：** 

- Nested-Loop Join 算法 
- Block Nested-Loop Join 算法

```
--表结构
CREATE TABLE IF NOT EXISTS `user1`(
   `id` INT UNSIGNED AUTO_INCREMENT,
   `num` int(11),
	 `num2` int(11),
   PRIMARY KEY ( `id` ),
	 KEY inx_nnum(`num`)
)ENGINE=InnoDB DEFAULT CHARSET=utf8;
--user2和user1表结构一致，user1插入1w数据，user2插入1百数据
```

Nested-Loop Join 算法概念：

顾名思义，是指**嵌套循环算法**，即将驱动表/外部表的结果集作为**循环基础数据**，然后循环从该结果集**每次获取一条**数据作为**下一个表的过滤条件**查询数据，然后**合并结果**。如果有多表join，则将**前面的表的结果集**作为循环数据，取到每行再到联接的下一个表中**循环匹配**，获取结果集返回给客户端。

举例：



```
explain select * from user1 a inner join user2 b on a.num = b.num;//num有索引
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1622819539531-bffc3f40-b0b8-4cd9-b386-aaee8a64cd18.png)

大致取数逻辑：

1. **此时驱动表是b表，被驱动表是a表**。会先从b中读取一行数据，如果有条件会先过滤条件，再拿出一行数据
2. 从上一步的数据中，拿出关联字段num，去a表中查找对应数据

1. 取出a表中符合条件的数据行，再和b表获取的数据结果合并，放到结果集中
2. 重复前面三个步骤，循环结束，将结果集返回给客户端

简化就是：b表100次扫描，每一次b的关联条件num去a中找数据，又在a中相对应的扫描了100次(可以认为最中只扫描了一行完整数据)，即整个过程总共扫描了200行(**磁盘扫描**)。



如果有多层join时，最前面的join结果集会当作下一层的驱动表来查询。

就像代码中的循环：

```
for(;;) //外层循环 
{ 
	for(;;) //内层循环 
	{ ... }
}
```

Block Nested Loop Join (BNLJ)算法:

BNLJ，块嵌套循环。BNLJ是优化版的NLJ，BNLJ区别于NLJ的地方是**它加入了缓冲区join buffer**，它的作用是外层驱动表可以**先将一部分数据**事先存到join buffer中，然后**再和内层的被驱动表进行条件匹配**，匹配成功的记录将会连接后存入结果集，等待全部循环结束后，将结果集发给client即完成一次join。



例子：

```
explain select * from user1 a inner join user2 b on a.num2 = b.num2;//num2无索引
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1622820370600-d74e59cf-1b32-4b33-9af5-3b91e92b955d.png)

大致取数逻辑：

1. **此时驱动表是b表，被驱动表是a表**。将b的所有数据都放到缓冲区中
2. 再将a表中每一行数据都取出来，与缓冲区中的数据做对比

1. 把满足join关联条件的数据返回给客户端

简化就是：a、b两个表都进行了一次全表扫描，扫描总数即为a+b的数据总量=10100。但是缓冲区中的数据是没有顺序的，因此对于a表(被驱动表)中的每一行数据，都要与之b表(驱动表)做b表数据总量100次的判断，所以**内存中**需要判断1w*100=1百万次。



MySQL使用Join Buffer有以下**要点**:

1.  join_buffer_size变量决定buffer大小。 
2.  只有在join类型为all, index, range的时候才可以使用join buffer。 

1.  能够被buffer的每一个join都会分配一个buffer, 也就是说一个query最终可能会使用多个join buffer。 
2.  第一个nonconst table不会分配join buffer, 即便其扫描类型是all或者index。 

1.  在join之前就会分配join buffer, 在query执行完毕即释放。 
2.  join buffer中只会保存参与join的列, 并非整个数据行。 

**用BNL磁盘扫描次数少很多，相比于磁盘扫描，BNL的内存计算会快得多。**



#### in和exsits优化

概念：

- 当A表中数据多于B表中的数据时，这时我们使用IN优于EXISTS
- 当B表中数据多于A表中的数据时，这时我们使用EXISTS优于IN

- 如果两张表中的数据差不多时那么是使用IN还是使用EXISTS差别不大。
- EXISTS子查询只返回true或者false，因此子查询中的select * from 可以是select 1或者其他

```
--A数据>B数据
select * from A where id in (select id from B);
--相当于
for(select id from B){ 
			select * from A where A.id = B.id 
}
--A数据<B数据
select * from A where exists (select 1 from B where B.id = A.id) 
--相当于
for(select * from A){ 
			select * from B where B.id = A.id 
}
```

#### count查询优化

字段有索引：count(*)≈count(1)>count(字段)>count(主键 id) //字段有索引，count(字段)统计走二级索引，二 

级索引存储数据比主键索引少，所以count(字段)>count(主键 id) 

字段无索引：count(*)≈count(1)>count(主键 id)>count(字段) //字段没有索引count(字段)统计走不了索引， 

count(主键 id)还可以走主键索引，所以count(主键 id)>count(字段) 



**count常见优化方法**

- 建表选择好对应的引擎，如myisam引擎会将count总行数，记录在磁盘中，查询不需要计算总行数。而innodb的机制不会，需要计算。
- show table status查看

- 维护总行数数据加入缓存中
- 维护计数的表，来专门处理



### 索引的设计原则

#### 选择唯一性索引

唯一性索引的值是唯一的，可以更快速的通过该索引来确定某条记录。例如，学生表中学号是具有唯一性的字段。为该字段建立唯一性索引可以很快的确定某个学生的信息。如果使用姓名的话，可能存在同名现象，从而降低查询速度。 

#### 经常需要排序、分组和联合操作的字段

经常需要 ORDER BY、GROUP BY、DISTINCT 和 UNION 等操作的字段，排序操作会浪费很多时间。如果为其建立索引，可以有效地避免排序操作。 

#### 经常需要当作查询条件的字段

如果某个字段经常用来做查询条件，那么该字段的查询速度会影响整个表的查询速度。因此，为这样的字段建立索引，可以提高整个表的查询速度。多个条件经常使用，建立联合索引。
注意：常查询条件的字段不一定是所要选择的列，换句话说，**最适合索引的列是出现在 WHERE 子句中的列**，或连接子句中指定的列，而不是出现在 SELECT 关键字后的选择列表中的列。 

#### 限制索引的数目

索引的数目不是“越多越好”。每个索引都需要占用磁盘空间，索引越多，需要的磁盘空间就越大。在修改表的内容时，索引必须进行更新，有时还可能需要重构。因此，索引越多，更新表的时间就越长。 

如果有一个索引很少利用或从不使用，那么会不必要地减缓表的修改速度。此外，MySQL 在生成一个执行计划时，要考虑各个索引，这也要花费时间。创建多余的索引给查询优化带来了更多的工作。索引太多，也可能会使 MySQL 选择不到所要使用的最佳索引。

#### 尽量使用数据量少的索引

如果索引的值很长，那么查询的速度会受到影响。例如，对一个 CHAR(100) 类型的字段进行全文检索需要的时间肯定要比对 CHAR(10) 类型的字段需要的时间要多。**name varchar(255) ->index(name(30))**

#### 数据量小的表最好不要使用索引

由于数据较小，查询花费的时间可能比遍历索引的时间还要短，索引可能不会产生优化效果。

#### 尽量使用前缀来索引

如果索引字段的值很长，最好使用值的前缀来索引。例如，TEXT 和 BLOG 类型的字段，进行全文检索会很浪费时间。如果只检索字段的前面的若干个字符，这样可以提高检索速度。

#### 删除不再使用或者很少使用的索引

表中的数据被大量更新，或者数据的使用方式被改变后，原有的一些索引可能不再需要。应该定期找出这些索引，将它们删除，从而减少索引对更新操作的影响。





# 事务与锁机制

## 事务基本

### 概念

数据库事务: 数据库事务通常指对数据库**进行读或写的一个操作序列**。

它的存在包含有以下两个目的：

 1、为数据库操作提供了一个从失败中**恢复**到正常状态的方法，同时提供了数据库即使在异常状态下仍能保持一致性的方法。

   2、当多个应用程序在**并发访问数据库**时，可以在这些应用程序之间**提供一个隔离方法**，以防止彼此的操作互相干扰。



### 常用指令

开启事务： begin、start transaction； 

提交事务：commit 中止事务：rollback 

自动提交：show variables like 'autocommit';默认开启，set autocommit = OFF/ON 

隐式提交：DDL（数据库定义语言）如：create、alter、drop 

保存点：savepoint 保存点名称；回滚到某个点： rollback to 保存点名称； 删除保存点：release savepoint xxx

[






](https://blog.csdn.net/z646721826/article/details/79412459)

### 特性

#### ACID属性

事务是由一组SQL语句组成的逻辑处理单元,事务具有以下4个属性,通常简称为事务的ACID属性。 

- 原子性(Atomicity) ：事务是一个**原子操作单元**,其对数据的修改,要么全都执行,要么全都不执行。 
- 一致性(Consistent) ：在事务开始和完成时,数据都必须保持一致状态。这意味着所有相关的数据规则都必须应用于事务的修改,以保持数据的完整性。 

- 隔离性(Isolation) ：数据库系统提供一定的**隔离机制**,保证事务在不受外部并发操作影响的“独立”环境执行。这意味着事务处理过程中的中间状态**对外部是不可见**的,反之亦然。 
- 持久性(Durable) ：事务完成之后,它对于数据的修改是永久性的,即使出现系统故障也能够保持。



#### 并发事物影响

**脏读（Dirty Reads）** 

事务A读取到了**事务B已经修改但尚未提交**的数据，被称之为“**脏数据**”，还在这个数据基础上做了操作。此时，如果B事务回滚，A读取的数据无效，**不符合一致性要求**。 （重点在于读取未提交的数据）

**不可重读（Non-Repeatable Reads）** 

事务A内部的相同查询语句在**不同时刻读出的结果不一致**，可能此时被别的事物进行了操作，这种现象就叫做“不可重复读”。**不符合隔离性** 。（重点在于多次查询不一致）

**幻读（Phantom Reads）** 

事务A读取到了事务B提交的**新增数据**，**不符合隔离性**。（重点在于新增或删除）

**更新丢失(Lost Update)或脏写** 

当两个或多个事务选择同一行，然后基于最初选定的值更新该行时，由于每个事务都不知道其他事务的存 

在，就会发生**丢失更新**问题–**最后的更新覆盖**了由其他事务所做的更新。 



### 隔离级别

#### 分类

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1622902060822-1a121b81-9129-4e3c-8345-947557975df8.png)

**READ_UNCOMMITTED：**能够读取到没有被提交的数据。

**READ_COMMITED：**能够读到那些已经提交的数据，自然能够防止脏读。

**REPEATABLE_READ**：一个事务第一次读过某条记录后，即使其他事务修改了该记录的值并且提交，该事务之后再读该条记录时，读到的仍是第一次读到的值，而不是每次都读到不同的数据，这就是可重复读。（**mysql5.7默认**）

**SERLALIZABLE**：以上3种隔离级别都允许对同一条记录同时进行读-读、读-写、写-读的并发操作，如果我们不允许读-写、写-读的并发操作，可以使用SERIALIZABLE隔离级别，**因为其会对每一条执行sql都进行上锁**(包括select，也只包括当前执行的条件，如select * from tab where id=1，但别的事务id=2是可以操作的)，于此操作都是串行的。（在此级别下，都是通过加锁互斥来实现的）



#### 常用指令

当前数据库的事务隔离级别: show variables like 'tx_isolation'; 

设置事务隔离级别：set tx_isolation='REPEATABLE-READ'; 



## 事务与锁

### 常见分类

| 类别                         | 锁                         |
| ---------------------------- | -------------------------- |
| 性能上                       | 乐观锁(Optimistic Locking) |
| 悲观锁(Pessimistic Lock)     |                            |
| 数据库操作上                 | 读锁(共享锁，S锁，shared)  |
| 写锁(排它锁，X锁，eXclusive) |                            |
| 数据操作上                   | 表锁                       |
| 行锁                         |                            |

### 读操作

对于普通 SELECT 语句，InnoDB 不会加任何锁 

**select ... lock in share mode** 将查找到的数据加上一个S锁，允许其他事务继续获取这些记录的S锁，不能获取这些记录的X锁（会阻塞） 

**select ... for update** 将查找到的数据加上一个X锁，不允许其他事务获取这些记录的S锁和X锁。



### 写操作

- DELETE：删除一条数据时，先对记录加X锁，再执行删除操作。 
- INSERT：插入一条记录时，会先加隐式锁，隐式锁来保护这条新插入的记录在本事务提交前不被别的事务访问到。 

- UPDATE：

- 如果被更新的列，修改前后没有导致存储空间变化，那么会先给记录加X锁，再直接对记录进行修改。 
- 如果被更新的列，修改前后导致存储空间发生了变化，那么会先给记录加X锁，然后将记录删掉，再Insert一条新记录。

```
隐式锁：一个事务插入一条记录后，还未提交，这条记录会保存本次事务id，而其他事务如果想来读取这个
记录会发现事务id不对应，所以相当于在插入一条记录时，隐式的给这条记录加了一把隐式锁。
```



### 行锁和表锁及衍生锁

#### 行锁

顾名思义，行锁就是一锁**锁一行或者多行记录**，mysql的行锁是**基于索引加载**的，所以行锁是要加在索引响应的行上，即命中索引。**当索引命中了多条记录时，会锁住多条记录**。

当其他事务访问数据库同一张表时，**被锁定的记录不能被访问，其他的记录都可以访问到**。(即一个session开启事务更新不提交，另一个session更新同一条记录会阻塞，更新不同记录不会阻塞)



行锁的特征：锁冲突概率低，并发性高，但每次操作行都会加锁，开销大，加锁慢；且会有死锁的情况出现。



#### 表锁

顾名思义，表锁就是一锁**锁一整张表**，在表被锁定期间，其他事务不能对该表进行操作，必须等当前表的锁被释放后才能进行操作。表锁响应的**是非索引字段**，即全表扫描。



表锁的特征：锁冲突概率最高，不会出现死锁的情况，开销小，加锁快，但并发度最低。



加锁：LOCK TABLE `table` [READ|WRITE];

解锁：UNLOCK TABLES;

查看加锁的表：show open tables；



#### 间隙锁(Gap Lock) 

顾名思义，间隙锁又称为区间锁，就是一锁就**锁的是一个区间，两个值的空隙**。其隶属于行锁，即触发条件必然是命中索引，所以当查询条件是范围查询时，假设没有查询到有效存在的记录，或者范围中间-表确实缺少的记录，表都会将其进行锁定。且其只会在**REPEATABLE_READ时才生效。**

如存在以下数据：

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1622908147291-55b5c0bf-13b8-4aee-bf13-418a6c140b97.png)

此时间隙区间可以划分为：(2,5),(5,10),(10,正无穷) 三个区间。

测试1:

```
update stock set money='66' where id > 2 and id < 5;
```

那么锁定的区间为(2,5]，是个左开右闭的区间。不包含2，包含5，不包含大于5的，都无法修改数据。

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1622908247584-971b82b9-babc-46ad-88f3-353e734cc3d9.png)

另一个事务中：

```
update stock set money='66' where id=2;
INSERT INTO `Test`.`stock` (`id`,`name`, `money`) VALUES (4,'zsl4', '340');
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1622908322360-e2e80c1e-09f0-42b7-894e-b6b18691b8b0.png)

测试2:

```
update stock set money='66' where id > 2 and id < 11;
```

(11,无穷)被锁了

```
INSERT INTO `Test`.`stock` (`id`,`name`, `money`) VALUES (12,'zsl4', '340');
```

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1622908801356-a901b80e-3d36-4247-8f24-44cb13ac0a20.png)



#### 临键锁(Next-key Locks)

mysql的行锁默认就是使用的临键锁，**临键锁是由行锁和间隙锁共同实现的**。

间隙锁的触发条件是命中索引，范围查询没有匹配到相关记录。而临键锁恰好相反，临键锁的触发条件也是**查询条件命中索引**，不过，临键锁**有匹配到数据库记录**。
间隙锁所**锁定的区间是一个左开右闭的集合**，而临键锁**锁定是当前记录的区间和下一个记录的区间。**



InnoDB的行锁是针对索引加的锁，不是针对记录加的锁。并且该索引不能失效，否则都会从行锁升级为表锁。 



#### 行锁分析 

通过检查InnoDB_row_lock状态变量来分析系统上的行锁的争夺情况 

```
show status like 'innodb_row_lock%'; 
```

对各个状态量的说明如下： 

Innodb_row_lock_current_waits: 当前正在等待锁定的数量 

Innodb_row_lock_time: 从系统启动到现在锁定总时间长度 

Innodb_row_lock_time_avg: 每次等待所花平均时间 

Innodb_row_lock_time_max：从系统启动到现在等待最长的一次所花时间

Innodb_row_lock_waits:系统启动后到现在总共等待的次数



### InnoDB下的当前读和快照读

- 当前读
  像select lock in share mode(共享锁), select for update ; update, insert ,delete(排他锁)这些操作都是一种当前读，为什么叫当前读？就是**它读取的是记录的最新版本**，读取时还要保证其他并发事务不能修改当前记录，会对读取的记录进行加锁。
- 快照读
  像不加锁的select操作就是快照读，即不加锁的非阻塞读；快照读的前提是隔离级别不是串行级别，串行级别下的快照读会退化成当前读；之所以出现快照读的情况，是基于提高并发性能的考虑，**快照读的实现是基于多版本并发控制**，即MVCC,可以认为MVCC是行锁的一个变种，但它在很多情况下，**避免了加锁操作，降低了开销**；既然是基于多版本，即快照读可能读到的**并不一定是数据的最新版本**，而有可能是之前的历史版本

说白了MVCC就是为了实现读-写冲突不加锁，而这个读指的就是快照读, 而非当前读，当前读实际上是一种加锁的操作，是悲观锁的实现。



### 版本链

每一行默认有两三个字段： **rowid、事务id、指针（指向上一个版本；版本链）** 

对于使用**InnoDB存储引擎**的表来说，它的聚簇索引记录中都包含**两个必要的隐藏列**（row_id并不是必要的，我们 

创建的表中有主键或者非NULL唯一键时都不会包含row_id列）： 

**trx_id**：每次对某条记录进行改动时，都会把对应的事务id赋值给trx_id隐藏列。 

**roll_pointer**：每次对某条记录进行改动时，这个隐藏列会存一个指针，可以通过这个指针找到该记录修改前的信息(**即上一次修改记录的指针**)。

**注：select语句的事务不会生成trx_id。**



### MVCC（Multi-Version Concurrency Control ，多版本并发控制）

问题：在**REPEATABLE_READ**隔离级别下如何保证事务的隔离性，同样的select查询语句在一个事务中执行多次的结果是相同的，就算别的事务修改了当前数据也不会受影响呢？

答：Mysql使用了MVCC机制来保证，但未真正的对记录进行加锁操作。在**REPEATABLE_READ和READ_COMMITED**隔离级别下都实现了MVCC机制。



#### undo日志

Undo log是InnoDB MVCC事务特性的重要组成部分。

当**对记录做了变更操作时就会产生undo记录**，Undo记录默认被记录到系统表空间(ibdata)中，但从5.6开始，也可以使用独立的Undo 表空间。

Undo记录中**存储的是老版本数据**，当一个旧的事务需要读取数据时，为了能读取到老版本的数据，需要顺着undo链(**记录版本链**)找到满足其可见性的记录。当版本链很长时，通常可以认为这是个比较耗时的操作。



- insert undo log
  代表事务在insert新记录时产生的undo log, 只在事务回滚时需要，并且在事务提交后可以被立即丢弃
- update undo log
  事务在进行update或delete时产生的undo log; 不仅在事务回滚时需要，在快照读时也需要；所以不能随便删除，且需要维护多版本信息；只有在快速读或事务回滚不涉及该日志时，对应的日志才会被**purge**线程统一清除。在InnoDB里，UPDATE和DELETE操作产生的Undo日志被归成一类，即update_undo。



其中就维护着每行的两个隐藏字段，**trx_id和roll_pointer，通过这两个字段与undo日志串联起来，形成一个历史记录版本链。(像一条链表：****undo log的链首就是最新的旧记录，链尾就是最早的旧记录****)**

**如图示例：(**7byte，回滚指针，此处随意打的**)**

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1623035754127-2eec9bd3-3d35-4595-a070-0e9420155507.png)

#### purge线程

- 从前面的分析可以看出，为了实现InnoDB的MVCC机制，更新或者删除操作都只是设置一下老记录的deleted_bit，**并不真正将过时的记录删除**。
- 为了节省磁盘空间，InnoDB有专门的purge线程来**清理deleted_bit为true的记录**。为了不影响MVCC的正常工作，**purge线程自己也维护了一个read view**（这个read view相当于系统中最老活跃事务的read view）;如果某个记录的deleted_bit为true，并且DB_TRX_ID相对于purge线程的read view可见，那么这条记录一定是可以被安全清除的。





#### read view视图

**概念**：当某个事务执行select语句的时候 mysql会为**当前事务**生成一个**readview**，它代表对当前事务发生时，当前库中所有事务情况的一个**一致性视图**，该视图在当前事务结束前都不会变化(此情形指**REPEATABLE_READ**级别，而在**READ_COMMITED**下**每次执行select语句都会重新生成视图**)。

**组成**：由执行查询时所有**未提交事务id数组**（数组里最小的id为**min_id**）和**已创建的最大事务id**（**max_id**）组成，事务里的任何sql查询结果需要**从对应版本链**里的**最新数据**开始逐条**跟read-view做比对**从而得到最终的快照结果。

```
注：两类表达
在MySQL中，一个事务开启，这里说的不是传统意义上的开启，
在InnoDB中，begin/start一个事务，并不会立即分配事务id，而是真正执行了操作才会分配事务id。

begin/start transaction 命令并不是一个事务的起点，在执行到它们之后的第一个修改操作InnoDB表的语句， 
事务才真正启动，才会向mysql申请事务id，mysql内部是严格按照事务的启动顺序来分配事务id的。
```

- m_ids：表示在生成ReadView时当前系统中**活跃的读写事务**的事务id列表。
- min_trx_id：表示在生成ReadView时当前系统中活跃的读写事务中**最小的事务id**，也就是m_ids中的最小值。

- max_trx_id：表示生成ReadView时系统中**已创建的最大事务id**。
- creator_trx_id：表示**生成**该ReadView的事务的事务id。



**版本链比对规则**：

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1622991868521-4b98d5e0-9ae0-4542-9580-8cc6118e7d1e.png)

\1. 如果 row 的 trx_id 落在**绿色部分**( trx_id<min_id )，表示这个版本是已提交的事务生成的，这个数据是可见的； 

\2. 如果 row 的 trx_id 落在**橘色部分**( trx_id>max_id )，表示这个版本是由将来启动的事务生成的，是不可见的(若 row 的 trx_id 就是当前自己的事务是可见的）；
\3. 如果 row 的 trx_id 落在**红色部分**(min_id <=trx_id<= max_id)，那就包括两种情况 

a. 若 row 的 trx_id 在视图数组**(活跃，即未commit)**中，表示这个版本是由还没提交的事务生成的，不可见(若 row 的 trx_id 就是当前自己的事务是可见的)； 

b. 若 row 的 trx_id 不在视图数组**(活跃，即未commit)**中，表示这个版本是已经提交了的事务生成的，可见。



#### 案例及整体流程

表行大小，代表往下的执行顺序。

| 事务50                                              | 事务100                                             | 事务200                                             | 查询1                                                 | 查询2                                                 |
| --------------------------------------------------- | --------------------------------------------------- | --------------------------------------------------- | ----------------------------------------------------- | ----------------------------------------------------- |
| begin;                                              | begin;                                              | begin;                                              | begin;                                                | begin;                                                |
| update user set name='a' and age=10 where **id=1**; |                                                     |                                                     |                                                       |                                                       |
|                                                     | update user set name='d' and age=5 where **id=2**;  |                                                     |                                                       |                                                       |
|                                                     |                                                     | update user set name='b' and age=10 where **id=1**; |                                                       |                                                       |
|                                                     |                                                     | commit;                                             |                                                       |                                                       |
|                                                     |                                                     |                                                     | select name from user where **id=1;****返回：name=b** |                                                       |
| 分割线1                                             |                                                     |                                                     |                                                       |                                                       |
| update user set name='c' and age=20 where **id=1**; |                                                     |                                                     |                                                       |                                                       |
|                                                     |                                                     |                                                     | select name from user where **id=1;****返回：name=b** |                                                       |
| 分割线2                                             |                                                     |                                                     |                                                       |                                                       |
| commit;                                             |                                                     |                                                     |                                                       |                                                       |
|                                                     | update user set name='d' and age=20 where **id=1**; |                                                     |                                                       |                                                       |
|                                                     |                                                     |                                                     | select name from user where **id=1;****返回：name=b** | select name from user where **id=1;****返回：name=c** |
| 分割线3                                             |                                                     |                                                     |                                                       |                                                       |
|                                                     | commit;                                             |                                                     |                                                       |                                                       |

总共分为三段操作(看select时间点)：

1.begin到分割线1之间，事务50(id=1)、事务100(id=2)未commit，事务200(id=1)commit(**id=1和id=5的都会生成对应的undo日志**)。

**当查询1执行selelct语句时，会为当前事务下的select生成一个readview视图**，执行id=1的查询语句，会拿到当前记录的两个隐藏字段，**tx_id,roll_pointer**。比对范围为: [50,100]中的**最小即50**，当前创建的**最大事务id即200**.

事务200commit，**去记录版本链中找记录，且拿到记录存下的tx_id**，再根据**readview的比对规则**，发现200并没有在**活跃数组中**，是可见的，返回undo日志这条记录版本链对应的name=b。

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1623053538808-ab14e6b2-7588-4993-badf-7134ccd9dc5b.png)



2.分割线1到分割线2之间

事务50(id=1)进行了修改(**生成对应的undo记录**)，但**未提交事务**，且查询1(id=1)执行select语句，**生成的readview是不变的**，比对范围为: [50,100]中的**最小即50**，当前创建的**最大事务id即200。**

**去记录版本链中找记录**。

2.1第一条是50，根据**readview的比对规则，其在活跃数组中**，不可见。

2.2第二条是200，可见，得到name=b。

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1623053465104-a0acabdd-a63d-47bb-af2f-98b604830aa2.png)



3.分割线2到分割线3之间

事务100(id=1)进行了修改(**生成对应的undo记录**)，但**未提交事务**，但上一次事务50(id=1)进行了修改**提交了事务**。

3.1查询1(id=1)执行select语句，**生成的readview是不变的**，比对范围为: [50,100]中的**最小即50**，当前创建的**最大事务id即200。**

**去记录版本链中找记录**：

3.1.1第一条100，**其在活跃数组中**，不可见。

3.1.2第二条50，**其在活跃数组中**，不可见。

3.1.3第三条200，**其不在活跃数组中**，可见，返回name=b。



3.2查询2(id=1)执行select语句，事务50commit，第一次执行select，需生成对应的**readview**，比对范围为: [100]中的**最小即100**，当前创建的**最大事务id即200。**

**去记录版本链中找记录**：

3.2.1第一条100，**其在活跃数组中**，不可见。

3.2.2第二条50，**其在已提交事务中**，可见，返回name=c。(**此处事务50确实已提交，select获取的是表实际的数据，验证正确**)

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1623053694701-e65e4736-73a5-4d3d-8bc9-b023bc429bd9.png)



总结：MVCC机制的实现就是**通过read-view机制与undo版本链比对机制**，使得不同的事务会根据数据版本链对比规则读取同一条数据在版本链上的不同版本数据。(每次事务中**进行了修改后**，都会记录到记录版本链中)

注：在**REPEATABLE_READ和READ_COMMITED**隔离级别下都实现了MVCC机制。但是**REPEATABLE_READ的readview在一个事务中是不会变的，但READ_COMMITED下，每次select都会重新快照读。**



部分参考：https://www.jianshu.com/p/8845ddca3b23

### InnoDB下的BufferPool缓存机制

顾名思义，缓存即代表了提前存储数据在内存中。如操作系统的缓存池机制，就是为了避免每次进行磁盘IO，加速数据的访问。MySql作为一个存储系统，固然也会有对应的缓存池机制，来**避免多余的IO操作**。

于此，也不可能将所有的数据都存储在缓存池中。因为访问需要快速，所对应的必然是存储容量的不大性。因此只能存储那些“热点”的数据，放到访问“最近”的地方，最大限度的降低磁盘的访问。



#### 预读

**概念**：磁盘读写，**并不是按需读取**，而是**按页读取**，一次至少读一页数据（一般是4K），如果未来要读取的数据就在页中，就能够省去后续的磁盘IO，提高效率。

**原理**：数据访问，通常都遵循“集中读写”的原则，使用一些数据，大概率会使用附近的数据，这就是所谓的“**局部性原理**”，它表明提前加载是有效的，确实能够减少磁盘IO。



**按页读取的作用**：

（1）磁盘访问按页读取能够提高性能，所以缓冲池一般也是按页缓存数据；

（2）预读机制启示了我们，能把一些“可能要访问”的页提前加入缓冲池，避免未来的磁盘IO操作；



**实现**：LRU(Least recently used)算法，像memcache，OS都会用LRU来进行页置换管理。



#### 传统的LRU

**问题：如何进行缓冲页管理？**



把入缓冲池的页放到**LRU的头部**，作为最近访问的元素，从而最晚被淘汰。这里又分两种情况：

（1）**页已经在缓冲池里**，那就只做“移至”LRU头部的动作，而没有页被淘汰；

（2）**页不在缓冲池里**，除了做“放入”LRU头部的动作，还要做“淘汰”LRU尾部页的动作；

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1623074998017-a0d29cb9-b486-4852-93d6-e38921bd1c14.png)

如LRU缓存池(**假定长度为8**)中，缓存了1、3、2...5等页数据

访问页号为6的数据：

1. 页号为6的页，存在于缓存池中，直接访问数据
2. 将页号为6的页，移动至head头部。**没有页数据被淘汰**

访问页号为21的数据：

1. 页号为21的页，不存在于缓存池中，访问库
2. 将页号为21的页，移动至head头部。**淘汰末尾为5的页数据，全往后移动一位。**



**带来的问题**

1.**预读失效**：由于预读(Read-Ahead)，提前把页放入了缓冲池，但最终MySQL并没有从页中读取数据，称为预读失效。

**优化**：

**要优化预读失效，思路**是：

（1）让预读失效的页，停留在缓冲池LRU里的时间尽可能短；

（2）让真正被读取的页，才挪到缓冲池LRU的头部；

以保证，真正被读取的热数据留在缓冲池里的时间尽可能长。

**具体方法**是：

（1）将LRU分为两个部分：

- 新生代(new sublist)
- 老生代(old sublist)

（2）新老生代首尾相连，即：新生代的尾(tail)连接着老生代的头(head)；

（3）新页（例如被预读的页）加入缓冲池时，只加入到老生代头部：

- **如果数据真正被读取**（预读成功），才会加入到新生代的头部
- **如果数据没有被读取**，则会比新生代里的“热数据页”更早被淘汰出缓冲池

案例：

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1623075810848-884b7bbe-55d1-4059-833c-aed2258ec502.png)

说明：LRU缓存池(假定长度为8)，新老生代首尾相连；





2.**缓冲池污染**：

但新老生代改进版LRU仍然解决不了缓冲池污染的问题。

概念：当某一个SQL语句，要批量扫描大量数据时，可能导致把缓冲池的所有页都替换出去，导致大量热数据被换出，MySQL性能急剧下降，这种情况叫**缓冲池污染**。



**怎么解决这类扫描大量数据导致的缓冲池污染问题？**

MySQL缓冲池加入了一个“老生代停留时间窗口”的机制：

（1）假设T=老生代停留时间窗口；

（2）插入老生代头部的页，即使立刻被访问，并不会立刻放入新生代头部；

（3）只有**满足**“被访问”并且“在老生代停留时间”大于T，才会被放入新生代头部；



**参数**：innodb_buffer_pool_size

**介绍**：配置缓冲池的大小，在内存允许的情况下，DBA往往会建议调大这个参数，越多数据和索引放到内存里，数据库的性能会越好。

**参数**：innodb_old_blocks_pct

**介绍**：老生代占整个LRU链长度的比例，默认是37，即整个LRU中新生代与老生代长度比例是63:37。

**参数**：innodb_old_blocks_time

**介绍**：老生代停留时间窗口，单位是毫秒，默认是1000，即同时满足“被访问”与“在老生代停留时间超过1秒”两个条件，才会被插入到新生代头部。



#### 数据在引擎中流转图

![img](https://cdn.nlark.com/yuque/0/2021/png/705191/1623140952750-15f33bce-e45a-4594-ba8c-2a9aa9252e0e.png)