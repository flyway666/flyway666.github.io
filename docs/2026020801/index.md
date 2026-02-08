# PostgreSQL锁机制

#PostgreSQL中有两类锁：表级锁和行级锁。当要查询、插入、更新、删除表中数据时，首先要获得表级锁，然后获得行级锁。

下面对PostgreSQL数据库锁机制的理解，大部分来自与《PostgreSQL修炼之道 从小工到专家》-唐成书中，以及网络上的博客的总结。通过实际测试发现，还是存在一些不合理的点，后面实际的案列中，会有一些说明。

1.表级锁模式
|  锁模式   | 解释  |
|  ----  | ----  |
| Access Share  | 只与Access Exclusive锁模式冲突。	 |
|	Access Share	|	查询命令（Select command）将会在它查询的表上获取Access Shared锁，一般地，任何一个对表上的只读查询操作都将获取这种类型锁。	|
|	Row Share	|	与Exclusive和Access Exclusive锁模式冲突。	|
|	Row Share	|	Select for update和Select for share命令将获得这种类型锁，并且所有被引用但没有for update 的表上会加上Access Shared锁。	|
|	Row Exclusive	|	与Share，Shared Row Exclusive，Exclusive，Access Exclusive模式冲突。	|
|	Row Exclusive	|	Update/Delete/Insert命令会在目标表上获得这种类型的锁，并且在其它被引用的表上加上Access Share锁，一般地，更改表数据的命令都将在这张表上获得Row Exclusive锁。	|
|	Share Update Exclusive	|	Share Update Exclusive，Share，Share Row Exclusive，Exclusive，Access exclusive模式冲突，这种模式保护一张表不被并发的模式更改和Vacuum。	|
|		|	Vacuum(without full)，Analyze 和 Create index concur-ently命令会获得这种类型锁。	|
|	Share	|	与Row Exclusive，Shared Update Exclusive，Share Row Exclusive，Exclusive，Access exclusive锁模式冲突，这种模式保护一张表数据不被并发的更改。	|
|	Share	|	Create index命令会获得这种锁模式。	|
|	Share Row Exclusive	|	与Row Exclusive，Share Update Exclusive，Shared，Shared Row Exclusive，Exclusive，Access Exclusive锁模式冲突。	|
|	Share Row Exclusive	|	任何PostgreSQL命令不会自动获得这种类型的锁。	|
|	Exclusive	|	与ROW Share , Row Exclusive, Share  Update  Exclusive, Share , Share  Row Exclusive, Exclusive, Access Exclusive模式冲突，这种锁模式仅能与Access Share 模式并发，换句话说，只有读操作可以和持有Exclusive锁的事务并行。	|
|	Exclusive	|	任何PostgreSQL命令不会自动获得这种类型的锁。	|
|	Access Exclusive	|	与所有模式锁冲突(Access Share，Row Share，Row Exclusive，Share  Update Exclusive，Share , Share Row Exclusive，Exclusive，Access Exclusive)，这种模式保证了当前只有一个人访问这张表；ALTER TABLE，DROP TABLE，TRUNCATE，REINDEX，CLUSTER，VACUUM FULL 命令会获得这种类型锁，在Lock table 命令中，如果没有申明其它模式，它也是默认模式。	|


2.表级锁的冲突矩阵
请求的锁模式

| 请求的锁模式 | "Access Share" | "Row Share" | "Row Exclusive" | "Share Update Exclusive" |Share | "Share Row Exclusive" | Exclusive | "Access Exclusive"|
|  ----  | ----  |----  |----  |----  |----  |----  |----  |----  |
|Access Share | Y | Y | Y | Y | Y | Y | Y | N|
|Row Share | Y | Y | Y | Y | Y | Y | N | N|
|Row Exclusive | Y | Y | Y | Y | N | N | N | N|
|Share Update Exclusive | Y | Y | Y | N | N | N | N | N|
|Share | Y | Y | Y | N | N | N | N | N|
|Share Row Exclusive | Y | Y | N | N | N | N | N | N|
|Exclusive | Y | N | N | N | N | N | N | N|
|Access Exclusive | N | N | N | N | N | N | N | N|


表中“N”表示这两种表冲突，也就是不同的进程不能同时持有这两种锁。

最普通的是Share和Exclusive这两种锁，它们分别是读、写锁的意思。加了Share锁，即读锁，表的内容就不被修改了；可以为多个事务加上此锁，只要任意一个事务不释放这个读锁，则其他事务就不能修改这个表。加上了Exclusive，相当于加了写锁，这时别的进程不能写也不能读这条数据。但后来数据库又加上了多版本的功能。修改一条语句的同时，允许了读数据，为了处理这种情况，又增加了两种锁Access Share和Access Excusive，锁中的关键字 Access 是与多版本相关的有了该功能。其实，有了该功能后，如果修改一行数据，实际并没有改原先那行数据，而是复制了一个新行，修改都在新行上，事务不提交，其他人是看不到修改的这条数据的。由于旧行数据没有变化，在修改过程中，读数据的人仍然可以读到旧的数据。

表级锁加锁对象是表，这使得加锁范围太大，导致并发并不高，于是人们提出了行级锁的概念，但行级锁与表级锁之间会产生冲突，这时需要一种机制来描述行级锁与表级锁之间的关系，有了意向锁的概念，这时又加了两种锁，即意向共享锁（Row Share） 和意向排他锁（Row Exclusive），由于意向锁之间不会产生冲突，因为他们是“有意”做，还没真做；而且意向排它锁相互之间也不会产生冲突，于是又需要更严格一些的锁，这样就产生了Share Update Exclusive，Share Row Exclusive可以看成Share与Row Exclusive，PostgreSQL不会自动请求这个锁模式，也就是PostgreSQL内部目前没有使用这种锁。

