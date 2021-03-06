# 优化锁定操作( Optimizing Locking Operations )

MySQL使用锁管理表内容的争用:

- 内部锁定在MySQL服务器内部执行，以管理多个线程对表内容的争用。这种类型的锁定是内部的，因为它完全由服务器执行，不涉及其他程序。

- 外部锁定发生在服务器和其他程序锁定MyISAM表文件以协调它们之间的关系时，哪个程序可以在什么时候访问表。

 ## 8.1 内部锁操作

本节讨论内部锁定;也就是说，在MySQL服务器内部执行锁定来管理多个会话对表内容的争用。这种类型的锁定是内部的，因为它完全由服务器执行，不涉及其他程序。


### 8.1.1 行级锁


MySQL使用`InnoDB`表的行级锁来支持多个会话的同时写访问，使它们适合多用户、高并发和OLTP应用程序。

为了避免在单个`InnoDB`表上执行多个并发写操作时出现死锁，请通过发出`SELECT ... FOR UPDATE`在事务开始时获取必要的锁。对于每个预期要修改的行组的语句，即使数据更改语句在事务中稍后出现。 如果事务修改或锁定多个表，则在每个事务中以相同的顺序发出适用的语句。 死锁会影响性能而不是表示严重错误，因为`InnoDB`会自动检测死锁条件并回滚其中一个受影响的事务。

在高并发系统上，当多个线程等待同一个锁时，死锁检测可能会导致速度下降。有时，禁用死锁检测并依赖`innodb_lock_wait_timeout`设置在死锁发生时执行事务回滚可能更有效。可以使用`innodb_deadlock_detection`配置选项禁用死锁检测。

行级锁的优点:

• 当不同的会话访问不同的行时，锁冲突更少。

• 回滚的更改更少。

• 可以长时间锁定一行。

### 8.1.2 表级锁

MySQL对`MyISAM`，`MEMORY`和`MERGE`表使用表级锁定，一次只允许一个会话更新这些表。 此锁定级别使这些存储引擎更适合于只读，大多数读取或单用户应用程序。

这些存储引擎总是在查询开始时一次性请求所有需要的锁，并且总是以相同的顺序锁定表，从而避免死锁。权衡是这种策略降低了并发性;希望修改表的其他会话必须等到当前数据更改语句完成。

表级锁定的优点:

- 所需内存相对较少(行锁定需要锁定每一行或每组行的内存)

- 在表的大部分区域使用时速度快，因为只涉及一个锁。

- 如果您经常对大部分数据进行分组操作，或者必须经常扫描整个表，那么速度会很快。


MySQL授予表写锁如下:

1. 如果表上没有锁，则在其上放置一个写锁。

2. 否则，将锁定请求放入写锁定队列。

MySQL授予表读锁如下:

1. 如果表上没有写锁，则在其上放一个读锁。

2. 否则，将锁请求放入读锁队列中


表更新的优先级高于表检索。 因此，当释放锁定时，锁定对写入锁定队列中的请求可用，然后对读取锁定队列中的请求可用。 这确保即使表的`SELEC`T活动很多，对表的更新也不会“缺乏”。 但是，如果表有很多更新，`SELECT`语句将一直等到没有更新。

您可以通过检查`Table_locks_immediate`和`table_locks_waiting`状态变量来分析系统上的表锁争用，这些状态变量分别表示可以立即授予表锁请求的次数和必须等待的次数:

```mysql
mysql> SHOW STATUS LIKE 'Table%';
+-----------------------+---------+
| Variable_name         | Value   |
+-----------------------+---------+
| Table_locks_immediate | 1151552 |
| Table_locks_waited    | 15324   |
+-----------------------+---------+
```

`MyISAM`存储引擎支持并发插入以减少给定表的读取器和写入器之间的争用：如果`MyISAM`表在数据文件的中间没有空闲块，则总是在数据文件的末尾插入行。 在这种情况下，您可以自由地为没有锁的`MyISAM`表混合并发`INSERT`和`SELECT`语句。 也就是说，您可以在其他客户端从中读取行的同时将行插入到`MyISAM`表中。 在表格中间删除或更新的行可能会导致漏洞。 如果存在漏洞，则会禁用并发插入，但在所有漏洞都填充新数据后会自动再次启用。 要控制此行为，请使用`concurrent_insert`系统变量。

如果使用`LOCK TABLES`显式获取表锁，则可以请求`READ LOCAL`锁而不是`READ`锁，以使其他会话在锁定表时执行并发插入。

要在不能进行并发插入时对表`t1`执行许多`INSERT`和`SELECT`操作，可以将行插入临时表`temp_t1`并使用临时表中的行更新实际表：

