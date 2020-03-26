# 优化MySQL服务器


本节讨论数据库服务器的优化技术，主要处理系统配置而不是调优SQL语句。本节中的信息适用于希望确保所管理服务器的性能和可伸缩性的DBA;用于构建包括设置数据库的安装脚本的开发人员;和人们自己运行MySQL进行开发，测试等等，他们希望最大限度地提高自己的工作效率。

## 9.1 系统因素

某些系统级因素可能会以一种主要方式影响性能：
• 如果您有足够的RAM，则可以删除所有交换设备。即使您有空闲内存，某些操作系统也会在某些情况下使用交换设备。
• 避免`MyISAM`表的外部锁定。默认设置是禁用外部锁定。`--external-locking`和`--skip-external-locking`选项显式启用和禁用外部锁定。

只要您只运行一台服务器，禁用外部锁定不会影响MySQL的功能。在运行`myisamchk`之前，请记住取下服务器（或锁定并刷新相关表）。在某些系统上，必须禁用外部锁定，因为它无论如何都不起作用。

您无法禁用外部锁定的唯一情况是，您在同一数据上运行多个MySQL服务器（而非客户端），或者运行`myisamchk`以检查（不修复）表而不告诉服务器先刷新并锁定表。请注意，除了使用NDB Cluster之外，通常不建议使用多个MySQL服务器同时访问相同的数据。

`LOCK TABLES`和`UNLOCK TABLES`语句使用内部锁定，因此您可以使用它们即使禁用外部锁定。


## 9.2 优化磁盘 I/O

本节介绍在您可以投入更多和更快速度时配置存储设备的方法存储硬件到数据库服务器。

• 磁盘搜索是一个巨大的性能瓶颈。当数据量开始增长到无法实现有效缓存时，这个问题就变得更加明显了。对于您或多或少随机访问数据的大型数据库，您可以确保至少需要一个磁盘读取和一些磁盘寻求写入内容。要最小化此问题，请使用寻道时间较短的磁盘。
• 通过将文件符号链接到不同磁盘或条带化磁盘来增加可用磁盘轴数（从而减少搜索开销）：
• 使用符号链接

这意味着，对于MyISAM表，您将索引文件和数据文件从它们在数据目录中的通常位置符号链接到另一个磁盘（也可能是条带化的）。这使得搜索和读取时间都更好，假设磁盘也不用于其他目的。

InnoDB表不支持符号链接。但是，可以将InnoDB数据和日志文件放在不同的物理磁盘上。

• Striping

条带化意味着您有许多磁盘并将第一个块放在第一个磁盘上，第二个块放在第二个磁盘上，第N个块放在（`N MOD number_of_disks`）磁盘上，依此类推。这意味着如果您的正常数据大小小于条带大小（或完全对齐），您将获得更好的性能。条带化非常依赖于操作系统和条带大小，因此使用不同的条带大小对应用程序进行基准测试。

条带化的速度差异很大程度上取决于参数。根据您设置条带化参数和磁盘数量的方式，您可能会获得以数量级计量的差异。您必须选择优化随机或顺序访问。

• 为了可靠性，您可能需要使用RAID 0 + 1（条带加镜像<striping plus mirroring>），但在这种情况下，您需要2×N驱动器可容纳N个数据驱动器。如果您有钱，这可能是最好的选择。但是，您可能还需要投资一些卷管理软件来有效地处理它。
• 一个好的选择是根据数据类型的重要程度来改变RAID级别。例如，存储可在RAID 0磁盘上重新生成的半重要数据，但在RAID 0 + 1或RAID N磁盘上存储非常重要的数据，如主机信息和日志。如果您有多次写入，由于更新奇偶校验位所需的时间，RAID N可能会出现问题。
• 您还可以设置数据库使用的文件系统的参数：如果您不需要知道上次访问文件的时间（这在数据库服务器上没有用），您可以使用`-o noatime`挂载文件系统选项。这会跳过文件系统上inode中最后一次访问时间的更新，从而避免了一些磁盘搜索。
在许多操作系统上，您可以通过使用-o async选项挂载来设置要异步更新的文件系统。如果您的计算机相当稳定，这应该可以在不牺牲太多可靠性的情况下提供更好的性能。 （默认情况下，此标志在Linux上处于启用状态。）

## 9.3 使用 NFS 和 MySQL

在考虑将NFS与MySQL一起使用时，建议小心。潜在问题因操作系统和NFS版本而异，包括：

- 放置在NFS卷上的MySQL数据和日志文件将被锁定且无法使用。例如，由于断电导致MySQL的多个实例访问同一数据目录或MySQL被不正确地关闭的情况，可能会发生锁定问题。 NFS版本4通过引入咨询和基于租约的锁定来解决潜在的锁定问题。但是，不建议在MySQL实例之间共享数据目录。

