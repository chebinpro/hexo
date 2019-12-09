---
title: MySQL事务机制
date: 2018-11-26 20:10:15
tags: MySQL
---
<script src="//cdnjs.cloudflare.com/ajax/libs/highlight.js/9.13.1/highlight.min.js"></script>
<link rel="stylesheet" href="https://cdn.bootcss.com/highlight.js/9.13.1/styles/monokai-sublime.min.css">
<script>
    hljs.initHighlightingOnLoad();
</script>

# 事务机制

数据库与文件系统的最大区别在于数据库实现了数据的一致性以及并发性。对于数据库管理系统而言事务机制与锁机制是实现数据一致性与并发性的基石。

事务通常包含一系列更新操作，这些更新操作是一个不可分割的逻辑工作单元。如果事务成功执行，那么该事务中所有的更新操作都会成功执行，并将执行结果提交到数据库文件中，成为数据库永久的组成部分。如果事务中某个更新操作执行失败，那么事务中的所有更新操作均被撤销。简言之：事务中的更新操作要么都执行，要么都不执行，这个特征叫做事务的原子性。
<!--more-->
说明

更新语句或更新操作主要是update、insert以及delete等语句。

由于MyISAM存储引擎暂时不支持事务，因此，使用的存储引擎为InnoDB存储引擎。

## 事务机制的必要性

对于银行系统而言，转账业务是银行最基本、最常用的业务，有必要将转账业务封装成存储过程，银行系统调用该存储过程后即可实现两个银行账户间的转账业务。

场景描述1：假设某个银行存在两个借记卡账户（account）甲与乙，并且要求这两个借记卡账户不能用于透支，即两个账户的余额（balance）不能小于零。

步骤1：创建account账户表，并将其设置为InnoDB存储引擎。account_no字段是账户表的主键，其值由MySQL自动生成；account_name字段是账户名；balance字段是余额，由于余额不能为负数，将其定义为无符号数。
```sql
create table account(
	account_no int auto_increment primary key,
	account_name char(10) not null,
	balance int unsigned
) ENGINE=innodb DEFAULT CHARSET=utf8 COLLATE=utf8_general_ci;
```

步骤2：添加测试数据。下面的SQL语句向账户account表中插入了“甲”和“乙”两条账户信息，余额都是1000元。
```sql
insert into account values(null,'甲',1000);

insert into account values(null,'乙',1000);
```

步骤3：创建存储过程。下面的MySQL代码创建了transfer_proc()存储过程，将from_account账户的money金额转账到to_account账户中，继而实现两个账户之间的转账业务。
```sql
delimiter $$
	create procedure transfer_proc(in from_account int,in to_account int,in money int)

	modifies sql data

	begin

	update account set balance=balance+money where account_no=to_account;
	update account set balance=balance-money where account_no=from_account;
	end
$$
delimiter ;
```

步骤4：测试存储过程。下面的MySQL代码首先调用了transfer_proc()存储过程，将账户“甲”的800元转账到了“乙”账户中。然后查询了账户表中的所有账户信息，执行结果如图9-1所示，此时两个账户的余额之和为2000元。
```sql
call transfer_proc(1,2,800);

select * from account;
```

步骤5：再次测试存储过程。再次调用transfer_proc()存储过程，将账户“甲”的800元转账到“乙”账户中。由于账户余额不能为负数，甲的账户余额200元减去800元，产生了错误代码
1690对应的错误信息（稍后为该错误代码定义错误处理程序）。
1690 - BIGINT UNSIGNED value is out of range in '(`test`.`account`.`balance` - money@2)'

然后查询账户表中的所有账户信息：
```sql
call transfer_proc(1,2,800);
select * from account;
```
执行结果如下。
account_no | account_name | balance 
- | :-: | -: 
1 | 甲 | 200 
2 | 乙 | 2600 

结论：甲账户的余额没有丝毫变化，但是乙账户的余额却凭空多出800元。甲、乙账户的余额之和由转账前2000元，变成了转账后的2800元，由此产生了数据不一致问题。

为了避免出现数据不一致问题，需要在存储过程中引入事务的概念，将transfer_proc()存储过程的两条update语句绑定到一起，让它们成为一个“原子性”的操作：两条update语句要么都执行，要么都不执行。

