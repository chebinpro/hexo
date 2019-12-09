---
title: MySQL锁机制
date: 2018-11-26 00:13:46
tags: MySQL
---
在同一时刻，如果数据库仅仅为单个MySQL客户机提供服务，仅通过事务机制即可实现数据库的数据一致性。但更多时候，在同一时刻，多个并发用户往往需要同时访问（检索或者更新）数据库中的同一个数据，此时仅仅通过事务机制无法保证多用户同时访问同一数据的数据一致性（参看场景描述2），因此有必要引入另一种机制实现数据的多用户并发访问。锁机制是MySQL实现多用户并发访问的基石。

<!--more-->

# 锁机制的必要性

MySQL客户机A与MySQL客户机B执行同一条SQL语句"select * from account;"时产生的结果截然不同，继而产生数据不一致问题。这种数据不一致问题产生的深层次原因在于，内存中的数据与外存中的数据不同步造成的（或者说是由内存中的表记录与外存中的表记录之间存在“同步延迟”造成的）。

MySQL客户机A访问数据时，如果能够对该数据“加锁”，阻塞（或者延迟）MySQL客户机B对该数据访问，直到MySQL客户机A数据访问结束，内存与外存中的数据同步后，MySQL客户机A对该数据“解锁”，“解锁”后，被阻塞的MySQL客户机B“被唤醒”，继而可以继续访问该数据，这样就可以实现多用户下数据的并发访问，如下图所示。  

{% asset_img 10.png %}

简言之，内存数据与外存数据之间的同步延迟，可以通过锁机制将“并发访问”延迟，进而实现数据的一致性访问以及并发访问。

当然，单条更新语句运行期间也会产生同步延迟。例如，场景描述2中，MySQL客户机A执行下面的update语句时，该update语句的执行过程可以简单描述为如下步骤，如下图所示。  
```sql
update account set balance = balance + 800 where account_no = 2;
```
{% asset_img 11.png %}

步骤1：使用索引查询是否存在“account_no=2”的账户信息。
步骤2：若存在，将该账户信息从外存加载到内存，在内存中生成old记录。
步骤3：修改old记录中的balance字段值，在内存中生成new记录。
步骤4：将内存中的new记录写入到外存，完成update操作。
	
上述每一个步骤的执行都需要一定的时间间隔（虽然短暂）。单个update语句运行期间，从步骤1运行到步骤4同样会产生延迟，这种延迟根本就无法避免，数据库开发人员也无需理会这种延迟，毕竟单条SQL语句运行期间会作为一个“原子”操作运行。数据库开发人员需要考虑的问题是:如何借助锁机制，解决**多用户并发访问**可能引起的数据不一致问题？

# MySQL锁机制的基础知识

简单地说，MySQL锁机制涉及的内容包括：锁的粒度、隐式锁与显式锁、锁的类型、锁的钥匙以及锁的生命周期等。

## 锁的粒度
锁的粒度是指锁的作用范围。就像读者有了防盗门的钥匙就可以回到“家”中，有了卧室的钥匙就可以进到卧室，有了保险柜的钥匙就可以打开保险柜，每一种“资源”存在一个与之对应“粒度”的锁，数据库亦是如此。对于MySQL而言，锁的粒度可以分为服务器级锁（server-level locking）和存储引擎级锁（storage-engine-level locking）。

服务器级锁是以服务器为单位进行加锁，它与表的存储引擎无关。在MySQL基础知识章节中讲解数据库备份时，为了保证数据备份过程中不会有新的数据写入，使用MySQL命令“flush tables with read lock;”锁定了当前MySQL服务实例，该锁是服务器级锁，并且是服务器级“读锁”。

也就是说， MySQL客户机A执行了MySQL命令“flush tables with read lock;”，锁定了当前MySQL服务实例后，MySQL客户机A针对服务器的写操作（例如insert，update，delete以及create等语句）抛出如下错误信息。
```sql
ERROR 1223（HY000）: Can't execute the query because you have a conflicting read lock
```
其他MySQL客户机（例如MySQL客户机B）针对服务器的写操作（例如insert，update，delete以及create等语句）被阻塞。

