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

服务器正在执行就地`ALTER TABLE`。

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

• Receiving from client

服务器正在从客户端读取数据包。这种状态在MySQL 5.7.8之前被称为从网络读取。

• Removing duplicates

该查询使用SELECT DISTINCT，以至于MySQL无法在早期阶段优化掉不同的操作。因此，在将结果发送到客户端之前，MySQL需要额外的阶段来删除所有重复的行。

• removing tmp table

该线程在处理SELECT语句后删除内部临时表。如果未创建临时表，则不使用此状态。

• rename

该线程正在重命名一个表。

• rename result table

该线程正在处理ALTER TABLE语句，创建了新表，并重命名它以替换原始表。

• Reopen tables

该线程获得了表的锁定，但在获取锁定之后注意到基础表结构发生了变化。它释放了锁，关闭了表，并试图重新打开它。

• Repair by sorting

修复代码使用排序来创建索引。

• preparing for alter table

服务器正准备执行就地ALTER TABLE。

• Repair done

该线程已完成MyISAM表的多线程修复。

• Repair with keycache

修复代码通过密钥缓存逐个创建密钥。这比通过排序修复要慢得多。

• Rolling back

该线程正在回滚一个事务。

• Saving state

对于`MyISAM`表操作（例如修复或分析），线程将新表状态保存到`.MYI`文件头。 `State`包括诸如行数，`AUTO_INCREMENT`计数器和密钥分发之类的信息。

• Searching rows for update

线程正在执行第一个阶段，在更新所有匹配的行之前查找它们。如果更新正在更改用于查找相关行的索引，则必须这样做。

• Sending data
线程正在读取和处理SELECT语句的行，并向客户机发送数据。由于在此状态期间发生的操作往往执行大量的磁盘访问(读取)，因此它通常是给定查询的生命周期中运行时间最长的状态。

• Sending to client

服务器正在向客户端写入数据包。 在MySQL 5.7.8之前，这种状态称为`“Writing to net”`。

• setup

该线程正在开始ALTER TABLE操作。

• Sorting for group
该线程正在进行排序以满足GROUP BY。

• Sorting for order
线程正在进行排序以满足ORDER BY。

• Sorting index
线程正在对索引页进行排序，以便在MyISAM表优化操作期间更有效地进行访问。

• Sorting result
对于SELECT语句，这类似于`Creating sort index`，但对于非临时表。

• statistics
服务器正在计算统计数据以开发查询执行计划。如果一个线程长时间处于这种状态，则服务器可能正在执行磁盘绑定的其他工作。

• System lock
该线程已调用mysql_lock_tables()，并且线程状态自此未更新。这是一个非常普遍的状态，可能由于多种原因而发生。

例如，线程将请求或正在等待表的内部或外部系统锁定。 当`InnoDB`在执行`LOCK TABLES`期间等待表级锁定时，会发生这种情况。如果此状态是由外部锁请求引起的，并且您没有使用多个访问相同`MyISAM`表的`mysqld`服务器，则可以禁用外部系统 使用`--skip-external-locking`选项锁定。 但是，默认情况下禁用外部锁定，因此该选项很可能无效。 对于`SHOW PROFILE`，此状态表示线程正在请求锁定（不等待它）。

• update
线程正准备开始更新表。

• Updating
线程正在搜索要更新的行并正在更新它们。

• updating main table
服务器正在执行多表更新的第一部分。 它仅更新第一个表，并保存用于更新其他（引用）表的列和偏移量。

• updating reference tables
服务器正在执行多表更新的第二部分，并更新其他表中的匹配行。

• User lock
线程将请求或正在等待通过`GET_LOCK()`调用请求的咨询锁。对于`SHOW PROFILE`，此状态表示线程正在请求锁定（不等待它）。

• User sleep
线程调用了SLEEP()调用。(The thread has invoked a SLEEP() call.)

• Waiting for commit lock
带有`READ LOCK`的`FLUSH TABLES`正在等待提交锁定。

• Waiting for global read lock
带有`READ LOCK`的`FLUSH TABLES`正在等待全局读锁定或正在设置全局read_only系统变量。