这里稍微补充一下多版本并发控制原理。

多版本并发控制原理：

大家熟知的读与写锁是不能并发的，所以有人想到一种新的能够让读写并发的方法，称这种方法为MVCC。MVCC的方法是写数据时，旧版本的数据并不删除，并发的读操作还能读到旧版本的数据。

实现MVCC的两种方法：

写新数据时，把旧数据移到一个单独的地方，如回滚段中，其他读数据时，从回滚段中把旧版本数据读出来。
写新数据时，旧版本的数据不删除，而是把新数据插入。
PostgreSQL数据库使用的正是第二种方法，而oracle与MySQL中的innodb引擎用的是第一种方法。

PostgreSQL实现该功能，需要在每张表上添加四个系统字段tmin、tmax、cmin、cmax，通过这四个字段可以区分并发时，记录不同数据的版本，和事务标识，当删除时，只会标记记录，而不会从数据块中删除，空间也没有立即释放。

PostgreSQL通过运行vaccum进程来进行回收之前的存储空间，默认PostgreSQL中的autovacuum是打开的，当一表更新达到一定数量时，autovacuum会自动回收空间。

3.表级锁类型对应的数据库操作
锁类型

对应的数据库操作

Access Share

select

Row Share

select for update, select for share

Row Exclusive

update，delete，insert

Share Update Exclusive

vacuum(without full)，analyze，create index concurrently

Share

create index

Share Row Exclusive

任何Postgresql命令不会自动获得这种类型的锁

Exclusive

任何Postgresql命令不会自动获得这种类型的锁

Access Exclusive

alter table，drop table，truncate，reindex，cluster，vacuum full

4.行级锁模式
行级锁模式只有两种，分别是共享锁和排他锁，或者说是读锁或写锁。由于多版本的实现，实际读取行数据时，并不会在行上执行任何锁。

特定行上的行级锁是在行被更新的时候自动请求的（或者被删除时或标记为更新）。 锁一直保持到事务提交或者回滚。行级锁不影响对数据的查询；它们只阻塞对同一行的写入。 要在不修改某行的前提下请求在该行的行级锁，用 SELECT FOR UPDATE 选取该行。请注意一旦我们请求了特定的行级锁，那么该事务就可以多次对该行进行更新而不用担心冲突。

《PostgreSQL修炼之道 从小工到专家》-唐成书中强调的是，行级锁只有共享锁与排他锁，也就是读锁、写锁，但是实际测试中有发现一些tuple类型的多版本控制的写锁。

5.页级锁
除了表级别的和行级别的锁以外， 页面级别的共享/排他销也用于控制对共享缓冲池中表页面的读/写访问。这些锁在抓取或者更新一行后马上被释放。应用程序员通常不需要关心页级锁，在这里提到它们只是为了完整。

6.锁命令
1.表级锁命令

在PsotgreSQL中，显示的在表上加锁命令为LOCK TABLE命令：

LOCK [ TABLE ] [ ONLY ] name [ * ] [, ...] [ IN lockmode MODE ] [ NOWAIT ]

说明如下：

name：要锁定的现有表的锁名称(可选模式限定)。 如果在表名之前指定ONLY，则仅该表被锁定，如果未指定ONLY，则表及其所有后代表(如果有)被锁定。
lock_mode：锁模式指定此锁与之冲突的锁。 如果未指定锁定模式，则使用最严格的访问模式ACCESS EXCLUSIVE。
NOWAIT：如果没有NOWAIT关键字时，当无法获得锁时，会一直等待，而如果加了NOWAIT关键字，在无法立即获取该锁时，此命令会立即退出并发出一个错误信息
2.行级锁命令

在PsotgreSQL中，显示的行级锁命令是由select命令发出的：

SELECT …… FOR  { UPDATE | SHARE } [OF table_name[,……]] [ NOWAIT]

说明如下：

指定 OF table_name，则只有被指定的表会被锁定。
例外情况，主查询中引用了WITH查询时，WITH查询中的表不被锁定。
如果需要锁定WITH查询中的表，需在WITH查询内指定FOR UPDATA或FOR SHARE。
NOWAIT关键字与显示加表级锁命令作用一样。
7.死锁
死锁是指两个或者两个以上的事务在执行过程中相互持有对方期待的资源，若没有其他机制，它们将无法进行下去。

1.形成死锁的4个必要条件

请求与保持条件：获取资源的进程可以同时申请新的资源。
非剥夺条件：已经分配的资源不能从该进程中剥夺。
循环等待条件：多个进程构成环路，并且每个进程都在等待相邻进程正占用的资源。
互斥条件：资源只能被一个进程使用。
 2.减少死锁的方法

在所有事务中都以相同的次序使用资源。
使事务尽可能简单并在一个批处理中。
为死锁超时参数设置一个合理范围，如10s；超时则自动放弃本操作，避免进程挂起。可以在postgresql.conf文件中设置：log_lock_waits = on         deadlock_timeout = 10s
避免在事务内核用户进行交互，减少资源的锁定时间。
使用较低的隔离级别，相比较高的隔离级别能够有效减少持有共享锁的时间，减少锁之间的竞争。
打