```mysql
mysql> LOCK TABLES t1 WRITE, temp_t1 WRITE;
mysql> INSERT INTO t1 SELECT * FROM temp_t1;
mysql> DELETE FROM temp_t1;
mysql> UNLOCK TABLES;
```

### 8.1.3 选择锁类型( Choosing the Type of Locking )

一般来说，在以下情况下，表锁优于行锁:

- 表的大多数语句都是读语句。

- 表的语句是读写的混合，其中写入是单行的更新或删除，可以使用一个键读取：

```mysql
UPDATE tbl_name SET column=value WHERE unique_key_col=key_value;
DELETE FROM tbl_name WHERE unique_key_col=key_value;
```

- `SELECT`与并发插入语句结合使用，很少有`UPDATE`或`DELETE`语句。

- 对整个表进行多次扫描或`GROUP BY`操作，无需任何写入程序。

使用更高级别的锁，您可以通过支持不同类型的锁来更容易地调优应用程序，因为锁开销比行级锁小。

行级锁定之外的其他选项:

- 版本控制(如在`MySQL`中用于并发插入的版本控制)，在这种情况下，可以同时拥有一个`writer`和多个`readers`。这意味着数据库或表支持不同的数据视图，具体取决于访问开始的时间。其他常用术语有`“time travel”`、`“copy on write”`或`“copy on demand”`

- 在许多情况下，按需复制优于行级锁定。然而，在最坏的情况下，它可以比使用普通锁使用更多的内存。

- 您可以使用应用程序级锁，而不是使用行级锁，例如MySQL中的`GET_LOCK()`和`RELEASE_LOCK()`提供的锁。 这些是咨询锁，因此它们仅适用于彼此协作的应用程序。

## 8.2 表锁问题

`InnoDB`表使用行级锁定，以便多个会话和应用程序可以同时读取和写入同一个表，而不会使彼此等待或产生不一致的结果。对于此存储引擎，请避免使用`LOCK TABLES`语句，因为它不提供任何额外保护，而是降低并发性。自动行级锁定使这些表适用于最繁忙的数据库，包含最重要的数据，同时还简化了应用程序逻辑，因为您不需要锁定和解锁表。因此，`InnoDB`存储引擎是MySQL的默认设置。

MySQL对除`InnoDB`之外的所有存储引擎使用表锁定（而不是页锁，行或列锁定）。锁定操作本身没有太多开销。但是因为在任何时候只有一个会话可以写入表，为了获得这些其他存储引擎的最佳性能，主要将它们用于经常查询但很少插入或更新的表。


### 8.2.1 有利于InnoDB的性能考虑

在选择是使用InnoDB还是使用其他存储引擎创建表时，请记住表锁定的以下缺点：

• 表锁定允许多个会话同时从表中读取，但如果会话要写入表，则必须首先获得独占访问权，这意味着它可能必须等待其他会话首先完成表。在更新期间，要访问此特定表的所有其他会话必须等到更新完成。

• 表会在会话等待时导致问题，因为磁盘已满并且在会话可以继续之前需要可用空间。在这种情况下，所有想要访问问题表的会话也会处于等待状态，直到有更多磁盘空间可用。

• 需要很长时间才能运行的`SELECT`语句会阻止其他会话在此期间更新表，从而使其他会话显得缓慢或无响应。当会话等待获得对表的独占访问以进行更新时，发出`SELECT`语句的其他会话将在其后排队，即使对于只读会话也会降低并发性。

### 8.2.2 锁定性能问题的变通方法
          
以下各项描述了避免或减少表锁定引起的争用的一些方法：

• 考虑将表切换到InnoDB存储引擎，在安装过程中使用`CREATE TABLE ... ENGINE = INNODB`，或者对现有表使用`ALTER TABLE ... ENGINE = INNODB`。

• 优化`SELECT`语句以更快地运行，以便它们可以在更短的时间内锁定表。您可能必须创建一些汇总表来执行此操作。

• 使用`--low-priority-updates`启动`mysqld`。对于仅使用表级锁定的存储引擎（例如`MyISAM`，`MEMORY`和`MERGE`），这使得所有更新（修改）表的语句的优先级低于`SELECT`语句。在这种情况下，前一个场景中的第二个`SELECT`语句将在`UPDATE`语句之前执行，并且不会等待第一个`SELECT`完成。

• 要指定在特定连接中发出的所有更新都应以低优先级完成，请将`low_priority_updates`服务器系统变量设置为`1`。

• 要使特定的`INSERT`，`UPDATE`或`DELETE`语句具有较低的优先级，请使用`LOW_PRIORITY`属性。