- 由于接收到的消息顺序错误或丢失了网络流量而导致的数据不一致。为了避免这个问题，使用带有`hard`和`intr`挂载选项的TCP。

- 最大文件大小限制。 NFS版本2客户端只能访问文件的最低2GB（带符号的32位偏移）。 NFS版本3客户端支持更大的文件（最多64位偏移）。支持的最大文件大小还取决于NFS服务器的本地文件系统。

在专业SAN环境或其他存储系统中使用NFS往往比在此类环境之外使用NFS提供更高的可靠性。但是，SAN环境中的NFS可能比直接连接或总线连接的非旋转存储更慢。

如果选择使用NFS，则建议使用NFS版本4或更高版本，以及在部署到生产环境之前彻底测试NFS设置。

## 9.4 使用符号链接

您可以将数据库或表从数据库目录移动到其他位置，并使用指向新位置的符号链接替换它们。您可能希望这样做，例如，将数据库移动到具有更多可用空间的文件系统，或者通过将表扩展到不同的磁盘来提高系统的速度。

对于`InnoDB`表，请使用`CREATE TABLE`语句中的`DATA DIRECTORY`子句而不是符号链接。此新功能是一种受支持的跨平台技术。

建议的方法是将整个数据库目录符号链接到其他磁盘。 Symlink `MyISAM`表仅作为最后的手段。

要确定数据目录的位置，请使用以下语句：

```mysql
SHOW VARIABLES LIKE 'datadir';
```

### 9.4.1 在Unix上使用数据库的符号链接

在Unix上，符号链接数据库的方法是首先在某个磁盘上创建一个目录，在该磁盘上有可用空间，然后从MySQL数据目录创建一个到它的软链接。

```bash
shell> mkdir /dr1/databases/test
shell> ln -s /dr1/databases/test /path/to/datadir
```

MySQL不支持将一个目录链接到多个数据库。只要不在数据库之间建立符号链接，就可以使用符号链接替换数据库目录。假设您在MySQL数据目录下有一个数据库`db1`，然后创建一个指向`db1`的符号链接`db2`：

```bash
shell> cd /path/to/datadir
shell> ln -s db1 db2
```

结果是，对于`db1`中的任何表`tbl_a`，`db2`中似乎还有一个表`tbl_a`。如果一个客户端更新`db1.tbl_a`而另一个客户端更新`db2.tbl_a`，则可能会出现问题。

### 9.4.2 在Unix上使用MyISAM表的符号链接

只有MyISAM表才支持符号链接。对于表用于其他存储引擎的文件，如果尝试使用符号链接，可能会遇到奇怪的问题。(对于InnoDB表，请使用第14.6.3.6节“在数据目录外创建表空间”中说明的替代技术。)

不要在没有完全可操作的`realpath()`调用的系统上进行符号链接表。(Linux和Solaris支持`realpath()`)。要确定您的系统是否支持符号链接，请使用以下语句检查`has_symlink`系统变量的值：

```mysql
SHOW VARIABLES LIKE 'have_symlink';
```

处理MyISAM表的符号链接的工作方式如下：

- 在数据目录中，始终具有表格式（`.frm`）文件，数据（`.MYD`）文件和索引（`.MYI`）文件。数据文件和索引文件可以移动到其他位置，并通过符号链接在数据目录中替换。格式文件不能。

- 您可以将数据文件和索引文件独立地符号链接到不同的目录。

- 要指示正在运行的MySQL服务器执行符号链接，请使用DATA DIRECTORY和INDEX DIRECTORY选项来创建表。或者，如果mysqld未运行，则可以使用命令行中的ln -s手动完成符号链接。

`注意`
与DATA DIRECTORY和INDEX DIRECTORY选项之一或两者一起使用的路径可能不包括MySQL数据目录。 （Bug 32167）
`注意`

- `myisamchk`不会用数据文件或索引文件替换符号链接。它直接在符号链接指向的文件上工作。在数据文件或索引文件所在的目录中创建任何临时文件。 `ALTER TABLE`，`OPTIMIZE TABLE`和`REPAIR TABLE`语句也是如此。

`注意`
删除使用符号链接的表时，符号链接和符号链接指向的文件都将被删除。这是一个非常好的理由，不能将`mysqld`作为`root`操作系统用户运行，或允许操作系统用户具有对MySQL数据库目录的写访问权。
`注意`

- 如果使用`ALTER TABLE ... RENAME`或`RENAME TABLE`重命名表，并且不将表移动到另一个数据库，则数据库目录中的符号链接将重命名为新名称，并相应地重命名数据文件和索引文件。

- 如果使用`ALTER TABLE ... RENAME`或`RENAME TABLE`将表移动到另一个数据库，则表将移动到另一个数据库目录。如果表名更改，则新数据库目录中的符号链接将重命名为新名称，并相应地重命名数据文件和索引文件。

