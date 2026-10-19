# InnoDB引擎

## 逻辑存储结构

![Snipaste_2026-10-13_08-44-44](..\images\Snipaste_2026-10-13_08-44-44.png)

![Snipaste_2026-10-18_17-03-47](..\images\Snipaste_2026-10-18_17-03-47.png)

## 架构

MySQL5.5版本后，默认使用InnoDB存储引擎，它擅长事务处理，具有崩溃恢复特性，在日常开发中使用非常广泛。下面是InnoDB架构图，左侧为内存结构，右侧为磁盘结构

![Snipaste_2026-10-19_09-19-48](..\images\Snipaste_2026-10-19_09-19-48.png)

### 内存架构

![Snipaste_2026-10-19_09-20-51](..\images\Snipaste_2026-10-19_09-20-51.png)

**Buffer pool**：缓冲池是主内存中的一个区域，里面可以缓存磁盘上经常操作的真实数据，在执行增删改查操作时，先操作缓冲池中的数据（如果缓冲池 没有数据，则从磁盘加载并缓存），然后再以一定频率刷新到磁盘，减少磁盘IO，加快处理速度

缓冲池以Page页为单位，底层采用链表数据结构管理Page。根据状态，将Page分成三种类型：

- free page：空闲page，未被使用
- clean page：被使用page，数据没有被修改过
- dirty page：脏页，被使用page，数据被修改过，页中的数据与磁盘中的数据产生了不一致

**Change Buffer**：更改缓冲区（针对于非唯一 二级索引页），在执行DML语句时，如果这些数据Page没有在Buffer Pool中，不会直接操作磁盘，而会将数据变更存在更改缓冲区Change Buffer中，在未来数据被读取时，再将数据合并恢复到Buffer Pool中，再讲合并后的数据刷新到磁盘中。

**Q：Change Buffer的意义是什么？**

与聚集索引不同，二级索引通常是非唯一的，而且以相对随机的顺序插入二级索引。同样，删除和更新可能会影响索引树中不相邻的二级索引页，如果每一次都操作硬盘，会造成大量磁盘IO。有了ChangeBUffer后，我们可以在缓冲池中进行合并操作，减少磁盘IO

**Adaptive Hash Index**：自适应hash索引，用于优化对Buffer Pool数据的查询。InnoDB存储引擎会监控对表上各索引页的查询，如果观察到hash索引可以提升速度，则建立hash索引，称之为自适应hash索引

**自适应哈希索引，无需人工干涉，是系统根据情况自动完成的**

参数：adaptive_hash_index

**Log Buffer**：日志缓冲区，用来保存要写入磁盘中的log日志数据（redo log、undo log），默认大小为16MB，日志缓冲区的日志会定期刷新到磁盘中。如果需要更新、插入或 删除许多行的事务，增加日志缓冲区的大小可以节省磁盘IO。

参数：innodb_log_buffer_size：缓冲区大小

​			innodb_flush_log_at_trx_commit：日志刷新到磁盘时机

​			innodb_flush_log_at_trx_commit有三种取值，1：日志在每次事务提交时写入并刷新到磁盘，0：每秒将日志接入并刷新到磁盘一次，2：日志在每次事务提交后写入，并且每秒刷新到磁盘一次

### 磁盘结构

![Snipaste_2026-10-19_10-40-19](..\images\Snipaste_2026-10-19_10-40-19.png)

**System Tablespace**：系统表空间是更改缓冲区的存储引擎。如果表是在系统表空间而不是某个标文件或通用表空间中创建的，它也可能包含表和索引数据（在MySQL5.x版本汇总还包含InnoDB数据字D典、undolog等）

参数：innodb_data_file_path

**File-Per-Tablespaces**：每个表的文件表空间包含单个InnoDB表的数据和索引，并存储在文件系统上的单个数据文件里

参数：innodb_file_per_table

```sql
#创建通用表空间
CREATE TABLESPACE 通用表空间名 ADD DATAFILE '文件名' ENGINE = engine_name;
#让某表格关联于这个通用表空间
CREATE TABLE 表名 TABLESPACE 通用表空间名;
```