## 关闭MySQL自动提交

默认情况下，MySQL开启了自动提交（auto_increment），这就意味着，之前编写的任意一条更新语句，一旦发送到MySQL服务器，MySQL服务实例会立即解析、执行，并将更新结果提交到数据库文件中，成为数据库永久的组成部分。

以转账存储过程transfer_proc()为例，该存储过程包含两条update语句，第一条update语句执行“加法”运算，第二条update语句执行“减法”运算。由于MySQL默认情况下开启了自动提交，因此第二条update语句无论执行成功还是失败，都不会影响第一条update语句的成功执行。如果第一条update语句成功执行，而第二条update语句执行失败，最终将导致数据不一致问题的发生。可以这样理解：产生上述数据不一致问题的根源在于MySQL开启了自动提交，并且没有引入事务的概念。

因此，对于诸如银行转账的业务逻辑而言，首要步骤是关闭MySQL自动提交，只有当所有的更新语句成功执行后，才提交（commit）所有的更新语句，否则回滚（rollback）所有的更新语句。关闭自动提交的方法有两种：一种是显式的关闭自动提交；另一种是隐式的关闭自动提交。

方法一：显式的关闭自动提交

使用MySQL命令“show variables like 'autocommit';”可以查看MySQL是否开启了自动提交。系统变量@@autocommit的值为ON或者1时，表示MySQL开启自动提交，默认情况下，MySQL开启了自动提交。系统变量@@autocommit 的值为OFF或者0时，表示MySQL关闭自动提交。使用MySQL命令“set autocommit=0;”可以显式的关闭MySQL自动提交。

说明

系统变量@@autocommit是<font color=red>会话变量</font>，MySQL客户机A对该系统会话变量的更改，不会影响到MySQL客户机B中该系统会话变量的值。

方法二：隐式的关闭自动提交
使用MySQL命令“start transaction;”可以隐式地关闭自动提交。隐式的关闭自动提交不会修改系统会话变量@@autocommit的值。

注意：对MyISAM存储引擎的表进行更新操作时，自动提交无论开启还是关闭，更新操作都将立即解析、执行，并将执行结果提交到数据库文件中，成为数据库永久的组成部分。

## 回滚

关闭MySQL自动提交后，数据库开发人员可以根据需要回滚（也叫撤销）更新操作。

场景描述2：接场景描述1的步骤，当甲、乙账户的余额分别变为200元以及2600元后，打开MySQL客户机A，然后输入下面的MySQL命令。该MySQL命令首先关闭MySQL的自动提交，接着修改乙账户（account_no=2）的余额（加800元），然后查询所有账户的余额。
```sql
set autocommit=0;
update account set balance=balance+800 where account_no=2;

select * from account;
```
结果为：
account_no | account_name | balance 
- | :-: | -: 
1 | 甲 | 200 
2 | 乙 | 3400 

从运行结果可以看到，乙账户的余额已经从2600元修改为3400元，事实果真如此？打开另一个MySQL客户机B，选择当前的数据库为choose数据库，使用select语句再次查询所有账户的余额。
```sql
use choose;

select * from account;
```
结果为：
account_no | account_name | balance 
- | :-: | -: 
1 | 甲 | 200 
2 | 乙 | 2600 

从MySQL客户机B的执行结果可以看到乙的余额依然是2600元，并没有增加800元。对于这个问题，可以通过下图进行解释。从图中可以得知：MySQL客户机A的update操作影响的仅仅是内存中new记录的值，且该值并没有写入数据库文件；当MySQL客户机A执行select语句时，查询到的3400元实际上是MySQL服务器内存中new记录的字段值；MySQL客户机B执行select语句时，看到的2600元是外存数据2600在服务器内存的一个副本。

{% asset_img 1.png %}

乙的最终余额究竟应该是多少元？这要取决于MySQL客户机A接下来的操作（可以分成两种情形：场景描述3与场景描述4）。