- 如果您没有使用符号链接，请使用`--skip-symbolic-links`选项启动`mysqld`，以确保没有人可以使用`mysqld`删除或重命名数据目录之外的文件。

不支持这些表符号链接操作：

`ALTER TABLE`忽略`DATA DIRECTORY`和`INDEX DIRECTORY`表选项。

如前所述，只有数据和索引文件可以是符号链接。 `.frm`文件绝不能是符号链接。尝试执行此操作（例如，使一个表名称成为另一个表名称的同义词）会产生不正确的结果。假设您在MySQL下有一个数据库`db1`数据目录，此数据库中的表tbl1，以及在db1目录中创建指向tbl1的符号链接tbl2：

```bash
shell> cd / path / to / datadir / db1
shell> ln -s tbl1.frm tbl2.frm
shell> ln -s tbl1.MYD tbl2.MYD
shell> ln -s tbl1.MYI tbl2.MYI
```

如果一个线程读取`db1.tbl1`而另一个线程更新`db1.tbl2`，则会出现问题：

- 查询缓存被“欺骗”（它无法知道tbl1尚未更新，所以它返回过时的结果）。

- `tbl2`上的`ALTER`语句失败。

### 9.4.3 在Windows上使用数据库的符号链接

..........

## 9.5 优化内存使用

### 9.5.1 MySQL如何使用内存

MySQL分配缓冲区和缓存以提高数据库操作的性能。默认配置旨在允许MySQL服务器在具有大约512MB RAM的虚拟机上启动。您可以通过增加某些缓存和缓冲区相关系统变量的值来提高MySQL性能。您还可以修改默认配置以在内存有限的系统上运行MySQL。

以下列表描述了MySQL使用内存的一些方法。适用时，引用相关的系统变量。某些项目是存储引擎或特定功能。

`InnoDB`缓冲池是一个内存区域，用于保存表，索引和其他辅助缓冲区的缓存`InnoDB`数据。为了提高大容量读取操作的效率，缓冲池被分成可以容纳多行的页面。为了提高缓存管理的效率，缓冲池被实现为链接的页面列表;使用LRU算法的变体，很少使用的数据在缓存中老化。

缓冲池的大小对系统性能很重要：

- 1 `InnoDB`使用`malloc()`操作在服务器启动时为整个缓冲池分配内存.`innodb_buffer_pool_size`系统变量定义缓冲池大小。通常，推荐的`innodb_buffer_pool_size`值是系统内存的50％到75％。在服务器运行时，可以动态配置`innodb_buffer_pool_size`。

- 2 在具有大量内存的系统上，可以通过将缓冲池划分为多个缓冲池实例来提高并发性。 `innodb_buffer_pool_instances`系统变量定义缓冲池实例的数量。

- 3 太小的缓冲池可能会导致过度搅动，因为从缓冲池中刷新页面后不久就需要再次刷新。

- 4 过大的缓冲池可能会因内存竞争而导致交换。

- 所有线程共享MyISAM密钥缓冲区。 key_buffer_size系统变量确定其大小。

对于服务器打开的每个MyISAM表，索引文件打开一次;为访问该表的每个并发运行的线程打开一次数据文件。对于每个并发线程，分配表结构，每列的列结构和大小为3 * N的缓冲区（其中N是最大行长度，不计算BLOB列）。 BLOB列需要五到八个字节加上BLOB数据的长度。 MyISAM存储引擎维护一个额外的行缓冲区供内部使用。

• 可以将`myisam_use_mmap`系统变量设置为1以启用所有`MyISAM`表的内存映射。

• 如果内部内存临时表变得太大（使用`tmp_table_size`和`max_heap_table_size`系统变量确定），MySQL会自动将表从内存转换为磁盘格式。磁盘上临时表使用`internal_tmp_disk_storage_engine`系统变量定义的存储引擎。

对于使用`CREATE TABLE`显式创建的`MEMORY`表，只有`max_heap_table_size`系统变量决定了表的增长程度，并且没有转换为磁盘格式。

• MySQL性能模式是一种用于监视低级别MySQL服务器执行的功能。性能模式以递增方式动态分配内存，将其内存使用量扩展到实际服务器负载，而不是在服务器启动期间分配所需的内存。分配内存后，在重新启动服务器之前不会释放内存。

• 服务器用于管理客户端连接的每个线程都需要一些特定于线程的空间。以下列表指出了这些以及哪些系统变量控制它们的大小：

- 1. 堆栈（thread_stack）
- 1. 连接缓冲区（net_buffer_length）
- 1. 结果缓冲区（net_buffer_length）