只有MySQL客户机A执行“unlock tables;”命令或者关闭MySQL客户机A的服务器连接，释放服务器级读锁后，才会“唤醒”MySQL客户机B的写操作，MySQL客户机B的写操作才能得以继续执行。MySQL客户机A施加的服务器级锁，只有MySQL客户机A才能解锁。

例如，在MySQL客户机A上锁定了当前MySQL服务实例后，在MySQL客户机B上创建视图test view将被阻塞，而在MySQL客户机A上创建视图test view将产生错误信息（ERROR 1223）。MySQL客户机A解锁后，MySQL客户机B才能成功创建视图test view，如下图所示（注意图中的粗体字）。从执行结果可以看出，MySQL客户机A施加服务器级锁后，该锁对MySQL客户机A的后续操作以及对MySQL客户机B的后续操作产生的效果并不相同。

{% asset_img 12.png %}

存储引擎级锁分为表级锁以及行级锁。表级锁是以表为单位进行加锁，MyISAM与InnoDB存储引擎的表都支持表级锁。行级锁是以记录为单位进行加锁，在MyISAM与InnoDB存储引擎中，只有InnoDB存储引擎支持行级锁。

小结：服务器级锁的粒度最大，表级锁的粒度次之，行级锁的粒度最小。锁粒度越小，并发访问性能就越高，越适合做并发更新操作（InnoDB表更适合做并发更新操作）；锁粒度越大，并发访问性能就越低，越适合做并发查询操作（MyISAM 表更适合做并发查询操作）。另外，锁粒度越小，完成某个功能时所需要的加锁、解锁的次数就会越多，反而会消耗较多的服务器资源，甚至会出现资源的恶性竞争，甚至发生死锁问题。

对于“选课系统”而言，系统需要为上百名学生，甚至几百名学生同时提供选课、调课、退课服务。为了提高并发性能，“选课系统”将选用行级锁，这也是“选课系统”的各个数据库表使用InnoDB存储引擎的原因（InnoDB存储引擎支持行级锁）。

## 隐式锁与显式锁
MySQL锁分为隐式锁以及显式锁。多个MySQL客户机并发访问同一个数据时，为保证数据的一致性，数据库管理系统会自动地为该数据加锁、解锁，这种锁称为隐式锁。隐式锁无需数据库开发人员维护（包括粒度、加锁时机、解锁时机等）。

如果应用系统存在多用户并发访问数据的行为，有时单靠隐式锁无法实现数据的一致性访问要求（例如多个学生同时选修同一门课程），此时需要数据库开发人员手动地加锁、解锁，这种锁称为显式锁。对于显式锁而言，数据库开发人员不仅需要确定锁的粒度，还需要确定锁的加锁时机（何时加锁）、解锁时机（何时解锁）以及锁的类型。

## 锁的类型
锁的类型包括读锁（read lock）和写锁（write lock），其中读锁也称为共享锁，写锁也称为排他锁或者独占锁。

读锁（read lock）：如果MySQL客户机A对某个数据施加了读锁，加锁期间允许其他MySQL客户机（例如MySQL客户机B）对该数据施加读锁，但会阻塞其他MySQL客户机（例如MySQL客户机C）对该数据施加写锁，除非MySQL客户机A释放该数据的读锁。简言之，读锁允许其他MySQL客户机对数据同时“读”，但不允许其他MySQL客户机对数据任何“写”（如下图所示）。如果“数据”是表，则该读锁是表级读锁；如果“数据”是记录，则该读锁是行级读锁。

{% asset_img 1.png %}

写锁（write lock）：如果MySQL客户机A对某个数据施加了写锁，加锁期间会阻塞其他MySQL客户机（例如MySQL客户机B）对该数据施加读锁以及写锁，除非MySQL客户机A释放该数据的写锁。简言之，写锁不允许其他MySQL客户机对数据同时“读”，也不允许其他MySQL客户机对数据同时“写”（见下图）。如果“数据”是表，则该写锁是表级写锁；如果“数据”是记录，则该写锁是行级写锁。

{% asset_img 2.png %}

## 锁的钥匙