• 要为特定的`SELECT`语句赋予更高的优先级，请使用`HIGH_PRIORITY`属性。

• 使用`max_write_lock_count`系统变量的低值启动`mysqld`，以强制MySQL临时提升在对表进行特定数量的插入后等待表的所有`SELECT`语句的优先级。这允许在一定数量的WRITE锁之后进行READ锁定。

• 如果`INSERT`与`SELECT`结合存在问题，请考虑切换到支持并发`SELECT`和`INSERT`语句的`MyISAM`表。

• 如果混合`SELECT`和`DELETE`语句有问题，`DELETE`的`LIMIT`选项可能会有所帮助。
 
• 将`SQL_BUFFER_RESULT`与`SELECT`语句一起使用可以帮助缩短表锁的持续时间。

• 通过允许查询针对一个表中的列运行，将表内容拆分为单独的表可能会有所帮助，而更新仅限于不同表中的列。

• 您可以更改`mysys/thr_lock.c`中的锁定代码以使用单个队列。 在这种情况下，写锁和读锁具有相同的优先级，这可能有助于某些应用程序。

## 8.3 并发插入

`MyISAM`存储引擎支持并发插入，以减少给定表的readers和writers之间的争用：如果`MyISAM`表在数据文件中没有漏洞（中间删除的行），则可以执行`INSERT`语句以向末尾添加行 表的同时`SELECT`语句正在从表中读取行。 如果有多个`INSERT`语句，则它们将排队并与`SELECT`语句同时执行。 并发`INSERT`的结果可能不会立即显示。

`concurrent_insert`系统变量可以设置为修改并发插入处理。默认情况下，该变量设置为`AUTO`（或1），并且如前所述处理并发插入。如果`concurrent_insert`设置为`NEVER`（或0），禁用并发插入。 如果变量设置为`ALWAYS`（或2），则即使对于已删除行的表，也允许在表末尾进行并发插入。 另请参见`concurrent_insert`系统变量的说明。

如果使用二进制日志，则并发插入将转换为`CREATE ... SELECT`或`INSERT ... SELECT`语句的常规插入。 这样做是为了确保您可以通过在备份操作期间应用日志来重新创建表的精确副本。 请参见第5.4.4节“二进制日志”。此外，对于这些语句，读取锁定位于selected-from表中，以便阻止对该表的插入。 结果是该表的并发插入也必须等待。

使用`LOAD DATA`，如果使用满足并发插入条件的MyISAM表指定`CONCURRENT`（即，它不包含中间的空闲块），则其他会话可以在`LOAD DATA`执行时从表中检索数据。 即使没有其他会话同时使用该表，使用`CONCURRENT`选项也会影响`LOAD DATA`的性能。

如果指定`HIGH_PRIORITY`，则在使用该选项启动服务器时，它将覆盖`--low-priority-updates`选项的效果。 它还会导致不使用并发插入。

对于`LOCK TABLE`，`READ LOCAL`和`READ`之间的区别在于`READ LOCAL`允许在保持锁定时执行非冲突的`INSERT`语句（并发插入）。 但是，如果要在保持锁定时使用服务器外部的进程操作数据库，则无法使用此功能。

## 8.4 元数据锁

MySQL使用元数据锁定来管理对数据库对象的并发访问，并确保数据一致性。元数据锁定不仅适用于表，还适用于模式和存储程序(过程、函数、触发器和计划的事件)和表空间。

性能模式`metadata_locks`表公开了元数据锁信息，这对于查看哪些会话持有锁、哪些会话被阻塞以等待锁等等非常有用。

元数据锁定确实涉及一些开销，这些开销会随着查询量的增加而增加。多个查询试图访问相同对象的次数越多，元数据争用就越多。

元数据锁定不是表定义缓存的替代，它的互斥锁和锁与`LOCK_open`互斥锁不同。下面的讨论提供了关于元数据锁定如何工作的一些信息。

### 8.4.1 元数据锁获取