连接缓冲区和结果缓冲区的大小都等于`net_buffer_length`字节，但根据需要动态扩大到`max_allowed_pa​​cket`字节。每个SQL语句后，结果缓冲区缩小为`net_buffer_length`个字节。在语句运行时，还会分配当前语句字符串的副本。

每个连接线程使用内存来计算语句摘要。服务器分配每个会话的`max_digest_length`个字节。

• 所有线程共享相同的基本内存。

• 当不再需要某个线程时，分配给它的内存将被释放并返回给系统，除非该线程返回到线程缓存中。在这种情况下，内存仍然是分配的。

• 每个执行表顺序扫描的请求都会分配一个读缓冲区。 `read_buffer_size`系统变量确定缓冲区大小。

• 以任意顺序读取行时（例如，排序后），可以分配随机读取缓冲区以避免磁盘搜索。 `read_rnd_buffer_size`系统变量确定缓冲区大小。

• 所有连接都在一次执行中执行，大多数连接都可以在不使用临时表的情况下完成。大多数临时表都是基于内存的哈希表。具有大行长度（计算为所有列长度的总和）或包含`BLOB`列的临时表存储在磁盘上。

• 执行排序的大多数请求根据结果集大小分配排序缓冲区和零到两个临时文件。

• 几乎所有解析和计算都在线程本地和可重用内存池中完成。小项目不需要内存开销，因此避免了正常的慢速内存分配和释放。内存仅分配给意外大的字符串。

• 对于具有BLOB列的每个表，动态放大缓冲区以读取更大的`BLOB`值。如果扫描表，缓冲区会增大到最大`BLOB`值。

• MySQL需要表缓存的内存和描述符。所有正在使用的表的处理程序结构都保存在表缓存中，并作为“先进先出”（FIFO）进行管理。 `table_open_cache`系统变量定义初始表缓存大小; MySQL还需要内存用于表定义缓存。 `table_definition_cache`系统变量定义可以存储在表定义高速缓存中的表定义（来自`.frm`文件）的数量。如果使用大量表，则可以创建大型表定义高速缓存以加快表的打开速度。与表缓存不同，表定义缓存占用的空间更少，不使用文件描述符。

• `FLUSH TABLES`语句或`mysqladmin flush-tables`命令一次性关闭所有未使用的表，并在当前正在执行的线程完成时标记要关闭的所有正在使用的表。这有效地释放了大多数使用中的内存。在所有桌子关闭之前，`FLUSH TABLES`不会返回。

•作为`GRANT`，`CREATE USER`，`CREATE SERVER`和`INSTALL PLUGIN`语句的结果，服务器将信息缓存在内存中。相应的`REVOKE`，`DROP USER`，`DROP SERVER`和`UNINSTALL PLUGIN`语句不会释放此内存，因此对于执行导致缓存的语句的许多实例的服务器，内存使用会增加。可以使用`FLUSH PRIVILEGES`释放此缓存的内存。


`ps`和其他系统状态程序可能会报告`mysqld`使用大量内存。这可能是由不同内存地址上的线程堆栈引起的。例如，Solaris版本的`ps`将堆栈之间未使用的内存计为已用内存。要验证这一点，请使用`swap -s`检查可用交换。我们使用几个内存泄漏检测器（商业和开源）测试`mysqld`，因此应该没有内存泄漏。

#### 9.5.1.1 监听MySQL内存使用

以下示例演示如何使用Performance Schema和sys schema 进行监视MySQL内存使用情况。

默认情况下禁用大多数Performance Schema内存检测。可以通过更新Performance Schema `setup_instruments`表的`ENABLED`列来启用仪器。存储器仪器具有`memory/code_area/instrument_name`形式的名称，其中`code_area`是诸如sql或`innodb`之类的值，`instrument_name`是仪器详细信息。

1. 要查看可用的MySQL内存仪器，请查询Performance Schema `setup_instruments`表。以下查询为所有人返回数百个内存工具代码区域。

```mysql
mysql> SELECT * FROM performance_schema.setup_instruments
WHERE NAME LIKE '%memory%';
```

您可以通过指定代码区域来缩小结果范围。例如，您可以通过将`innodb`指定为代码区域来将结果限制为`InnoDB`内存仪器。

```mysql
mysql> SELECT * FROM performance_schema.setup_instruments
WHERE NAME LIKE '%memory/innodb%';
+-------------------------------------------+---------+-------+
| NAME                                      | ENABLED | TIMED |
+-------------------------------------------+---------+-------+
| memory/innodb/adaptive hash index         | NO      | NO    |
| memory/innodb/buf_buf_pool                | NO      | NO    |
| memory/innodb/dict_stats_bg_recalc_pool_t | NO      | NO    |
| memory/innodb/dict_stats_index_map_t      | NO      | NO    |
| memory/innodb/dict_stats_n_diff_on_level  | NO      | NO    |
| memory/innodb/other                       | NO      | NO    |
| memory/innodb/row_log_buf                 | NO      | NO    |
| memory/innodb/row_merge_sort              | NO      | NO    |
| memory/innodb/std                         | NO      | NO    |
| memory/innodb/trx_sys_t::rw_trx_ids       | NO      | NO    |
...
```
根据您的MySQL安装，代码区域可能包括`performance_schema`, `sql`,`client`, `innodb`, `myisam`, `csv`, `memory`, `blackhole`, `archive`, `partition`等。

