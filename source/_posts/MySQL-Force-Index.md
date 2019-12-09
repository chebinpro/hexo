---
layout: title
title: MySQL Force Index
date: 2019-05-23 17:14:05
tags:
---
以下的文章主要介绍的是MySQL Force Index强制索引,以及其他的强制操作，其优先操作的具体操作步骤如下：我们以MySQL中常用的hint来进行详细的解析。

<!--more-->

强制索引MySQL FORCE INDEX

SELECT * FROM TABLE1 FORCE INDEX (FIELD1) … 
以上的SQL语句只使用建立在FIELD1上的索引，而不使用其它字段上的索引。

忽略索引 IGNORE INDEX

SELECT * FROM TABLE1 IGNORE INDEX (FIELD1, FIELD2) … 
在上面的SQL语句中，TABLE1表中FIELD1和FIELD2上的索引不被使用。

关闭查询缓冲 SQL_NO_CACHE

SELECT SQL_NO_CACHE field1, field2 FROM TABLE1; 
有一些SQL语句需要实时地查询数据，或者并不经常使用（可能一天就执行一两次）,这样就需要把缓冲关了,不管这条SQL语句是否被执行过，服务器都不会在缓冲区中查找，每次都会执行它。

MySQL force Index 强制索引：强制查询缓冲 SQL_CACHE

SELECT SQL_CALHE * FROM TABLE1; 
如果在my.ini中的query_cache_type设成2，这样只有在使用了SQL_CACHE后，才使用查询缓冲。

优先操作 HIGH_PRIORITY

HIGH_PRIORITY可以使用在select和insert操作中，让MySQL知道，这个操作优先进行。

SELECT HIGH_PRIORITY * FROM TABLE1; 
滞后操作 LOW_PRIORITY

LOW_PRIORITY可以使用在insert和update操作中，让MySQL知道，这个操作滞后。

update LOW_PRIORITY table1 set field1= where field1= … 
延时插入 INSERT DELAYED

INSERT DELAYED INTO table1 set field1= … 
INSERT DELAYED INTO，是客户端提交数据给MySQL，MySQL返回OK状态给客户端。而这是并不是已经将数据插入表，而是存储在内存里面等待排队。当MySQL有空余时，再插入。另一个重要的好处是，来自许多客户端的插入被集中在一起，并被编写入一个块。这比执行许多独立的插入要快很多。坏处是，不能返回自动递增的ID，以及系统崩溃时，MySQL还没有来得及插入数据的话，这些数据将会丢失。

强制连接顺序 STRAIGHT_JOIN

SELECT TABLE1.FIELD1, TABLE2.FIELD2 FROM TABLE1 STRAIGHT_JOIN TABLE2 WHERE … 
由上面的SQL语句可知，通过STRAIGHT_JOIN强迫MySQL按TABLE1、TABLE2的顺序连接表。如果你认为按自己的顺序比MySQL推荐的顺序进行连接的效率高的话，就可以通过STRAIGHT_JOIN来确定连接顺序。

MySQL force Index 强制索引:强制使用临时表 SQL_BUFFER_RESULT

SELECT SQL_BUFFER_RESULT * FROM TABLE1 WHERE … 
当我们查询的结果集中的数据比较多时，可以通过SQL_BUFFER_RESULT.选项强制将结果集放到临时表中，这样就可以很快地释放MySQL的表锁（这样其它的SQL语句就可以对这些记录进行查询了），并且可以长时间地为客户端提供大记录集。

分组使用临时表 SQL_BIG_RESULT和SQL_SMALL_RESULT

SELECT SQL_BUFFER_RESULT FIELD1, COUNT(\*) FROM TABLE1 GROUP BY FIELD1; 
一般用于分组或DISTINCT关键字，这个选项通知MySQL，如果有必要，就将查询结果放到临时表中，甚至在临时表中进行排序。SQL_SMALL_RESULT比起SQL_BIG_RESULT差不多，很少使用。

# 创建索引 

索引的创建可以在CREATE TABLE语句中进行，也可以单独用CREATE INDEX或ALTER TABLE来给表增加索引。以下命令语句分别展示了如何创建主键索引（PRIMARY KEY），唯一索引（UNIQUE）和普通索引（INDEX）的方法。 

```sql
mysql>ALTER TABLE `表名` ADD INDEX `索引名` (column list);  
  
mysql>ALTER TABLE `表名` ADD UNIQUE `索引名` (column list);  
  
mysql>ALTER TABLE `表名` ADD PRIMARY KEY `索引名` (column list);  
  
mysql>CREATE INDEX `索引名` ON `表名` (column_list);  
  
mysql>CREATE UNIQUE INDEX `索引名` ON `索引名` (column_list);  
  
mysql>ALTER TABLE `表名` ADD INDEX (`id`,`order_id`);给article表增加id索引，order_id索引  
  
mysql>ALTER TABLE `表名` ADD INDEX `id`;//给article表增加id索引  
```


2、重建索引 

重建索引在常规的数据库维护操作中经常使用。在数据库运行了较长时间后，索引都有损坏的可能，这时就需要重建。对数据重建索引可以起到提高检索效率。 

Sql代码  收藏代码
mysql> REPAIR TABLE `table_name` QUICK;  


3、查询数据表索引 

mysql> SHOW INDEX FROM `table_name`; 

4、删除索引 

删除索引可以使用ALTER TABLE或DROP INDEX语句来实现。DROP INDEX可以在ALTER TABLE内部作为一条语句处理，其格式如下： 

Sql代码  收藏代码
mysql>DROP index `index_name` ON `table_name` (column list);  
  
mysql>ALTER TABLE `table_name` DROP INDEX `index_name` (column list);  
  
mysql>ALTER TABLE `table_name` DROP UNIQUE `index_name` (column list);  
  
mysql>ALTER TABLE `table_name` DROP PRIMARY KEY `index_name` (column list);  


在前面的三条语句中，都删除了table_name中的索引index_name。而在最后一条语句中，只在删除PRIMARY KEY索引中使用，因为一个表只可能有一个PRIMARY KEY索引，因此也可不指定索引名。如果没有创建PRIMARY KEY索引，但表具有一个或多个UNIQUE索引，则MySQL将删除第一个UNIQUE索引。如果从表中删除某列，则索引会受影响。对于多列组合的索引，如果删除其中的某列，则该列也会从索引中删除。如果删除组成索引的所有列，则整个索引将被删除。 

5、强制使用索引 

Sql代码  收藏代码
mysql>SELECT * FROM TABLE1 FORCE INDEX (索引名或PRIMARY) ;  


6、联合索引 

Sql代码  收藏代码
mysql>alter table test add key id_a_b(a,b) ;  


对于联合索引当条件为 a=1 and b=1 则使用索引 ，当a=1 时也使用索引 当单独使用b=1时则不使用索引。 