如果一个给定的锁有多个等待器，那么优先级最高的锁请求首先得到满足，`max_write_lock_count`系统变量除外。写锁请求的优先级高于读锁请求。但是，如果`max_write_lock_count`被设置为某个较低的值(例如，如果读锁请求已经被10个写锁请求传递，则读锁请求可能优先于挂起的写锁请求。通常不会发生这种行为，因为`max_write_lock_count`默认值非常大。

语句逐个获取元数据锁，而不是同时获取，并在进程中执行死锁检测。

DML语句通常按照语句中提到的表的顺序获取锁。

DDL语句，`LOCK TABLES`和其他类似语句尝试通过按名称顺序获取显式命名表上的锁来减少并发DDL语句之间可能出现的死锁数。 对于隐式使用的表（例如也必须锁定的外键关系中的表），可以以不同的顺序获取锁。

例如，`RENAME TABLE`是一个DDL语句，它按名称顺序获取锁:

- 这个`RENAME TABLE`语句将`tbla`重命名为其他名称，并将`tblc`重命名为`tbla`:

```mysql
RENAME TABLE tbla TO tbld, tblc TO tbla;
```

语句按顺序获取`tbla`、`tblc`和`tbld`上的元数据锁(因为tbld按名称顺序跟随tblc):

- 这个稍微不同的语句也将`tbla`重命名为其他名称，并将`tblc`重命名为`tbla`：

```mysql
RENAME TABLE tbla TO tblb, tblc TO tbla;
```

在这种情况下，语句按顺序获取`tbla`、`tblb`和`tblc`上的元数据锁(因为`tblb`在名称顺序上先于`tblc`):


这两个语句都按这个顺序获取`tbla`和`tblc`上的锁，但是对于其余表名上的锁是在`tblc`之前还是之后获得的，情况有所不同。

当多个事务同时执行时，元数据锁获取顺序会影响操作结果，如下面的示例所示

从具有相同结构的两个表`x`和`x_new`开始。三个客户端发出涉及这些表的语句:

Client 1:

```mysql
LOCK TABLE x WRITE, x_new WRITE;
```

语句在`x`和`x_new`上请求并获得按名称顺序排列的写锁。

Client 2:

```mysql
INSERT INTO x VALUES(1);
```

语句请求并阻塞等待`x`上的写锁。

Client 3:

```mysql
RENAME TABLE x TO x_old, x_new TO x;
```

该语句按`x`、`x_new`和`x_old`上的名称顺序请求独占锁，但阻塞等待`x`上的锁。

Client 1:

```mysql
UNLOCK TABLES;
```

该语句释放`x`和`x_new`上的写锁。 Client 3对x的独占锁定请求具有比Client 2的写入锁定请求更高的优先级，因此Client 3获取其对x的锁定，然后还在`x_new`和`x_old`上执行重命名，并释放其锁定。 然后，Client 2获取其对`x`的锁定，执行插入并释放其锁定。

锁定获取顺序导致在`INSERT`之前执行`RENAME TABLE`。 插入发生的`x`是当Client 2发出插入并由Client 3重命名为`x`时名为`x_new`的表：

```mysql
mysql> SELECT * FROM x;
+------+
| i    |
+------+
| 1    |
+------+

mysql> SELECT * FROM x_old;
Empty set (0.01 sec)
```

现在从具有相同结构的名为`x`和`new_x`的表开始。同样，三个客户端发出涉及这些表的语句:

Client 1

```mysql
LOCK TABLE x WRITE, new_x WRITE;
```

该语句在`new_x`和`x`上请求并获得按名称顺序排列的写锁。

Client 2

```mysql
INSERT INTO x VALUES(1);
```

语句请求并阻塞等待`x`上的写锁。

Client 3

```mysql
RENAME TABLE x TO old_x, new_x TO x;
```

该语句在`new_x`、`old_x`和`x`上按名称顺序请求独占锁，但在`new_x`上阻塞等待锁。

Client 1

```mysql
UNLOCK TABLES;
```

该语句释放`x`和`new_x`上的写锁。 对于`x`，唯一的待处理请求是Client 2，因此Client 2获取其锁定，执行插入并释放锁定。 对于`new_x`，唯一的待处理请求是Client 3，允许获取该锁（以及`old_x`上的锁）。 重命名操作仍会阻止`x`上的锁定，直到Client 2插入完成并释放其锁定。 然后，Client 3获取`x`上的锁定，执行重命名，并释放其锁定。

在这种情况下，锁定获取顺序导致`INSERT`在`RENAME TABLE`之前执行。 插入发生的`x`是原始`x`，现在通过重命名操作重命名为`old_x`：

```mysql
mysql> SELECT * FROM x;
Empty set (0.01 sec)

mysql> SELECT * FROM old_x;
+------+
| i    |
+------+
| 1    |
+------+
```

如果并发语句中的锁获取顺序对操作结果中的应用程序有影响(如上例所示)，则可以调整表名以影响锁获取的顺序。

### 8.4.2 元数据锁释放

为了确保事务的可串行性，服务器必须不允许一个会话对另一个会话中未完成的显式或隐式启动的事务中使用的表执行数据定义语言(DDL)语句。服务器通过在事务中使用的表上获取元数据锁，并将这些锁的释放推迟到事务结束时，从而实现这一点。表上的元数据锁可以防止对表结构的更改。这种锁定方法意味着一个会话内的事务正在使用的表不能在DDL中使用。

这个原则不仅适用于事务性表，也适用于非事务性表。假设一个会话开始一个事务，该事务使用事务表`t`和非事务表`nt`，如下所示:

```mysql
START TRANSACTION;
SELECT * FROM t;
SELECT * FROM nt;
```

服务器持有`t`和`nt`上的元数据锁，直到事务结束。如果另一个会话尝试对这两个表执行DDL或write lock操作，它将阻塞，直到事务结束时元数据锁释放。例如，如果第二个会话尝试以下任何操作，它就会阻塞:

```mysql
DROP TABLE t;
ALTER TABLE t ...;
DROP TABLE nt;
ALTER TABLE nt ...;
LOCK TABLE t ... WRITE;
```

同样的行为适用于`LOCK TABLES ... READ`。 也就是说，更新任何表（事务性或非事务性）的显式或隐式启动事务将阻塞并被`LOCK TABLES ...`阻止该表的READ。


如果服务器为语法上有效但在执行过程中失败的语句获取元数据锁，则不会提前释放锁。锁释放仍然延迟到事务的末尾，因为失败的语句被写入二进制日志，锁保护日志一致性。

在自动提交模式下，每个语句实际上都是一个完整的事务，因此为语句获取的元数据锁只保留到语句末尾。

在准备语句后，即使在多语句事务中进行准备，也会释放`PREPARE`语句期间获取的元数据锁。

## 8.5 外部锁

外部锁定是使用文件系统锁定来管理多个进程对`MyISAM`数据库表的争用。 外部锁定用于不能假定单个进程（如MySQL服务器）是唯一需要访问表的进程的情况。 这里有些例子：

• 如果运行多个使用相同数据库目录的服务器（不推荐），则每个服务器必须启用外部锁定。

• 如果使用`myisamchk`对`MyISAM`表执行表维护操作，则必须确保服务器未运行，或者服务器已启用外部锁定，以便根据需要锁定表文件以与`myisamchk`协调以访问表。 使用`myisampack`打包`MyISAM`表也是如此。

如果服务器在启用外部锁定的情况下运行，则可以随时使用`myisamchk`进行读取操作，例如检查表。 在这种情况下，如果服务器尝试更新`myisamchk`正在使用的表，服务器将等待`myisamchk`完成后再继续。

如果使用`myisamchk`进行写操作（例如修复或优化表），或者使用`myisampack`打包表，则必须始终确保`mysqld`服务器不使用该表。 如果你没有停止`mysqld`，至少在你运行`myisamchk`之前做一个`mysqladmin flush-tables`。 如果服务器和`myisamchk`同时访问表，您的表可能会损坏。


在外部锁定生效的情况下，需要访问表的每个进程都会在继续访问表之前获取表文件的文件系统锁。 如果无法获取所有必需的锁，则阻止进程访问表，直到可以获取锁（在当前持有锁的进程释放它们之后）。

外部锁定会影响服务器性能，因为服务器有时必须等待其他进程才能访问表。

如果运行单个服务器来访问给定的数据目录（通常情况下），并且在服务器运行时没有其他程序（如`myisamchk`）需要修改表，则不需要外部锁定。 如果您只读取其他程序的表，则不需要外部锁定，尽管如果服务器在`myisamchk`读取时更改了表，`myisamchk`可能会报告警告。

禁用外部锁定后，要使用`myisamchk`，必须在执行`myisamchk`时停止服务器，或者在运行`myisamchk`之前锁定并刷新表。 （请参见第8.12.1节“系统因素”。）为避免此要求，请使用`CHECK TABLE`和`REPAIR TABLE`语句检查和修复`MyISAM`表。

对于`mysqld`，外部锁定由`skip_external_locking`系统变量的值控制。 启用此变量时，将禁用外部锁定，反之亦然。 默认情况下禁用外部锁定。

可以使用`--external-locking`或`--skip-external-locking`选项在服务器启动时控制外部锁定的使用。

如果使用外部锁定选项从许多MySQL进程启用对`MyISAM`表的更新，则必须确保满足以下条件：

• 不要将查询缓存用于使用由其他进程更新的表的查询。

• 不要使用`--delay-key-write = ALL`选项启动服务器，也不要对任何共享表使用`DELAY_KEY_WRITE = 1`表选项。 否则，可能会发生索引损坏。


满足这些条件的最简单方法是始终使用`--external-locking`与`--delay-key-write = OFF`和`--query-cache-size = 0`。 （默认情况下不会这样做，因为在许多设置中，混合使用前面的选项很有用。）