2. 要启用内存仪器，请在MySQL配置文件中添加`performance-schema-instrument`规则。例如，要启用所有内存仪器，请将此规则添加到配置文件并重新启动服务器：

```mysql
performance-schema-instrument='memory/%=COUNTED'
```

`注意`
启动时启用内存仪器可确保计算启动时发生的内存分配。
`注意`

重新启动服务器后，Performance Schema `setup_instruments`表的`ENABLED`列应为您启用的内存仪器报告`YES`。对于内存仪器，将忽略`setup_instruments`表中的`TIMED`列，因为内存操作未定时。

```mysql
mysql> SELECT * FROM performance_schema.setup_instruments
WHERE NAME LIKE '%memory/innodb%';
+-------------------------------------------+---------+-------+
| NAME                                      | ENABLED | TIMED |
+-------------------------------------------+---------+-------+
| memory/innodb/adaptive hash index         | NO      | NO    |
| memory/innodb/buf_buf_pool                | NO      | NO    |
| memory/innodb/dict_stats_bg_recalc_pool_t | NO      | NO    |
| memory/innodb/dict_stats_index_map_t      | NO      | NO    |
| memory/innodb/dict_stats_n_diff_on_level  | NO      | NO    |
| memory/innodb/other                       | NO      | NO    |
| memory/innodb/row_log_buf                 | NO      | NO    |
| memory/innodb/row_merge_sort              | NO      | NO    |
| memory/innodb/std                         | NO      | NO    |
| memory/innodb/trx_sys_t::rw_trx_ids       | NO      | NO    |
...
```

查询内存仪器数据。在此示例中，在Performance Schema `memory_summary_global_by_event_name`表中查询内存仪器数据，该表汇总了`EVENT_NAME`的数据。 `EVENT_NAME`是该工具的名称。

以下查询返回`InnoDB`缓冲池的内存数据。

```mysql
mysql> SELECT * FROM performance_schema.memory_summary_global_by_event_name
WHERE EVENT_NAME LIKE 'memory/innodb/buf_buf_pool'\G
                  EVENT_NAME: memory/innodb/buf_buf_pool
                 COUNT_ALLOC: 1
                  COUNT_FREE: 0
   SUM_NUMBER_OF_BYTES_ALLOC: 137428992
    SUM_NUMBER_OF_BYTES_FREE: 0
              LOW_COUNT_USED: 0
          CURRENT_COUNT_USED: 1
             HIGH_COUNT_USED: 1
    LOW_NUMBER_OF_BYTES_USED: 0
CURRENT_NUMBER_OF_BYTES_USED: 137428992
   HIGH_NUMBER_OF_BYTES_USED: 137428992
```

可以使用`sys` schema `memory_global_by_current_bytes`表查询相同的基础数据，该表显示服务器内的当前内存使用情况，按分配类型细分。

```mysql
mysql> SELECT * FROM sys.memory_global_by_current_bytes
WHERE event_name LIKE 'memory/innodb/buf_buf_pool'\G
*************************** 1. row ***************************
       event_name: memory/innodb/buf_buf_pool
    current_count: 1
    current_alloc: 131.06 MiB
current_avg_alloc: 131.06 MiB
       high_count: 1
       high_alloc: 131.06 MiB
   high_avg_alloc: 131.06 MiB
```

此`sys` schema查询按代码区域聚合当前分配的内存（`current_alloc`）：

```mysql
mysql> SELECT SUBSTRING_INDEX(event_name,'/',2) AS
code_area, sys.format_bytes(SUM(current_alloc))
AS current_alloc
FROM sys.x$memory_global_by_current_bytes
GROUP BY SUBSTRING_INDEX(event_name,'/',2)
ORDER BY SUM(current_alloc) DESC;
+---------------------------+---------------+
| code_area                 | current_alloc |
+---------------------------+---------------+
| memory/innodb             | 843.24 MiB    |
| memory/performance_schema | 81.29 MiB     |
| memory/mysys              | 8.20 MiB      |
| memory/sql                | 2.47 MiB      |
| memory/memory             | 174.01 KiB    |
| memory/myisam             | 46.53 KiB     |
| memory/blackhole          | 512 bytes     |
| memory/federated          | 512 bytes     |
| memory/csv                | 512 bytes     |
| memory/vio                | 496 bytes     |
+---------------------------+---------------+
```