**Undo Tablespaces**：撤销表空间，MySQL实例在初始化时会自动创建两个默认的undo表空间（初始大小为16M），用于存储undo log日志

**Temporary Tablespaces**：InnoDB使用会话临时表空间和全局临时表空间，存储用户创建的临时表等数据

**Doublewrite Buffer Files**：双写缓冲区，InnoDB引擎将数据页从Buffer Pool刷新到磁盘前，先将数据页写入双写缓冲区文件中，便于系统异常时恢复数据

```
#磁盘里两个文件名
'#ib_16384_0.dblwr'
'#ib_16384_1.dblwr'
```

**Redo Log**：重做日志，是用来实现事务的持久性。该日志文件由两部分组成：重做日志缓冲（redo log buffer）以及重做日志文件（redo log），前者是在内存中，后者在磁盘中。当事务提交后会把所有修改信息都存到该日志中，用于在刷新脏页到磁盘时，发生错误时，进行数据恢复使用

以循环形式接入重做日志文件

```
#涉及的两个文件
'ib_logfile0'
'ib_logfile1'
```

### 后台线程

1. **Master Thread**

核心后台线程，负责调度其他线程，还负责将缓冲池中的数据异步刷新到磁盘中，保持数据的一致性，还包括脏页的刷新、合并插入缓存、undo页的回收。

2. **IO Thread**

在InnoDB存储引擎中大量使用了AIO来处理IO请求，这样可以极大地提高数据库的性能，而IO Thread主要负责这些IO请求的回调。

| 线程类型             | 默认个数 | 职责                         |
| :------------------- | :------- | :--------------------------- |
| Read thread          | 4        | 负责读操作                   |
| Write thread         | 4        | 负责写操作                   |
| Log thread           | 1        | 负责将日志缓冲区刷新到磁盘   |
| Insert buffer thread | 1        | 负责将写缓冲区内容刷新到磁盘 |

3. **Purge Thread**

主要用于回收事务已经提交了的undo log，在事务提交之后，undo log可能不用了，就用它来回收。

4. **Page Cleaner Thread**

协助 Master Thread 刷新脏页到磁盘的线程，它可以减轻 Master Thread 的工作压力，减少阻塞。

![Snipaste_2026-10-19_11-19-01](..\images\Snipaste_2026-10-19_11-19-01.png)

## 事务原理

事务是一组操作的集合，是一个不可分割的工作单位。事务会把所有的操作作为一个整体一起向系统提交或撤销操作请求，也就是说，这些操作**要么同时成功，要么同时失败**

- 特性

  - 原子性（Atomicity）

    事务是不可分割的最小操作单元，要么全部成功，要么全部失败

    联想到——`Java`的`Synchronized`

  - 一致性（Consistency）

    事务完成时，必须使所有的数据都保持一致

  - 隔离性（Isolation）

    数据库系统提供的隔离机制，保证事务在不受外部并发操作影响的独立环境下运行

  - 持久性（Durability）

    事务一旦提交或回滚，那么它对与数据库中数据的改变就是永久的

**redo log**

重做日志，记录事务提交时数据页的物理修改，用来实现事务的持久性

该日志文件由两部分组成：重做日志缓冲（redo log buffer）以及重做日志文件（redo log file），前者是在内存中，后者是在磁盘中。当事务提交后会把所有修改信息都存到该日志文件红，用于在刷新脏页到磁盘，发生错误时，进行数据恢复使用

**undo log**

回滚日志，用于记录数据被修改前的信息，作用有两个：提供回滚和MVCC（多版本并发控制）

undo log和redo log记录物理日志不一样，它是逻辑日志。可以认为当delete一条记录时，undo log会记录一条对应的insert记录，反之亦然，当update一条记录时，它记录一条相反的update记录。当执行rollback时，就可以从undo log中的逻辑记录读取到相应的内容并进行回滚

undo log销毁：undo log在事务执行时产生，事务提交时，并不会立即删除undo  log ，因为这些日志还可能用于MVCC

