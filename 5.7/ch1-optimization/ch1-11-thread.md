# 检查线程信息

您可以查询`performance_schema`数据库中的表，以查看有关服务器及其运行的应用程序的性能特征的实时信息。

## 11.1 检查线程信息

当您尝试确定MySQL服务器正在执行的操作时，检查进程列表会很有帮助，该列表是当前在服务器中执行的线程集。可从以下来源获取流程列表信息：

- `SHOW [FULL] PROCESSLIST`语句

- `SHOW PROFILE`语句

- `INFORMATION_SCHEMA PROCESSLIST`表

- `mysqladmin processlist`命令

- Performance Schema`thread`表，阶段表和锁定表

- `sys` schema `processlist`视图，以更易于访问的格式显示Performance Schema `threads`表中的信息 

- `sys` schema `session`视图，显示有关用户会话的信息（如sys架构进程列表视图，但后台进程已过滤掉）

对`threads`的访问不需要互斥锁，对服务器性能的影响最小。 `INFORMATION_SCHEMA.PROCESSLIST`和`SHOW PROCESSLIST`会产生负面的性能影响，因为它们需要互斥锁。线程还显示有关后台线程的信息，`INFORMATION_SCHEMA.PROCESSLIST`和`SHOW PROCESSLIST`不会。这意味着线程可用于监视其他线程信息源无法进行的活动。

您始终可以查看有关自己的线程的信息。要查看有关正在为其他帐户执行的线程的信息，您必须具有`PROCESS`权限。

每个进程列表条目包含几条信息：

• Id是与线程关联的客户端的连接标识符。

• `User`和`Host`指示与线程关联的帐户。

• `db`是线程的默认数据库，如果没有选择，则为`NULL`。

• `Command` 和 `State`指示线程正在做什么。

大多数状态对应于非常快速的操作。如果一个线程在给定状态下停留很多秒，则可能存在需要调查的问题。

• 时间表示线程处于当前状态的时间。线程的当前概念在某些情况下，时间可能会改变：线程可以使用`SET TIMESTAMP = value`更改时间。对于正在处理来自主站的事件的从站上运行的线程，线程时间设置为在事件中找到的时间，因此反映了主站而不是从站的当前时间。

• `Info`包含线程正在执行的语句的文本，如果没有执行，则为`NULL`。默认情况下，此值仅包含语句的前`100`个字符。要查看完整的语句，请使用`SHOW FULL PROCESSLIST`。

以下部分列出了可能的`Command`值以及按类别分组的`State`值。其中一些价值观的含义是不言而喻的。对于其他人，提供了额外的描述。

## 11.2 线程命令值

线程可以具有以下任何Command值：

• `Binlog Dump`

这是主服务器上的一个线程，用于将二进制日志内容发送到从属服务器。

• `Change user`
线程正在执行更改用户操作。

• `Close stmt`
线程正在关闭一个准备好的语句。

• `Connect`
复制从站连接到其主站。(A replication slave is connected to its master.)

• `Connect Out`
复制从站正在连接到其主站。(A replication slave is connecting to its master.)

• `Create DB`
这线程执行创建数据库操作。

• `Daemon`
此线程是服务器的内部线程，而不是为客户端连接提供服务的线程。

• `Debug`
此线程生成调试信息

• `Delayed insert`
该线程是一个延迟插入处理程序。

• `Drop DB`
此线程执行drop-database操作。

• `Error`
• `Execute`
线程正在执行预准备语句。

• `Fetch`
该线程正在从执行预准备语句中获取结果。

• `Field List`
该线程正在检索表列的信息。

• `Init DB`
该线程正在选择默认数据库。

• `Kill`
该线程正在杀死另一个线程。

• `Long Data`
线程正在检索执行预准备语句的结果中的长数据。

• `Ping`
线程正在检索执行预准备语句的结果中的长数据。

• `Prepare`
该主题正在准备一份准备好的声明。

• `Processlist`
该线程正在生成有关服务器线程的信息。

• `Query`
线程正在执行一个语句。

• `Quit`
线程正在终止。

• `Refresh`
该线程正在刷新表，日志或缓存，或重置状态变量或复制服务器信息。

• `Register Slave`
线程正在注册从服务器。

• `Reset stmt`
该线程正在重置预准备语句。

• `Set option`
该线程正在设置或重置客户端语句执行选项。

• `Shutdown`
该线程正在关闭服务器。

• `Sleep`
线程正在等待客户端向其发送新语句。

• `Statistics`
该线程正在生成服务器状态信息。

• Table Dump
线程正在将表内容发送到从属服务器。