### 9.5.2 Enabling Large Page Support

某些硬件/操作系统体系结构支持大于默认值的内存页面（通常为4KB）。此支持的实际实现取决于底层硬件和操作系统。执行大量内存访问的应用程序可能会因为减少了转换后备缓冲区（TLB）丢失而使用大页面来提高性能。

在MySQL中，InnoDB可以使用大页面来为其缓冲池和额外的内存池分配内存。

在MySQL中标准使用大页面试图使用支持的最大大小，最多4MB。在Solaris下，“超大页面”功能允许使用高达256MB的页面。此功能适用于最新的SPARC平台。可以使用`--super-large-pages`或`--skip-super-large-pages`选项启用或禁用它。

MySQL还支持Linux实现大页面支持（在Linux中称为HugeTLB）。

在Linux上可以使用大页面之前，必须启用内核来支持它们，并且必须配置HugeTLB内存池。作为参考，HugeTBL API记录在Linux源的`Documentation/vm/hugetlbpage.txt`文件中。

最近一些系统（如Red Hat Enterprise Linux）的内核似乎默认启用了大页面功能。要检查内核是否为真，请使用以下命令查找包含“huge”的输出行：

```bash
shell> cat /proc/meminfo | grep -i huge
HugePages_Total: 0
HugePages_Free: 0
HugePages_Rsvd: 0
HugePages_Surp: 0
Hugepagesize: 4096 kB
```

非空命令输出表示存在大页面支持，但零值表示没有配置使用的页面。

如果需要重新配置内核以支持大页面，请参阅`hugetlbpage.txt`文件以获取说明。

假设您的Linux内核启用了大页面支持，请使用以下命令将其配置为供MySQL使用。通常，您将这些文件放在系统引导序列期间执行的`rc`文件或等效的启动文件中，以便每次系统启动时执行这些命令。在MySQL服务器启动之前，命令应该在引导序列的早期执行。请务必根据您的系统更改分配编号和组编号。

```bash
# Set the number of pages to be used.
# Each page is normally 2MB, so a value of 20 = 40MB.
# This command actually allocates memory, so this much
# memory must be available.
echo 20 > /proc/sys/vm/nr_hugepages
# Set the group number that is permitted to access this
# memory (102 in this case). The mysql user must be a
# member of this group.
echo 102 > /proc/sys/vm/hugetlb_shm_group
# Increase the amount of shmem permitted per segment
# (12G in this case).
echo 1560281088 > /proc/sys/kernel/shmmax
# Increase total amount of shared memory. The value
# is the number of pages. At 4KB/page, 4194304 = 16GB.
echo 4194304 > /proc/sys/kernel/shmall
```

对于MySQL的使用，通常希望`shmmax`的值接近`shmall`的值。

要验证大页面配置，请再次检查`/proc/meminfo`，如前所述。现在您应该看到一些非零值：

```bash
shell> cat /proc/meminfo | grep -i huge
HugePages_Total: 20
HugePages_Free: 20
HugePages_Rsvd: 0
HugePages_Surp: 0
Hugepagesize: 4096 kB
```

使用`hugetlb_shm_group`的最后一步是为`mysql`用户提供memlock限制的“unlimit”值。这可以通过编辑`/etc/security/limits.conf`或将以下命令添加到`mysqld_safe`脚本来完成：

```bash
ulimit -l unlimited
```

将`ulimit`命令添加到`mysqld_safe`会导致`root`用户在切换到`mysql`用户之前将memlock限制设置为`unlimited`。 （这假设`mysqld_safe`是由`root`启动的。）

默认情况下禁用MySQL中的大页面支持。要启用它，请使用`--large-pages`选项启动服务器。例如，您可以在服务器`my.cnf`文件中使用以下行：

```ini
[mysqld]
large-pages
```

使用此选项，`InnoDB`会自动为其缓冲池和额外的内存池使用大页面。如果`InnoDB`无法执行此操作，则会回退使用传统内存并向错误日志写入警告：`Warning: Using conventional memory pool`

要验证是否正在使用大页面，请再次检查`/proc/meminfo`：

```bash
shell> cat /proc/meminfo | grep -i huge
HugePages_Total: 20
HugePages_Free: 20
HugePages_Rsvd: 2
HugePages_Surp: 0
Hugepagesize: 4096 kB
```

## 9.6 优化网络使用

### 9.6.1 MySQL如何处理客户端连接

本节描述MySQL服务器如何管理客户端连接。

#### 9.6.1.1 网络接口和连接管理器线程

服务器能够监听多个网络接口上的客户机连接。连接管理器线程在服务器监听的网络接口上处理客户端连接请求:

- 在所有平台上，一个管理器线程处理TCP/IP连接请求。