多个MySQL客户机并发访问同一个数据时，如果MySQL客户机A对该数据成功地施加了锁，那么只有MySQL客户机A拥有这把锁的“钥匙”，也就是说，只有MySQL客户机A能够对该锁进行解锁操作。

## 锁的生命周期

锁的生命周期是指在同一个MySQL会话内，对数据加锁到解锁之间的时间间隔。锁的生命周期越长，并发访问性能就越低；锁的生命周期越短，并发访问性能就越高。另外，锁是数据库管理系统重要的数据库资源，需要耗费一定的服务器内存，锁的生命周期越长，该锁占用服务器内存的时间间隔就越长；锁的生命周期越短，该锁占用服务器内存的时间间隔就越短。因此为了节省服务器资源，数据库开发人员必须尽可能的缩短锁的生命周期，尽可能早地释放锁资源。

小结：不恰当的锁粒度、锁生命周期不仅会影响数据库的并发性能，还会造成锁资源的浪费。

# MyISAM表的表级锁

对MyISAM存储引擎的表进行检索（select）操作时，select语句执行期间（时间间隔虽然短暂），MyISAM存储引擎会自动地给涉及到的MyISAM表施加“隐式读锁”；select语句执行完毕后，MyISAM存储引擎会自动地为这些表进行“解锁”。因此select语句的执行时间就是“隐式读锁”的生命周期。

对MyISAM存储引擎的表进行更新（insert、update以及delete）操作时，更新语句（例如insert、update以及delete）执行期间（时间间隔虽然短暂），MyISAM存储引擎会自动地给涉及到的MyISAM表施加“隐式写锁”；更新语句执行完毕后，MyISAM存储引擎会自动地为这些表进行解锁，更新语句的执行时间就是“隐式写锁”的生命周期。

可以看到，任何针对MyISAM表的查询操作或者更新操作，都会隐式地施加表级锁。隐式锁的生命周期非常短暂，且不受数据库开发人员的控制。

有时，应用系统要求数据库开发人员延长MyISAM表级锁的生命周期，MySQL为数据库开发人员提供了显式地施加表级锁以及显式地解锁的MySQL命令，以便数据库开发人员能够控制MyISAM表级锁的生命周期，MySQL客户机A施加表级锁以及解锁的MySQL命令的语法格式如下图所示。

{% asset_img 3.png %}

注意事项：

（1）上述语法格式主要针对MyISAM表显式地施加表级锁以及解锁，该语法格式同样适用于InnoDB表。只不过因为InnoDB表支持行级锁，在InnoDB表中表级锁的概念比较淡化。

（2）read与write选项的功能在于说明施加表级读锁还是表级写锁。对表施加读锁后，MySQL客户机A对该表的后续更新操作将出错，错误信息如上图所示；MySQL客户机B对该表的后续查询操作可以继续进行，而对该表的后续更新操作将被阻塞。出错与阻塞是两个不同的概念。

MySQL客户机A对表施加写锁后，MySQL客户机A的后续查询操作以及后续更新操作都可以继续进行；MySQL客户机B对该表的后续查询操作以及后续更新操作都将被阻塞。

MySQL客户机A为某个表加锁后，加锁期间MySQL客户机A对该表的后续操作，MySQL客户机B对该表的后续操作以及MySQL客户机B对该表加锁之间的关系如下表所示。

{% asset_img 4.png %}

