# 数据库结构优化

在您作为数据库设计者的角色中，寻找最有效的方式去管理你的schemas、tables和columns。与调优应用程序代码时一样，您将最小化I/O，将相关项放在一起，并提前计划，以便随着数据量的增加性能保持较高。从高效的数据库设计开始，团队成员可以更轻松地编写高性能的应用程序代码，并使数据库可以随着应用程序的发展和重写而持久。

## 3.1 优化数据大小(Optimizing Data Size)

设计您的表以最小化磁盘上的空间。通过减少写入和读取磁盘的数据量，这可以带来巨大的改进。较小的表通常需要较少的主内存，而其内容在查询执行期间被主动处理。 表数据的任何空间缩减也会导致较小的索引可以更快地处理。

MySQL支持许多不同的存储引擎(表类型)和行格式。对于每个表，您可以决定使用哪种存储和索引方法。为应用程序选择合适的表格式可以大大提高性能。

### 3.1.1 表列(Table Columns)

- 尽可能使用最有效(最小)的数据类型。MySQL有许多专门的类型可以节省磁盘空间和内存。例如，如果可能的话，使用较小的整数类型来获得较小的表。`MEDIUMINT`通常是比`INT`更好的选择，因为`MEDIUMINT`列使用的空间少25%。

- **如果可能，将列声明为`NOT NULL`。通过更好地使用索引和消除测试每个值是否为`NULL`的开销，它使SQL操作更快。** 您还节省了一些存储空间，每列一个位。如果您确实需要表中的`NULL`值，请使用它们。只要避免默认设置，即允许每一列中都有`NULL`值。

### 3.1.2 行格式(Row Format)

- 默认情况下，`InnoDB`表是使用`DYNAMIC`行格式创建的。要使用`DYNAMIC`以外的行格式，请配置`innodb_default_row_format`，或在`CREATE TABLE`或`ALTER TABLE`语句中显式指定`ROW_FORMAT`选项。

紧凑的行格式系列( `COMPACT`, `DYNAMIC` 和 `COMPRESSED`)减少了行存储空间，但代价是增加了某些操作的CPU使用量。如果您的工作负载是受缓存命中率和磁盘速度限制的典型工作负载，那么它可能会更快。如果受到CPU速度限制的情况很少见，那么可能会慢一些。

 当使用可变长度字符集（如`utf8mb3`或`utf8mb4`）时，紧凑的行格式系列还可优化`CHAR`列存储。当`ROW_FORMAT=REDUNDANT`，`CHAR(N)`占用了`N`×字符集的最大字节长度。许多语言主要可以使用单字节`utf8`字符来编写，因此固定的存储长度常常会浪费空间。借助紧凑的行格式系列，`InnoDB`在`N`到`N`的范围内分配可变数量的存储 × 这些列的字符集的最大字节长度 通过去除尾随空格。(With the compact family of rows formats, InnoDB allocates a variable amount of storage in the range of N to N
× the maximum byte length of the character set for these columns by stripping trailing spaces.)。最小存储长度为`N`字节，以方便在典型情况下进行就地更新。

- 若要通过以压缩形式存储表数据进一步减少空间，请指定`ROW_FORMAT=COMPRESSED`在创建`InnoDB`表时，或者在现有的`MyISAM`表上运行`myisampack`命令。(`InnoDB`压缩表是可读可写的，而`MyISAM`压缩表是只读的。)

- 对于`MyISAM`表，如果没有任何可变长度的列(`VARCHAR`、`TEXT`或`BLOB`列)，则使用固定大小的行格式。这更快，但可能会浪费一些空间。您可以使用`CREATE TABLE`选项`ROW_FORMAT= FIXED`提示您希望有固定长度的行，即使您有`VARCHAR`列。

### 3.1.3 索引(Indexes)