场景描述3：接场景描述2的步骤，在MySQL客户机A上执行MySQL命令“rollback;”，接着在MySQL客户机A、MySQL客户机B上执行select语句查询甲、乙账户的余额，两次执行结果相同（乙账户的余额均为2600），结果如下：
account_no | account_name | balance 
- | :-: | -: 
1 | 甲 | 200 
2 | 乙 | 2600 
可以这样理解：MySQL客户机A关闭MySQL自动提交后，MySQL客户机A执行的所有更新操作，都会在MySQL服务器内存中产生若干条new记录。如果在MySQL客户机A上执行了rollback命令，MySQL服务器内存中，与MySQL客户机A对应的new记录将被丢弃，回滚了（也叫撤销了）MySQL客户机A执行的更新操作。

说明

insert语句产生new记录，delete语句产生old记录，update语句产生new记录以及old记录，关于这方面的知识请参看视图与触发器章节的内容。

## 提交

MySQL自动提交一旦关闭，数据库开发人员需要“提交”更新语句，才能将更新结果提交到数据库文件中，成为数据库永久的组成部分。自动提交关闭后，MySQL的提交方式分为显式的提交与隐式的提交。
显式的提交：MySQL自动提交关闭后，使用MySQL命令“commit;”可以显式的提交更新语句。

隐式的提交：MySQL自动提交关闭后，使用下面的MySQL语句，可以隐式的提交更新语句。begin、set autocommit=1、start transaction、rename table、truncate table等语句；数据定义（create、alter、drop）语句，例如create database、create table、create index、create function、create procedure、alter table、alter function、alter procedure、drop database、drop table、drop function、drop index、drop procedure等语句；权限管理和账户管理语句（例如grant、revoke、set password、create user、drop user、rename user等语句）；锁语句（lock tables、unlock tables）。举例来说，MySQL客户机A关闭MySQL自动提交后，执行了若干条更新语句。此时如果MySQL客户机A执行了“create table test_commit(a int primary key);”语句，该语句成功创建test_commit表后，还会提交之前的所有更新语句。

为了有效地提交事务，推荐数据库开发人员尽可能地使用显式的提交方式，尽量不要使用（或者避免使用）隐式的提交方式。

场景描述4：重做场景描述2中的所有操作，在MySQL客户机A上执行MySQL命令“commit;”，接着在MySQL客户机A、MySQL客户机B上执行select语句查询甲、乙账户的余额，两次执行结果相同（乙账户的余额增加了800元，变为3400）。从执行结果可以看出,MySQL客户机A执行commit命令后, MySQL服务器内存中的new记录被更新到数据库文件中,成为数据库永久的组成部分。

说明
关闭MySQL的自动提交后,需要显式提交（或者隐式提交）更新语句,否则所有的更新语句影响的仅仅是MySQL服务器内存中的new记录,更新语句提交后,才会将new记录的值写入数据库文件。

无论开启自动提交,还是关闭自动提交,使用触发器时, InnoDB存储引擎都会保证触发事件与触发程序的原子性操作。

# 事务
使用MySQL命令"start transaction;"可以开启一个事务,该命令开启事务的同时,会隐式的关闭MySQL自动提交。使用commit命令可以提交事务中的更新语句;使用rollback命令可以回滚事务中的更新语句。
典型的事务处理使用方法如下图所示。

{% asset_img 1.png %}

场景描述5:银行转账业务的两条update语句是一个整体,如果其中任意一条update语句执行失败,则所有的update语句应该撤销,从而确保转账前后的总额不变。使用事务机制、错误处理机制可以避免银行转账时数据不一致问题的发生

下面的MySQL代码,首先删除原有的transfer_proc()存储过程,然后重建ransfer_proc()存储过程,并将代码修改为下面的代码。其中, "declare continue handler for 1690"负责处理MysQL错误代码1690,当发生该错误时,执行回滚操作; "start transaction"负责开启事务,并隐式地关闭自动提交;"commit"负责提交事务。
```sql
drop procedure transfer_proc;
delimiter $$
create procedure transfer_proc (in from_account int,in to_account int,in money int)
		modifies sql data
		begin
			declare continue handler for 1690
		begin
			rollback;
		end;
		start transaction;
			update account set balance=balance+money where account_no=to_account;
			update account set balance=balance-money where account_no=from_account;
		commit;
end
$$
delimiter ;
```
说明
如果账户余额balance字段定义为整数(不是无符号整数) ,那存储过程transfer_proc()也可以通过判断账户余额是否小于零,继而决定是否回滚(rollback)转账业务