（3）MySQL客户机A使用lock tables命令可以同时为多个表施加表级锁（包括读锁或者写锁），并且加锁期间，MySQL客户机A不能对“没有锁定的表”进行更新及查询操作，否则将抛出“表未被锁定”的错误信息。例如，在MySQL客户机A上运行下面的MySQL代码，对account表施加读锁，加锁期间对book表的查询操作将抛出错误信息。读者可以自行分析，使用显式锁后，锁的生命周期是否延长。
```sql
alter table account engine=MyISAM;
alter table book engine=MyISAM;
lock tables account read;
select * from account;  ERROR 1100（HY000）: Table 'book' was not locked with LOCK TABLES
select * from book;
unlock tables;
```
（4）如果需要为同一个表同时施加读锁与写锁，那么需要为该表起两个别名，以区分读锁与写锁。
例如，下面的MySQL代码首先将account表的存储引擎设置为MyISAM。然后向account表同时施加读锁（account表的别名为a）以及写锁（account表的别名为b）。接着将account表重命名为a进行查询操作，将account表重命名为b进行查询操作。如果直接查询account表中的所有记录，则将抛出错误信息，原因是并没有为account表施加一个名字为account的锁，抛出错误信息"account表未被锁定"也在情理之中，执行结果如下所示。读者可以自行分析，使用显式锁后，锁的生命周期是否延长。
```sql
alter table account engine=MyISAM;
lock tables account as a read，account as b write;
select * from account as a;
select * from account as b;
select * from account; ERROR 1100（HY000）: Table 'account' was not locked with LOCK TABLES
unlock tables;
```
说明
为了便于理解，读者可以认为每个表的锁必须有锁名，且默认情况下锁名就是表名。当某个表既存在读锁又存在写锁时，需要为表名起多个别名，且每个别名对应一个锁名。

（5）read local与read选项之间的区别在于，如果MysQL客户机A使用read选项为某个MyISAM表施加读锁，加锁期间，MySQL客户机A以及MySQL客户机B都不能对该表进行插入操作。如果MySQL客户机A使用read local选项为某个MyISAM表施加读锁，加锁期间，MySQL客户机B可以对该表进行插入操作，前提是新记录必须插入到表的末尾。对InnoDB表施加读锁时，read local选项与read选项的功能完全相同。

**场景描述8:** read local与read选项之间的区别。
首先在MySQL客户机A上执行下面的MySQL命令，并为account表施加local读锁。
```sql
alter table account engine=MyISAM;
lock tables account read local;
```
然后打开MySQL客户机B，在MySQL客户机B上执行下面的insert语句，向account表中添加一条记录。从执行结果可以看出，MySQL客户机A为account表施加local读锁后，MySQL客户机B可以向account表中添加记录。local关键字使得MyISAM表最大限度地支持查询和插入的并发操作。
```sql
insert into account values（null， '丁'， 1000）;
```
最后在MySQL客户机A上执行下面的MySQL命令，为account表解锁。
```sql
unlock tables;
```
（6）MySQL客户机A对某个表施加读锁的同时， MySQL客户机B对该表施加写锁，默认情况下会优先施加写锁，这是因为更新操作比查询操作更为重要。如果MySQL客户机C...Z对该表同时也施加了写锁，可能造成读锁“饿死”。为了避免读锁“饿死”，MySQL客户机B....Z可以使用low_priority write选项降低写锁的优先级，以便MySQL客户机A及时取得读锁，不被饿死。

（7）unlock tables用于解锁，它会解除当前MySQL服务器连接中所有MyISAM表的所有锁。

（8）lock tables与unlock tables语句会引起事务的隐式提交。

（9）MySQL客户机一旦关闭， unlock tables语句将会被隐式地执行。因此，如果要让表锁定生效就必须一直保持MySQL服务器连接。

# InnoDB表的行级锁

InnoDB表的锁比MyISAM表的锁更为复杂，原因在于InnoDB表既支持表级锁，又支持行级锁，又存在意向锁，再把事务掺入其中，会给初学者的学习带来不少麻烦。使用lock tables命令为InnoDB表施加表级锁与使用lock tables命令为MyISAM表施加表级锁的用法基本相同，不再赘述，这里主要讨论InnoDB行级锁以及意向锁的用法。

InnoDB提供了两种类型的行级锁，分别是共享锁（S）以及排他锁（X），其中共享锁也叫读锁，排他锁也叫写锁。InnoDB行级锁的粒度仅仅是受查询语句或者更新语句影响的那些记录。在查询（select）语句或者更新（ insert，update以及delete）语句中，为受影响的记录施加行级锁的方法也非常简单。

方法1:在查询（select）语句中，为符合查询条件的记录施加共享锁，语法格式如下所示。
	select * from 表 where 条件语句 **lock in share mode**;

方法2:在查询（select）语句中，为符合查询条件的记录施加排他锁，语法格式如下所示。
	select * from 表 where 条件语句 **for update**;