• Waiting for tables
线程得到一个通知，表明表的底层结构已经改变，它需要重新打开表以获得新结构。 但是，要重新打开表，它必须等到所有其他线程关闭了相关表。 如果另一个线程在相关表上使用了`FLUSH TABLES`或以下语句之一，则会发出此通知：`FLUSH TABLES tbl_name`，`ALTER TABLE`，`RENAME TABLE`，`REPAIR TABLE`，`ANALYZE TABLE`或`OPTIMIZE TABLE`。线程得到一个通知，表明表的底层结构已经改变，它需要重新打开表以获得新结构。 但是，要重新打开表，它必须等到所有其他线程关闭了相关表。 如果另一个线程在相关表上使用了`FLUSH TABLES`或以下语句之一，则会发出此通知：`FLUSH TABLES tbl_name`，`ALTER TABLE`，`RENAME TABLE`，`REPAIR TABLE`，`ANALYZE TABLE`或`OPTIMIZE TABLE`。
• Waiting for table flush
线程正在执行`FLUSH TABLES`并等待所有线程关闭它们的表，或者线程得到一个表的基础结构已经改变的通知，它需要重新打开表以获得新结构。 但是，要重新打开表，它必须等到所有其他线程都关闭了有问题的表。如果另一个线程在相关表上使用了`FLUSH TABLES`或下列语句之一，则会发生此通知：`FLUSH TABLES tbl_name`，`ALTER TABLE` ，`RENAME TABLE`，`REPAIR TABLE`，`ANALYZE TABLE`或`OPTIMIZE TABLE`。
• Waiting for lock_type lock
服务器正在等待从元数据锁定子系统获取`THR_LOCK`锁或锁，其中`lock_type`指示锁的类型。
此状态表示等待`THR_LOCK`：
• Waiting for table level lock

这些状态表示等待元数据锁定：

• 等待事件元数据锁定
• 等待全局读锁定
• 等待架构元数据锁定
• 等待存储的功能元数据锁定
• 等待存储过程元数据锁定
• 等待表元数据锁定
• 等待触发器元数据锁定

• Waiting on cond
线程等待条件变为真的一般状态。没有具体的状态信息。
• Writing to net
服务器正在将数据包写入网络。 从MySQL 5.7.8开始，此状态称为发送到客户端

## 11.4 查询缓存的线程状态

这些线程状态与查询缓存相关联

• checking privileges on cached query
服务器正在检查用户是否具有访问缓存查询结果的权限。

• checking query cache for query
服务器正在检查查询缓存中是否存在当前查询。

• invalidating query cache entries
查询缓存条目被标记为无效，因为基础表已更改。

• sending cached result to client
服务器从查询缓存中获取查询结果并将其发送到客户端。

• storing result in query cache
服务器将查询结果存储在查询缓存中。

• Waiting for query cache lock
在会话等待获取查询缓存锁定时发生此状态。 对于需要执行某些查询缓存操作的任何语句，例如使查询缓存无效的`INSERT`或`DELETE`，查找缓存条目的`SELECT`，`RESET QUERY CACHE`等，都会发生这种情况。

## 11.5 复制主线程状态

以下列表显示了在从属服务器I/O线程的State列中看到的最常见状态。 此状态也出现在`SHOW SLAVE STATUS`显示的`Slave_IO_State`列中，因此您可以通过使用该语句很好地了解正在发生的情况。
• Checking master version

• Connecting to master
线程正在尝试连接到主服务器。

• Queueing master event to the relay log
线程已读取事件并将其复制到中继日志，以便SQL线程可以处理它。

• Reconnecting after a failed binlog dump request
线程正在尝试重新连接到主服务器。

• Reconnecting after a failed master event read
线程正在尝试重新连接到主服务器。 当再次建立连接时，状态变为等待主节点发送事件。

• Registering slave on master
建立与主站连接后非常短暂的状态。

• Requesting binlog dump
在建立与主站的连接之后非常短暂地发生的状态。 线程从请求的二进制日志文件名和位置开始向主机发送对其二进制日志内容的请求。

• Waiting for its turn to commit
如果启用了`slave_preserve_commit_order`，则当从属线程正在等待较旧的工作线程提交时发生的状态。
• Waiting for master to send event
线程已连接到主服务器并正在等待二进制日志事件到达。 如果主站空闲，这可能会持续很长时间。 如果等待持续slave_net_timeout秒，则发生超时。 此时，线程认为连接被破坏并尝试重新连接。