默认情况下, InnoDB存储引擎既不会对异常进行回滚,也不会对异常进行提交,而这是十分危险的。异常发生后,数据库开发人员需要借助错误处理程序,显式地提交事务或者显式地回滚事务。可以这样理解:事务的提交与回滚,好比if-else语句中的then子句与else子句,两者只能选其一。

在实际的数据库开发过程中,不建议使用MySQL命令"set autocommit=0;"显式地关闭MySQL自动提交,建议选用"start transaction;"命令,该命令不仅可以开启新的事务,还可以隐式地关闭MySQL自动提交,而且"start transaction;"命令不会影响@@autocommit系统会话变量的值。

## 保存点
默认情况下,事务一旦回滚,事务内的所有更新操作都将撤销。有些时候,仅仅希望撤销事务内的一部分更新操作,保存点(也称为检查点)可以实现事务的"部分”提交或者“部分"撤销。使用MySQL命令"savepoint 保存点名;"可以在事务中设置一个保存点,使用MySQL命令"rollback to savepoint 保存点名;"可以将事务回滚到保存点状态,如下图所示。当事务回滚到保存点后,那么数据库将进入到一致性状态B。

{% asset_img 5.png %}

场景描述6:为了演示保存点的使用,下面两个存储过程save_point1_proc()与save_point2_proc()都试图在同一条事务中创建两个账号相同的银行账户。由于银行账号account_no是主键,两个银行账号不能相同,因此第二条insert语句会产生MySQL错误代码1062,两个存储过程处理MySQL错误代码1062的方法截然不同。
	
方法一:下面的MySQL代码创建了save_point1_proc()存储过程,该存储过程撤销了所有的insert语句。
```sql
delimiter $$
	create procedure save_point1_proc()modifies sql data
		begin
			declare continue handler for 1062
		begin
			rollback to B;
			rollback;
		end;
		start transaction;
			insert into account values(null,'丙', 1000);
		savepoint B;
			insert into account values(last_insert_id(),'丁', 1000);
		commit;
		end
$$
delimiter ;
```
说明
存储过程save_point1_proc()中,为了保证"丙”与"丁"两个账户的账号相同,创建"丁"账户的insert语句使用last_insert_id()函数获取了"丙"账户的账号。第二条insert语句将抛出错误信息(ERROR 1062:主键不能相同) ,第二条insert语句出错后,错误处理程序将事务回滚到B保存点(rollback to B) ,然后回滚整个事务(rollback)如上图所示。
调用存储过程save_point1_proc(),然后查询账户account表的所有记录。从查询结果可以看出,两条insert语句都被撤销。
```sql
call save_point1_proc();
select * from account;
```
mysql> call save_point1_proc();
Query OK, 0 rows affected (0.03 sec)

mysql> select * from account;
account_no | account_name | balance 
- | :-: | -: 
1 | 甲 | 200 
2 | 乙 | 3400 
2 rows in set (0.08 sec)

方法二:下面的MySQL代码创建了save_point2_proc()存储过程,该存储过程仅仅撤销第二条insert语句,但提交了第一条insert语句。
```sql
delimiter $$
create procedure save_point2_proc()modifies sql data
	begin
		declare continue handler for 1062
	begin
		rollback to B;
		commit;
	end;
	start transaction;
		insert into account values(null,'丙', 1000);
		savepoint B;
		insert into account values(last_insert_id(),'丁', 1000);
	commit;
	end
$$
delimiter ;
```
存储过程save_point2_proc()中,第二条insert语句出错后,错误处理程序将事务回滚到B保存点(rollback to B),并提交事务(commit),导致第一条insert语句成功提交到数据库中,。
	
调用存储过程save_point2_proc(),然后查询账户account表的所有记录。从查询结果可以看出,第一条insert语句成功执行,第二条insert语句被撤销。
```sql
call save_point2_proc();
select * from account;
```
mysql> call save_point2_proc();
Query OK, 0 rows affected (0.02 sec)

mysql> select * from account;
+------------+--------------+---------+
| account_no | account_name | balance |
+------------+--------------+---------+
|          1 | 甲           |     200 |
|          2 | 乙           |    3400 |
|          4 | 丙           |    1000 |
+------------+--------------+---------+
3 rows in set (0.06 sec)
说明
"rollback to savepoint B"仅仅是让数据库回到事务中的某个"一致性状态B" ,而"一致性状态B"仅仅是一个“临时状态”,该“临时状态”并没有将更新回滚,也没有将更新提交。事务回滚必须借助于rollback(而不是"rollback to savepoint B") ,而事务的提交需借助于commit
	