方法3:在更新（insert，update以及delete）语句中， InnoDB存储引擎将符合更新条件的记录自动施加排他锁（隐式锁），即InnoDB存储引擎自动地为更新语句影响的记录施加隐式排他锁。

说明
方法1与方法2是显式地施加行级锁，方法3是隐式地施加行级锁。这3种方法施加的行级锁的生命周期非常短暂，为了延长行级锁的生命周期，最为通用的做法是开启事务。事务提交或者回滚后，行级锁才被释放，这样就可以延长行级锁的生命周期，此时事务的生命周期就是行级锁的生命周期。

**场景描述9:**通过事务延长行级锁的生命周期。

步骤1:在MySQL客户机A上执行下面的MySQL语句，开启事务，并为student表施加行级写锁。
```sql
use choose;
start transaction;
select * from student for update;
```

步骤2 :打开MySQL客户机B ，在MySQL客户机B上执行下面的MySQL语句，开启事务，并为student表施加行级写锁。此时， MySQL客户机B被阻塞。
```sql
use choose;
start transaction;
select * from student for update;
```

步骤3:在MySQL客户机A上执行下面的MySQL命令，为student表解锁。此时，MySQL客户机A释放了student表的行级写锁， MySQL客户机B被“唤醒" ，得以继续执行。
```sql
commit;
```
可以看到，通过事务延长了MySQL客户机A针对student表的行级锁的生命周期。

结论:事务中的行级共享锁（S）以及行级排他锁（X）的生命周期从加锁开始，直到事务提交或者回滚，行级锁才会释放。
MySQL客户机A使用"select * from 表 where 条件语句 lock in share mode;"为InnoDB表中符合条件语句的记录施加共享锁后，加锁期间，MySQL客户机A可以对该表的所有记录进行查询以及更新操作。加锁期间，MySQL客户机B可以查询该表的所有记录（甚至施加共享锁），可以更新不符合条件语句的记录，然而为符合条件语句的记录施加排他锁时将被阻塞。

MySQL客户机A使用"select * from 表 where 条件语句 for update;"或者更新语句（例如insert，update以及delete）为InnoDB表中符合条件语句的记录施加排他锁后，加锁期间，MySQL客户机A可以对该表的所有记录进行查询以及更新操作。加锁期间，MySQL客户机B可以查询该表的所有记录，可以更新不符合条件语句的记录，然而为符合条件语句的记录施加共享锁或者排他锁时将被阻塞。
	
为了便于读者更好地理解共享锁以及排他锁之间的关系，可以参看表9-2所示的内容。

# "选课系统"中的行级锁

场景描述10 :实现调课功能的存储过程replace_course_proc（）存在功能缺陷。考虑这样的场景:张三与李四"同时"选择同一门目标课程，且目标课程就剩下一个席位（此时目标课程available的字段值为1）。张三以及李四为了实现调课功能，"同时"调用存储过程replace_course_proc（），假设两人“同时”执行存储过程中的select语句"查询目标课程available字段值"：
```sql
select available into s from course where course_no=c_after;
```
张三以及李四可能都读取到available的值为1（大于零），最后的结果是张三与李四都选择了目标课程。
	
可以看出，存储过程replace_course_proc（）读取课程的available字段值时，有必要为张三与李四选择相同的目标课程施加排他锁，避免多名学生同时读取同一门课程的available字段值。将存储过程replace_course_proc（）中的代码片段:
```sql
select available into s from course where course_no=c after;
```
修改为如下的代码片段：
```sql
select available into s from course where course_no=c_after for update;
```

说明
为了延长行级排他锁的生命周期，将该select语句写在了start transaction语句后，封装到事务中。

此时，当张三、李四以及其他更多的学生同时“争夺”同一门目标课程的最后一个席位时，可以保证只有一个学生能够读取该席位，其他学生将被阻塞（如图9-27所示）。这样就可以防止张三与李四都选择了目标课程的最后一个席位。很多读者可能觉得:多个学生同时选择最后一个席位"的可能性微乎其微，但如果最后的一个"席位"是春运期间某趟列车的最后一张火车票呢?现实生活中，类似的"资源竞争"问题还有很多（例如团购、秒杀等），使用锁机制可以有效解决此类“资源竞争”问题。