• Waiting for master update
连接到master之前的初始状态。

• Waiting for slave mutex on exit
线程停止时短暂发生的状态。

• Waiting for the slave SQL thread to free enough relay log space
您正在使用非零的relay_log_space_limit值，并且中继日志已经增长到足以使其组合大小超过此值。 I / O线程正在等待，直到SQL线程通过处理中继日志内容释放足够的空间，以便它可以删除一些中继日志文件。

• Waiting to reconnect after a failed binlog dump request
如果二进制日志转储请求失败（由于断开连接），则线程在休眠时进入此状态，然后尝试定期重新连接。 可以使用`CHANGE MASTER TO`语句指定重试之间的间隔。
• Waiting to reconnect after a failed master event read
读取时发生错误（由于断开连接）。 在尝试重新连接之前，线程正在休眠由`CHANGE MASTER TO`语句（默认为`60`）设置的秒数。


## 11.6 复制从SQL线程状态

以下列表显示了您可能在从属服务器SQL线程的State列中看到的最常见状态：

• Killing slave
该线程正在处理`STOP SLAVE`语句。
• Making temporary file (append) before replaying LOAD DATA INFILE
线程正在执行LOAD DATA语句，并将数据附加到包含从属将从中读取行的数据的临时文件。

• Making temporary file (create) before replaying LOAD DATA INFILE
该线程正在执行`LOAD DATA`语句，并且正在创建一个临时文件，其中包含从属将从中读取行的数据。只有在运行MySQL版本低于MySQL 5.0.3的主服务器记录原始`LOAD DATA`语句时才会遇到此状态。

• Reading event from the relay log
线程已从中继日志中读取事件，以便可以处理事件。

• Slave has read all relay log; waiting for more updates
该线程已处理中继日志文件中的所有事件，现在正在等待I / O线程将新事件写入中继日志。

• Waiting for an event from Coordinator
使用多线程从站（`slave_parallel_workers`大于`1`），其中一个从属工作线程正在等待来自协调器线程的事件。

• Waiting for slave mutex on exit
线程停止时发生的非常短暂的状态。

• Waiting for Slave Workers to free pending events
当Workers正在处理的事件的总大小超过`slave_pending_jobs_size_max`系统变量的大小时，会发生此等待操作。当大小低于此限制时，协调器将恢复计划。仅当`slave_parallel_workers`设置为大于`0`时才会出现此状态。

• Waiting for the next event in relay log
从中继日志中读取事件之前的初始状态。

• Waiting until MASTER_DELAY seconds after master executed event
SQL线程已读取事件，但正在等待从属延迟失效。使用`CHANGE MASTER TO`的MASTER_DELAY选项设置此延迟。
SQL线程的`Info`列也可能显示语句的文本。这表明线程已从中继日志中读取事件，从中提取语句，并可能正在执行它。

## 11.7 复制从属连接线程状态

这些线程状态出现在复制从属服务器上，但与连接线程相关联，而不是与I / O或SQL线程相关联。

• Changing master
该线程正在处理`CHANGE MASTER TO`语句。

• Killing slave
该线程正在处理`STOP SLAVE`语句。

• Opening master dump table
This state occurs after `Creating table from master dump`.

• Reading master dump table data
This state occurs after `Opening master dump table`.

• Rebuilding the index on master dump table
This state occurs after `Reading master dump table data`.

## 11.8  NDB Cluster Thread States

..........

##11.9 事件调度程序线程状态

这些状态发生在`Event Scheduler`线程，为执行调度事件而创建的线程或终止调度程序的线程中。

• Clearing
调度程序线程或正在执行事件的线程正在终止并即将结束。

• Initialized
调度程序线程或将执行事件的线程已初始化。

• Waiting for next activation
调度程序具有非空事件队列，但下一次激活是将来的。

• Waiting for scheduler to stop
线程发出`SET GLOBAL event_scheduler = OFF`并等待调度程序停止。

• Waiting on empty queue
调度程序的事件队列为空并且正在休眠。






























































































































































































































































































