- 在Unix上，同一个管理器线程还处理Unix套接字文件连接请求。

- 在Windows上，一个管理器线程处理共享内存连接请求，另一个线程处理命名管道连接请求。


服务器不会创建线程来处理它不侦听的接口。例如,一个不支持启用命名管道连接的Windows server不会创建线程来处理它们。

#### 9.6.1.2 客户端连接线程管理

连接管理器线程将每个客户机连接与一个专用于该连接的线程关联，该线程处理该连接的身份验证和请求处理。管理器线程在必要时创建一个新线程，但要避免这样做，请首先咨询线程缓存，看看它是否包含可以用于连接的线程。当连接结束时，如果缓存未满，则将其线程返回给线程缓存。

在这个连接线程模型中，当前连接的客户端数量与线程数量相同，当服务器工作负载必须扩展以处理大量连接时，这有一些缺点。例如，线程的创建和处理变得非常昂贵。此外，每个线程都需要服务器和内核资源，比如堆栈空间。为了容纳大量的并发连接，每个线程的堆栈大小必须保持较小，从而导致要么堆栈太小，要么服务器消耗大量内存。其他资源的耗尽也可能发生，并且调度开销可能变得非常大。

MySQL Enterprise Edition包含一个线程池插件，它提供了一个替代的线程处理模型，旨在减少开销和提高性能。它实现了一个线程池，通过高效地管理大量客户机连接的语句执行线程来提高服务器性能。

为了控制和监视服务器如何管理处理客户端连接的线程，几个系统和状态变量是相关的。

`thread_cache_size`系统变量决定线程缓存大小。默认情况下，服务器在启动时自动调整值的大小，但是可以显式地设置它来覆盖此默认值。值0禁用缓存，这将导致为每个新连接设置一个线程，并在连接终止时处理该线程。要启用N个非活动连接线程进行缓存，请在服务器启动时或运行时将`thread_cache_size`设置为`N`。当与连接线程关联的客户机连接终止时，连接线程将变为非活动状态。

要监视缓存中的线程数量以及由于无法从缓存中取出线程而创建了多少线程，请检查`Threads_cached`和`Threads_created`状态变量。

当线程堆栈太小时，这会限制服务器可以处理的SQL语句的复杂性、存储过程的递归深度和其他消耗内存的操作。要为每个线程设置`N`字节的堆栈大小，请使用`thread_stack`启动服务器。在服务器启动时设置为`N`。

#### 9.6.1.3 连接卷管理

若要控制服务器允许同时连接的最大客户端数量，请设置`max_connections`系统变量在服务器启动时或运行时。可能有必要增加`max_connections`如果有更多客户端尝试同时连接，则将服务器配置为处理。

`mysqld`实际上允许`max_connections + 1`客户机连接。额外的连接保留给具有超级特权的帐户使用。通过将特权授予管理员而不是普通用户(普通用户不应该需要它)，拥有流程特权的管理员可以连接到服务器，并使用`SHOW PROCESSLIST`诊断问题，即使连接了最多数量的非特权客户机。

如果服务器因为达到`max_connections`限制而拒绝连接，则会递增`Connection_errors_max_connections`状态变量。

MySQL支持的最大连接数(即`max_connections`的最大值)取决于以下几个因素:

- 给定平台上线程库的质量。

- 可用RAM的数量。

- 每个连接使用的RAM数量。

- 每个连接的工作负载。

- 所需的响应时间。

- 可用文件描述符的数量。

Linux或Solaris通常应该能够支持至少500到1000个并发连接，如果您有很多gb的RAM可用，并且每个RAM的工作负载都很低，或者响应时间目标不高，那么可以支持多达10,000个连接。

增加`max_connections`值会增加`mysqld`需要的文件描述符的数量。如果没有所需的描述符数量，服务器将减少`max_connections`的值。

增加`open_files_limit`系统变量可能是必要的，这也可能需要提高操作系统对MySQL可以使用多少文件描述符的限制。参考您的操作系统文档，确定是否可以增加限制以及如何增加限制。

### 9.6.2 DNS查找优化和主机缓存

MySQL服务器在内存中维护一个主机缓存，其中包含关于客户机的信息:IP地址、主机名和错误信息。性能模式`host_cache`表公开主机缓存的内容，以便可以使用`SELECT`语句检查它。这可以帮助您诊断连接问题的原因。

`注意`

服务器仅对非本地TCP连接使用主机缓存。它不为使用环回接口地址(例如，`127.0.0.1`或`::1`)建立的TCP连接使用缓存，也不为使用Unix套接字文件、命名管道或共享内存建立的连接使用缓存。

`注意`

#### 9.6.2.1 主机缓存操作

服务器将主机缓存用于以下几个目的:

