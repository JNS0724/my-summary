# mysql三个日志

## 前言

事务是数据库规定的，由应用程序与其交互的、或者说数据库对数据修改的工作单元。它的粒度是一组sql语句组成的，即使是一条sql语句，也可以构成一个微小的事务。由于事务是由多条sql语句构成，就可能涉及一张表或多张表，落到存储层面，通常是多个数据页。

为了加快数据库的数据操作过程，数据库由buffer pool来加载数据页，在执行sql语句时，通常在buffer pool上进行更新，再定时刷新脏页。

以上这些属性，导致了数据库的崩溃之后，需要有一个恢复机制，主要由Redo log和Undo log来实现。

Redo log恢复已提交事务的数据，Undo log回滚未提交事务的状态。

## Redo log

重做日志主要用于在数据库崩溃后对所有已提交的事务进行数据恢复。

innodb充分利用缓存机制，Redo log有内存和磁盘两个区域，在内存上使用Redo log buffer来表示，在磁盘上由两个名为ib_logfile0和ib_logfile1的文件物理表示。MySQL以循环方式写入重做日志文件。

### Redo log buffer

Log Buffer内存空间的大小由启动参数innodb_log_buffer_size来指定，默认是16MB，这片内存空间被划分成若干个连续的redo log block,这也是Log Buffer的内存结构。

log buffer本质上是由若干个512字节大小的block组成的一片连续的内存空间。redo log落盘也是以block为单位写到日志组文件中去的。

![redolog格式](/数据库/asserts/redolog格式.png)

其中type表示Redo log的日志类型，Innodb引擎针对不同场景对数据页的修改，制定了几十种不同类型的Redo日志。Space ID和Page Number是前边我们反复提到的表空间id和数据页id，根据它们能唯一标示一个数据页，再然后的data就是对该数据页到底做了哪些修改了。

简单的redo log日志

MLOG_WRITE_STRING日志

| type | space id | page number | offset | len | content |
| ---- | -------- | ----------- | ------ | --- | ------- |

data部分需要记录三种信息：

* 要修改的内容在数据页中的偏移量（offset）
* 修改了多少个字节的数据(len)
* 修改后的数据内容是什么(content)

复杂的redolog：

物理日志混合逻辑日志

![redolog恢复数据](/%E6%95%B0%E6%8D%AE%E5%BA%93/asserts/redolog%E6%81%A2%E5%A4%8D%E6%95%B0%E6%8D%AE.png)

### Mini-Transaction

Innodb引擎对底层页的一次原子访问的过程叫做Mini-Transaction，例如更改了插入数据页后，需要同时在B+辅助索引树中插入索引信息，对于一个Mini-Transactoin产生的redo log日志都会被划分到一个组当中去，在进行系统崩溃重启恢复时，针对某个组中的redo日志，要么把全部的日志都恢复掉，要么就一条也不恢复。

每条sql语句由多个Mini-Transaction组成。

### Redo流程

![redolog恢复数据](/%E6%95%B0%E6%8D%AE%E5%BA%93/asserts/redo%E6%B5%81%E7%A8%8B.png)

### WAL Write-Ahead Log

预写日志机制，当事务提交时，先将 redo log buffer 写入到 redo log file 进行持久化，待事务的commit操作完成时才算完成。至于change buffer的数据，则定时刷新。如果宕机了，则靠redo log来恢复。

## Undo log

作用：用于实现MVCC乐观锁机制、回滚没有commit的事务。

原理：

* 记录和sql语句反向操作的语句，在事务回滚时，执行这些操作就可以实现事务回滚。
* 通过读取undo log日志，读取历史版本的数据，就是快照读。

undo log是采用分段(segment)的方式进行存储的。rollback segment称为回滚段，每个回滚段中有1024个undo log segment。在MySQL5.5之前，只支持1个rollback segment，也就是只能记录1024个undo操作。在MySQL5.5之后，可以支持128个rollback segment，分别从resg slot0 - resg slot127，每一个resg slot，也就是每一个回滚段，内部由1024个undo segment 组成，即总共可以记录128 * 1024个undo操作。

在更新数据之前，MySQL会提前生成undo log日志，当事务提交的时候，并不会立即删除undo log，因为后面可能需要进行回滚操作，要执行回滚（rollback）操作时，从缓存中读取数据。undo log日志的删除是通过通过后台purge线程进行回收处理的。

### Undo log的写入时机

* DML操作修改聚簇索引前，记录undo日志
* 二级索引记录的修改，不记录undo日志

需要注意的是，undo页面的修改，同样需要记录redo日志。

### Undo的类型

* insert undo log
* update undo log

insert undo log是指在insert 操作中产生的undo log，因为insert操作的记录，只对事务本身可见，对其他事务不可见。故该undo log可以在事务提交后直接删除，不需要进行purge操作。

而update undo log记录的是对delete 和update操作产生的undo log，该undo log可能需要提供MVCC机制，因此不能再事务提交时就进行删除。提交时放入undo log链表，等待purge线程进行最后的删除。

### 回滚段

InnoDB在undo tablespace中使用回滚段来组织undo log。同时为了保证事务的并发操作，在写undo log时不产生冲突，InnoDB使用 回滚段 来维护undo log的并发写入和持久化；而每个回滚段 又有多个undo log slot。通常通过Rollback Segment Header来管理回滚段，Rollback Segment Header通常在回滚段的第一个页

Rollback Segment Header里面最重要的两部分就是history list与undo slot directory。

其中history list把所有已经提交但还没有被purge事务的undo log串联起来，purge线程可以通过此list对没有事务使用的undo log进行purge。

### UndoPage

undo page一般分两种情况：header page和normal page。
header page除了normal page所包含的信息，还包含一些undo segment信息

## binlog

在引擎层的日志，用于主从同步

为了保持redo log和binlog的一致性，使用两阶段提交。

保证 redo log 和 binlog 的操作记录一致的流程是，将操作先更新到内存，再写入 redo log，此时标记为 prepare 状态，再写入 binlog，此时再提交事务，将 redo log 标记为 commit 状态。

### redolog 和 binlog

* redo log 是InnoDB 引擎特有的；而 binlog 是MySQL Server 层实现的
* redo log 是物理日志，记录的是“在某个数据页做了什么修改”；而 binlog 是逻辑日志，记录的是语句的原始逻辑。比如update T set c=c+1 where ID=2;这条SQL，redo log 中记录的是 ：xx页号，xx偏移量的数据修改为xxx；binlog 中记录的是：id = 2 这一行的 c 字段 +1
* redo log 是循环写的，固定空间会用完；binlog 可以追加写入，一个文件写满了会切换到下一个文件写，并不会覆盖之前的记录
* 记录内容时间不同，redo log 记录事务发起后的 DML 和 DDL语句；binlog 记录commit 完成后的 DML 语句和 DDL 语句
* 作用不同，redo log 作为异常宕机或者介质故障后的数据恢复使用；binlog 作为恢复数据使用，主从复制搭建。