undo log存储：undo log采用段的方式进行管理和记录，存放在前面介绍的rollback segment回滚段中，内部包含1024个undo log segment

## MVCC

### 基本概念

- 当前读

  读取的是记录的最新版本，读取时还要保证其他并发事务不能修改当前记录，会对读取的记录进行加锁。对于我们日常的操作，如：select ... lock in share mode(共享锁)，select ... for update,update,insert ,delete(排他锁)都是一种当前读

- 快照读

  简单的SELECT（不加锁）就是一快照读，快照读，读取的是记录数据的可见版本，有可能是历史数据，不加锁，是非阻塞读

  - Read Committed：每次select，都生成一个快照读
  - Repeatable Read：开启事务后第一个select语句才是快照读的地方
  - Serializable：快照读会退化为当前读

- MVCC

  全称Multi-Version Concurrency Control，多版本并发控制。指维护一个数据的多个版本，是的读写操作没有冲突，快照读为MySQL实现MVCC提供了一个非阻塞读功能。MVCC的具体实现，还需要依赖于数据库记录中的三个隐式字段、undo log日志、readView

### 隐藏字段

**数据表示例：**

| id   | age  | name |
| :--- | :--- | :--- |
| 1    | 1    | tom  |
| 3    | 3    | cat  |

**InnoDB 底层实际存储的行记录结构（包含隐藏字段）：**

> 除了用户定义的数据列（`id`, `age`, `name`）外，实际存储时还会附带隐藏字段：
> **[ id | age | name | DB_TRX_ID | DB_ROLL_PTR ]**
> *(注：如果表没有指定主键，还会在最前面增加 `DB_ROW_ID`)*

**隐藏字段含义：**

| 隐藏字段        | 含义                                                         |
| :-------------- | :----------------------------------------------------------- |
| **DB_TRX_ID**   | 最近修改事务ID，记录插入这条记录或最后一次修改该记录的事务ID。 |
| **DB_ROLL_PTR** | 回滚指针，指向这条记录的上一个版本，用于配合undo log，指向上一个版本。 |
| **DB_ROW_ID**   | 隐藏主键，如果表结构没有指定主键，将会生成该隐藏字段。       |

### undo log

回滚日志，在insert、update、delete的时候产生的便于数据回滚的日志

在insert的时候，产生的undo log日志只是在回滚时需要，在事务提交后可以被立即删除

而update、delete时，产生的undo log日志不仅在回滚时需要，在快照读时也需要，不会立即删除

**undo log版本链**

![Snipaste_2026-10-19_13-37-25](..\images\Snipaste_2026-10-19_13-37-25.png)

不同事务或者相同事务对同一条记录进行修改，会导致该记录的undo log生成一条记录版本链表，链表的头部是最新的旧记录，链表的尾部是最早的旧记录

### readview

Readview（读视图）是快照读SQL执行时MVCC提取数据的依据，记录并维护系统当前活跃的事务（未提交的）id

readview包含了四个核心字段

| 字段               | 含义                                                 |
| :----------------- | :--------------------------------------------------- |
| **m_ids**          | 当前活跃的事务ID集合                                 |
| **min_trx_id**     | 最小活跃事务ID                                       |
| **max_trx_id**     | 预分配事务ID，当前最大事务ID+1（因为事务ID是自增的） |
| **creator_trx_id** | ReadView创建者的事务ID                               |

**版本连数据访问规则**

trx_id：当前事务的id

1. trx_id == creator_trx_id? 说明数据是当前这个事务更改的，可以访问该版本
2. trx_id < min_trx_id? 说明数据已经提交了，可以访问该版本
3. trx_id > max_trx_id? 说明该事物是在ReadView生成后才开启，不可以访问该版本
4. min_trx_id <= trx_id <= max_trx_id 如果trx_id不在m_ids中，说明数据已经提交，是可以访问该版本的

**不同的隔离级别，生成ReadView的时机也不同**

- READ COMMITTED ：事务中每一次执行快照读时生成ReadView
- REPEATABLE READ ： 仅在事务中第一次执行快照读时生成ReadView，后续复用该ReadView