- 通过缓存ip到主机名查找的结果，服务器避免了域名系统(DNS)查找每个客户端连接。相反，对于给定的主机，它只需要对来自该主机的第一个连接执行查找。

- 缓存包含有关连接过程中发生的错误的信息。有些错误被认为是“阻塞”。如果在没有成功连接的情况下，给定主机上连续发生了太多这样的连接，服务器就会阻塞来自该主机的进一步连接。max_connect_errors系统变量确定发生阻塞之前允许的连续错误数量。

对于每个新的客户机连接，服务器使用客户机IP地址检查客户机主机名是否在主机缓存中。如果是，服务器拒绝或继续处理连接请求，这取决于主机是否被阻塞。如果主机不在缓存中，服务器将尝试解析主机名。首先，它将IP地址解析为主机名，并将主机名解析回IP地址。然后将结果与原始IP地址进行比较，以确保它们是相同的。服务器将有关此操作结果的信息存储在hos中

服务器处理主机缓存中的条目如下:

- 当第一个TCP客户机连接从给定的IP地址到达服务器时，将创建一个新的缓存条目来记录客户机IP、主机名和客户机查找验证标志。最初，主机名设置为`NULL`，标志为false。此条目还用于来自相同初始IP的后续客户机TCP连接。

- 如果客户机IP条目的验证标志为false，服务器将尝试IP到主机的名称到IP DNS解析。如果成功，则使用解析后的主机名更新主机名，并将验证标志设置为true。如果解决不成功，所采取的操作取决于错误是永久性的还是暂时性的。对于永久故障，主机名保持`NULL`，验证标志设置为true。对于瞬时故障，主机名和验证标志保持不变。(在本例中，下一次客户机连接时将尝试另一个DNS解析。

- 如果在处理来自给定IP地址的传入客户机连接时发生错误，服务器将在该IP的条目中更新相应的错误计数器。

服务器使用`gethostbyaddr()`和`gethostbyname()`系统调用执行主机名解析。

要解除阻塞的主机，可以通过执行`flush hosts`语句(`TRUNCATE`)来刷新主机缓存表语句，截断性能模式`host_cache`表，或`mysqladmin flush-hosts`命令。刷新主机和`mysqladmin`刷新主机需要重新加载特权。`TRUNCAT`E表需要`host_cache`表的`DROP`特权。

如果从阻塞主机的上次连接尝试以来发生了来自其他主机的活动，即使不刷新主机缓存，阻塞主机也有可能被解除阻塞。这可能发生，因为当连接从不在缓存中的客户机IP到达时，如果缓存已满，服务器将丢弃最近最少使用的缓存条目，以便为新条目腾出空间。如果丢弃的条目是针对阻塞主机的，则该主机将被解除阻塞。

有些连接错误与TCP连接无关，发生在连接过程的早期(甚至在已知IP地址之前)，或者与任何特定的IP地址无关(例如内存不足)。

#### 9.6.2.2 主机缓存配置

默认情况下启用了主机缓存。`host_cache_size`系统变量控制它的大小，以及公开缓存内容的性能模式`host_cache`表的大小。缓存大小可以在服务器启动时设置，并在运行时更改。例如，要在启动时将大小设置为`100`，请将这些行放在服务器`my.cnf`文件中:

```mysql
[mysqld]
host_cache_size=200
```

要在运行时将大小更改为300，请执行以下操作:

```mysql
SET GLOBAL host_cache_size=300;
```

在服务器启动或运行时将`host_cache_size`设置为0，将禁用主机缓存。禁用缓存后，服务器在每次客户机连接时执行DNS查找。

在运行时更改缓存大小会导致隐式`FLUSH HOSTS`操作，该操作清除主机缓存、截断`host_cache`表并解除任何阻塞的主机。

使用`--skip-host-cache`选项类似于将`host_cache_size`系统变量设置为但是`host_cache_size`更灵活，因为它还可以在运行时(而不仅仅是在服务器启动时)调整主机缓存的大小、启用和禁用主机缓存。使用`--skip-host-cache`启动服务器并不会阻止对`host_cache_size`值的更改，但是这些更改没有效果，即使在运行时`host_cache_size`设置为大于0，也不会重新启用缓存。

要禁用DNS主机名查找，请使用`--skip-name-resolve`选项启动服务器。在本例中，服务器仅使用IP地址而不使用主机名来匹配将主机连接到grant MySQL表。只能使用那些表中使用IP地址指定的帐户。(如果没有指定客户机IP地址的帐户，客户机可能无法连接。)

如果您有一个非常慢的DNS和许多主机，您可以通过禁用DNS查找(使用`--skip-name-resolve`)或通过增加`host_cache_size`的值使主机缓存更大来提高性能。

要完全禁用TCP/IP连接，请使用`--skip-networking`选项启动服务器。