使用MySQL命令"release savepoint 保存点名;"可以删除一个事务的保存点。如果该保存点不存在,则该命令将出现错误信息: ERROR 1305 (42000): SAVEPOINT does not exist。如果当前的事务中存在两个相同名字的保存点,则旧保存点将被自动丢弃。

# "选课系统"中的事务
“选课系统"中,最为复杂的业务逻辑莫过于"学生选课"以及“学生调课”功能的实现,之前的章节已经编写了choose proc)存储过程,实现了学生的选课功能。本章将借用事务的概念编写调课存储过程replace course-proco,实现"选课系统”的调课功能。

场景描述7:使用存储过程实现“选课系统"的调课功能,图9-13所示的程序流程图阐述了某个学生的调课流程
(其中, c before表示调课前的课程, c after表示目标课程或者调课后的课程)。从程序流程图中可以看到,调课时,首先要判断调课前的课程与目标课程是否相同,如果相同,则将调课的状态值state设置为-1;接着判断目标课程是否已经审1核,是否已经报满,如果课程未审核或者课程available字段值为0 (课程报满) ,则将状态值state设置为-2;如果调课成功,则将状态值state设置为调课成功后的课程course no。由于调课涉及3条update语句,为了保证它们的原子性,必须将它们封装到事务中。	下面的SQL语句创建了名字为replace course proc)的存储过程,该存储过程接收学生学号(s no)、课程号(c before)以及课程号(c after)为输入参数,经过存储过程一系列处理,返回调课state状态值。如果输出参数state的值大于0,则说明学生调课成功;如果输出参数state的值等于-1,则意味着该生调课前后选择的课程相同;如果输出参数state的值等于-2,则意味着目标课程未审核或者已经报满。请读者注意粗体字代码。
```sql
delimiter $$
create procedure replace course proc(in s no char(11),in c before int,in c after int,out state int)modifies sql databegin
declare s int;
declare status char(8);set state =0;
set status="未审核";if(c before=c after) thenset state =-1;else
start transactionselect state into status from course where course no=c after;select available into s from course where course no-c after;if(s=0 1l status=未审核) therset state =-2:elseif(state=0) therupdate choose set course no-c after,choose time-now0) where student no=s no andcourse no=c before;update course set available-available+1 where course no-c beforeupdate course set available=available-1 where course no=c after;set state =c after;end if;commit;end if;end$:
delimiter;
```
下面的MySQL语句负责调用replace-course-proc)存储过程,对该存储过程进行简单的测试。首先使用下面的select语句查看学号2012002的选课信息,执行结果如图9-14所示。select * from choose where student no=2012002;


接着将该生选修的课程号3,调换为课程号1,执行下面的MySQL命令,执行结果如图9-15所示。set @s no ='2012002;set @c before =3;set @c after =1;set @state =0;call replace course proc(@s no,@c before,@c after,@state);select @state
最后使用下面的select语句查看学号2012002最终的选课信息,验证调课是否成功,执行结果如图9-16所示。select * from choose where student no=2012002;

说明
学生选课以及学生调课是"选课系统"的核心功能。存储过程与游标章节编写的"选课存储过程" chooseproco使用了触发器维护course表的available字段值。由于InnoDB存储引擎中,触发器已经保证了触发事件与触发程序的原子性,因此choose proc)存储过程可以保证insert语句与insert触发程序的原子性。而本章编写的"调课存储过
程" replace course proc)使用事务的概念,将3条update语句封装到一个事务中,同样也可以保证3条update语句的原子性操作。在真正的项目案例中,推荐使用事务实现更新语句的原子性操作,不建议使用触发器。
	一般情况下,一系列关系紧密的更新语句(例如insert, delete或者update语句)都需要封装到一个事务中。由于查询语句不会导致数据发生变化,因此一般不需要封装到事务中。细心的读者会发现,在replace course proc)存储过程中,粗体字的select语句负责"查询目标课程的available字段值" ,该select语句也封装到了事务中,具体原因在"锁机制”章节中进行讲解