• Time
Unused.


## 11.3 通用线程状态

以下列表描述了与常规查询处理相关联的线程状态值，而不是更复杂的特殊活动。其中许多仅用于查找服务器中的错误。

• After create
当线程在创建表的函数末尾创建表（包括内部临时表）时，会发生这种情况。即使由于某些错误而无法创建表，也会使用此状态。

• Analyzing
该线程正在计算MyISAM表键分布（例如，对于`ANALYZE TABLE`）。

• checking permissions
线程正在检查服务器是否具有执行语句所需的权限。

• Checking table
该线程正在执行表检查操作。

• cleaning up
该线程已经处理了一个命令，并准备释放内存并重置某些状态变量。

• closing tables
该线程正在将更改的表数据刷新到磁盘并关闭已使用的表。这应该是一个快速的操作。如果没有，请验证您没有完整磁盘并且磁盘使用不是很大。

• converting HEAP to ondisk
该线程正在将内部临时表从MEMORY表转换为磁盘表。

• copy to tmp table
该线程正在处理ALTER TABLE语句。在创建具有新结构的表但在将行复制到其中之前，将发生此状态。对于处于此状态的线程，可以使用性能模式来获取有关复制操作的进度。

• Copying to group table
如果语句具有不同的ORDER BY和GROUP BY条件，则按组对行进行排序并将其复制到临时表。

• Copying to tmp table
服务器正在复制到内存中的临时表。

• altering table
服务器正在执行就地ALTER TABLE。

• Copying to tmp table on disk
服务器正在复制到磁盘上的临时表。临时结果集变得太大。因此，线程正在将临时表从内存更改为基于磁盘的格式以节省内存。

• Creating index
该线程正在为MyISAM表处理ALTER TABLE ... ENABLE KEYS。

• Creating sort index
该线程正在处理使用内部临时表解析的SELECT。

• creating table
线程正在创建一个表。这包括创建临时表。

• Creating tmp table
该线程正在内存或磁盘上创建临时表。如果表在内存中创建但稍后转换为磁盘表，则该操作期间的状态将是复制到磁盘上的tmp表。

• committing alter table to storage engine
服务器已完成就地ALTER TABLE并提交结果。

• deleting from main table
服务器正在执行多表删除的第一部分。它仅从第一个表中删除，并保存用于从其他（引用）表中删除的列和偏移量。

• deleting from reference tables
服务器正在执行多表删除的第二部分，并从其他表中删除匹配的行。

• discard_or_import_tablespace
该线程正在处理ALTER TABLE ... DISCARD TABLESPACE或ALTER TABLE ... IMPORT TABLESPACE语句。

• end
这发生在结束但在清除ALTER TABLE，CREATE VIEW，DELETE，INSERT，SELECT或UPDATE语句之前。

• executing
该线程已开始执行语句。

• Execution of init_command
该线程正在init_command系统变量的值中执行语句。

• freeing items
线程执行了一个命令。在此状态期间完成的一些项目的释放涉及查询缓存。这种状态通常随后进行清理。

• FULLTEXT initialization
服务器正准备执行自然语言full-text搜索。

• init
这发生在ALTER TABLE，DELETE，INSERT，SELECT或UPDATE语句的初始化之前。服务器在此状态下采取的操作包括刷新二进制日志，InnoDB日志和一些查询缓存清理操作。

对于最终状态，可能会发生以下操作：
1. 删除表中的数据后删除查询缓存条目
2. 将事件写入二进制日志
3. 释放内存缓冲区，包括blob
4. Killed

有人向线程发送了一个KILL语句，它应该在下次检查kill标志时中止。在MySQL的每个主循环中都会检查标志，但在某些情况下，线程可能还需要很短的时间才能死掉。如果线程被某个其他线程锁定，则一旦另一个线程释放其锁定，kill就会生效。

• logging slow query
该线程正在向慢查询日志写一条语句。

• login
连接线程的初始状态，直到客户端成功通过身份验证。

• manage keys
服务器正在启用或禁用表索引。

• NULL
此状态用于SHOW PROCESSLIST状态。

• Opening tables
线程正在尝试打开一个表。这应该是非常快的程序，除非有什么东西阻止打开。例如，ALTER TABLE或LOCK TABLE语句可以阻止在语句完成之前打开表。还需要检查table_open_cache值是否足够大。

• optimizing
服务器正在对查询执行初始优化。

• preparing
在查询优化期间发生此状态。

• Purging old relay logs
该线程正在删除不需要的中继日志文件。

• query end
处理查询后但在释放项状态之前发生此状态。

















































































































































































































































































































































