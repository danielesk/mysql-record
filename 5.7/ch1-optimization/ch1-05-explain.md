# 理解查询执行计划(Understanding the Query Execution Plan)

根据表、列、索引的详细信息和WHERE子句中的条件，MySQL优化器考虑了许多技术来有效地执行SQL查询中涉及的查找。可以在不读取所有行的情况下执行对大型表的查询;可以在不比较每个行组合的情况下执行涉及多个表的连接。 优化程序选择执行最有效查询的操作集称为“查询执行计划”，也称为EXPLAIN计划。您的目标是识别EXPLAIN计划中表明查询已经过优化的方面，并了解SQL语法和索引技术，以便在看到一些低效的操作时改进计划。

## 5.1 使用EXPLAIN优化查询

`EXPLAIN`语句提供有关MySQL如何执行语句的信息：

- `EXPLAIN`可以处理`SELECT`、`DELETE`、`INSERT`、`REPLACE`和`UPDATE`语句。

- 当`EXPLAIN`与可解释语句一起使用时，MySQL显示来自优化器的关于语句执行计划的信息。也就是说，MySQL解释了它将如何处理语句，包括关于表如何连接以及以何种顺序连接的信息。

- 当`EXPLAIN`与`FOR CONNECTION connection_id`一起使用而不是可解释语句时，它将显示在命名连接中执行的语句的执行计划。

- 对于SELECT语句，`EXPLAIN`会生成可以使用`SHOW WARNINGS`显示的其他执行计划信息。

- `EXPLAIN`对于检查涉及分区表的查询非常有用。

- `FORMAT`选项可用于选择输出格式。 `TRADITIONAL`以表格格式显示输出。 如果不存在`FORMAT`选项，则这是默认值。 `JSON`格式显示`JSON`格式的信息。


在`EXPLAIN`的帮助下，您可以看到应该在何处向表添加索引，以便通过使用索引查找行来加快语句的执行速度。还可以使用`EXPLAIN`检查优化器是否以最佳顺序连接表。要提示优化器使用与`SELECT`语句中表的命名顺序相对应的连接顺序，请以`SELECT STRAIGHT_JOIN`而不是`SELECT`开始该语句。但是，`STRAIGHT_JOIN`可能会阻止使用索引，因为它会禁用半连接转换。

优化器跟踪有时可能提供与EXPLAIN互补的信息。但是，优化器跟踪格式和内容可能在版本之间发生更改。

如果您在认为应该使用索引时遇到问题，请运行`ANALYZE TABLE`以更新表统计信息，例如密钥的基数，这会影响优化程序的选择让。


`注意`
`EXPLAIN`还可用于获取有关表中列的信息。`EXPLAIN tbl_name`与`DESCRIBE tbl_name`和`SHOW COLUMNS FROM tbl_name`同义。
`注意`


## 5.2 EXPLAIN格式化输出

`EXPLAIN`语句提供了关于MySQL如何执行语句的信息。`EXPLAIN`可以处理`SELECT`、`DELETE`、`INSERT`、`REPLACE`和`UPDATE`语句。

`EXPLAIN`为`SELECT`语句中使用的每个表返回一行信息。 它按照MySQL在处理语句时读取它们的顺序列出输出中的表。 MySQL使用嵌套循环连接方法解析所有连接。 这意味着MySQL从第一个表中读取一行，然后在第二个表，第三个表中找到匹配的行，依此类推。 处理完所有表后，MySQL将通过表列表输出所选列和回溯，直到找到有更多匹配行的表。 从该表中读取下一行，并继续下一个表。

`EXPLAIN`输出包括分区信息。 此外，对于`SELECT`语句，`EXPLAIN`生成扩展信息，可以使用`EXPLAIN`后的`SHOW WARNINGS`显示。

`注意`
在较早的MySQL版本中，使用`EXPLAIN PARTITIONS`和`EXPLAIN EXTENDED`生成分区和扩展信息。 这些语法仍然可以向后兼容，但默认情况下现在启用了分区和扩展输出，因此`PARTITIONS`和`EXTENDED`关键字是多余的并且已弃用。 他们的使用会导致警告，他们会在将来的MySQL版本中从`EXPLAIN`语法中删除。

您不能在同一个`EXPLAIN`语句中一起使用已弃用的`PARTITIONS`和`EXTENDED`关键字。 此外，这些关键字都不能与`FORMAT`选项一起使用。


MySQL Workbench具有可视化解释功能，它提供了` EXPLAIN `输出的可视化表示。
`注意`

### 5.2.1 EXPLAIN输出列 

本节介绍EXPLAIN生成的输出列。 后面的部分提供有关`type`和`Extra`列的其他信息。

`EXPLAIN`中的每个输出行提供关于一个表的信息。列名显示在表的第一列中;当使用`FORMAT=JSON`时，第二列提供输出中显示的等效属性名。

![](../images/ch1-05-explain-1.png)

`注意`
`NULL`格式的SON属性不以JSON格式显示EXPLAIN输出。
`注意`


- `id` (JSON名称:select_id)

`SELECT`标识符。 这是查询中`SELECT`的序列号。 如果行引用其他行的并集结果，则该值可以为`NULL`。 在这种情况下，表列显示类似`<unionM，N>`的值，表示该行引用`id`值为`M`和`N`的行的并集。

- `select_type` (JSON名称:none)

`SELECT`的类型，可以是下表中显示的任何类型。 JSON格式的`EXPLAIN`将`SELECT`类型公开为`query_block`的属性，除非它是`SIMPLE`或`PRIMARY`。 JSON名称（如果适用）也显示在表中。

![](../images/ch1-05-explain-2.png)

`DEPENDENT`通常表示使用关联子查询。

`DEPENDENT SUBQUERY`评估与`UNCACHEABLE SUBQUERY`评估不同。 对于`DEPENDENT SUBQUERY`，子查询仅针对来自其外部上下文的变量的每组不同值重新评估一次。 对于`UNCACHEABLE SUBQUERY`，将为外部上下文的每一行重新评估子查询。

子查询的可缓存性不同于查询缓存中查询结果的缓存。子查询缓存发生在查询执行期间，而查询缓存仅用于在查询执行完成后存储结果。

使用`EXPLAIN`指定`FORMAT = JSON`时，输出没有直接等同于`select_type`的单个属性; `query_block`属性对应于给定的`SELECT`。 可以使用与刚显示的大多数`SELECT`子查询类型等效的属性（示例为`MATERIALIZED`的`materialized_from_subquery）`，并在适当时显示。 `SIMPLE`或`PRIMARY`没有JSON等价物。

non-`SELECT`语句的`select_type`值显示受影响的表的语句类型。例如，对于`DELETE`语句，`select_type`为`DELETE`。

- `table` (JSON名称:table_name)

输出行所指向的表的名称。这也可以是下列值之一:

1. `<unionM,N>`:行表示`id`值为`M`和`N`的行的并集。

2. `<derivedN>`：该行引用`id`值为`N`的行的派生表结果。例如，派生表可能来自`FROM`子句中的子查询。`<derivedN>`：该行引用派生表 `id`值为`N`的行的结果。例如，派生表可能来自`FROM`子句中的子查询。

3. `<subqueryN>`：该行引用`id`值为`N`的行的具体化子查询的结果。


- `partitions`(JSON名称:partition)

查询将匹配记录所在的分区。对于非分区表，该值为`NULL`。

- `type`(JSON名称:access_type)

连接类型。

-  `possible_keys` (JSON名称: possible_keys)

`possible_keys`列指示MySQL可以选择从中查找此表中的行的索引。 请注意，此列完全独立于`EXPLAIN`输出中显示的表的顺序。 这意味着`possible_keys`中的某些键可能无法在生成中使用生成的表顺序。

如果此列为`NULL`（或在JSON格式的输出中未定义），则没有相关索引。 在这种情况下，您可以通过检查WHERE子句来提高查询的性能检查它是否引用了一些适合索引的列或列。 如果是，请创建适当的索引并再次使用`EXPLAIN`检查查询。

要查看表有哪些索引，请使用`SHOW INDEX FROM tbl_name`。

-  `key` (JSON名称: key)

`key`列表示MySQL实际决定使用的键(索引)。如果MySQL决定使用一个`possible le_keys`索引来查找行，那么该索引将作为键值列出。

`key`可能会命名一个不存在于`possible_keys`值中的索引。 如果所有`possible_keys`索引都不适合查找行，则会发生这种情况，但查询选择的所有列都是其他索引的列。 也就是说，命名索引覆盖了所选列，因此虽然它不用于确定要检索的行，但索引扫描比数据行扫描更有效。

对于InnoDB，即使查询还选择主键，辅助索引也可能覆盖所选列，因为InnoDB将主键值与每个辅助索引一起存储。 如果key为NULL，则MySQL找不到用于更有效地执行查询的索引。

要强制MySQL使用或忽略`possible_keys`列中列出的索引，请在查询中使用`FORCE INDEX`，`USE INDEX`或`IGNORE INDEX`。对于`MyISAM`表，运行`ANALYZE TABLE`可帮助优化器选择更好的索引。对于`MyISAM`表，`myisamchk --analyze`也是如此。

- `key_len` (JSON名称：key_length)

`key_len`列指示MySQL决定使用的密钥的长度。 `key_len`的值使您可以确定MySQL实际使用的多部分密钥的多少部分。如果键列显示`NULL`，则`len_len`列也会显示`NULL`。

由于密钥存储格式，对于可以为NULL的列而言，密钥长度比对于NOT NULL列更大。

- ref (JSON名称: ref)

`ref`列显示哪些列或常量与`key`列中指定的索引进行比较，以从表中选择行。

如果值为`func`，则使用的值是某个函数的结果。要查看哪个函数，请使用`EXPLAIN`之后的`SHOW WARNINGS`来查看扩展的`EXPLAIN`输出。该函数实际上可能是算术运算符等运算符。

- rows (JSON名称 rows)

rows列表示MySQL认为必须检查以执行查询的行数。
对于InnoDB表，此数字是估计值，可能并不总是准确的。

-  filtered (JSON名称: filtered)

`filtered`列指示将按表条件筛选的表行的估计百分比。最大值为100，这意味着不会对行进行过滤。值从100开始减少表示过滤量增加。`rows`显示检查的估计行数，`rows`×`filtered`显示将与下表连接的行数。例如，如果`rows`为1000且`filtered`为50.00（50％），则使用下表连接的行数为1000×50％= 500。

- Extra (JSON名称: none)

此列包含有关MySQL如何解析查询的其他信息。

没有与`Extra`列对应的单个JSON属性;但是，此列中可能出现的值将作为JSON属性公开，或者作为`message`属性的文本公开。

### 5.2.2 EXPLAIN Join 类型

`EXPLAIN`输出的`type`列描述了表的连接方式。在JSON格式的输出中，这些是作为`access_type`属性的值找到的。以下列表描述了从最佳类型到最差类型的连接类型：

- system 

该表只有一行（=系统表）。这是`const`连接类型的特例。

- const

该表最多只有一个匹配行，在查询开头读取。因为只有一行，所以优化器的其余部分可以将此行中列的值视为常量。 `const`表非常快，因为它们只读一次。

将`PRIMARY KEY`或`UNIQUE`索引的所有部分与常量值进行比较时使用`const`。在以下查询中，`tbl_name`可用作`const`表：

```mysql
SELECT * FROM tbl_name WHERE primary_key=1;
SELECT * FROM tbl_name
WHERE primary_key_part1=1 AND primary_key_part2=2;
```

- eq_ref

对于前面表格中的每个行组合，从该表中读取一行。除了`system`和`const`类型，这是最好的连接类型。当连接使用索引的所有部分并且索引是`PRIMARY KEY`或`UNIQUE NOT NULL`索引时使用它。

`eq_ref`可用于使用`=`运算符进行比较的索引列。比较value可以是常量，也可以是使用在此表之前读取的表中的列的表达式。在以下示例中，MySQL可以使用`eq_ref`连接来处理`ref_table`：

```mysql
SELECT * FROM ref_table,other_table
WHERE ref_table.key_column=other_table.column;
SELECT * FROM ref_table,other_table
WHERE ref_table.key_column_part1=other_table.column
AND ref_table.key_column_part2=1;
```

- ref

对于每个行的组合，从该表中读取具有匹配索引值的所有行以前的表格。如果连接仅使用键的最左前缀或者键不是`PRIMARY KEY`或`UNIQUE`索引（换句话说，如果连接不能基于键值选择单行），则使用`ref`。如果使用的密钥只匹配几行，这是一个很好的连接类型。

`ref`可用于使用`=`或`<=>`运算符进行比较的索引列。在以下示例中，MySQL可以使用`ref join`来处理`ref_table`：

```mysql
SELECT * FROM ref_table WHERE key_column=expr;
SELECT * FROM ref_table,other_table
WHERE ref_table.key_column=other_table.column;
SELECT * FROM ref_table,other_table
WHERE ref_table.key_column_part1=other_table.column
AND ref_table.key_column_part2=1;
```

- fulltext

使用`FULLTEXT`索引执行连接。

- ref_no_null

这种连接类型与`ref`类似，但附加的是MySQL对包含`NULL`值的行进行额外搜索。此连接类型优化最常用于解析子查询。在以下示例中，MySQL可以使用`ref_or_null`连接来处理`ref_table`：

```mysql
SELECT * FROM ref_table
WHERE key_column=expr OR key_column IS NULL;
```

- index_merge

此连接类型表示使用了索引合并优化。在这种情况下，输出行中的`key`列包含使用的索引列表，`key_len`包含所用索引的最长键部分列表。

- unique_subquery

此类型替换以下形式的某些`IN`子查询的`eq_ref`：

```mysql
value IN (SELECT primary_key FROM single_table WHERE some_expr)
```

`unique_subquery`只是一个索引查找函数，它可以完全替换子查询以提高效率。

-  index_subquery

此连接类型类似于`unique_subquery`。它取代了`IN`子查询，但它适用于以下形式的子查询中的非唯一索引：

```mysql
value IN (SELECT key_column FROM single_table WHERE some_expr)
```

- range

仅检索给定范围内的行，使用索引选择行。输出行中的`key`列指示使用哪个索引。 `key_len`包含使用的最长的关键部分。对于此类型，`ref`列为`NULL`。

使用`=`，`<>`，`>`，`> =`，`<`，`<=`，`IS NULL`，`<=>`，`BETWEEN`，`LIKE`或`IN()`运算符中的任何一个将键列与常量进行比较时，可以使用范围：

```mysql
SELECT * FROM tbl_name
WHERE key_column = 10;
SELECT * FROM tbl_name
WHERE key_column BETWEEN 10 and 20;
SELECT * FROM tbl_name
WHERE key_column IN (10,20,30);
SELECT * FROM tbl_name
WHERE key_part1 = 10 AND key_part2 IN (10,20,30);
```

- index

`index`连接类型与`ALL`相同，只是扫描索引树。这有两种方式：

1.  如果索引是查询的覆盖索引，并且可用于满足表中所需的所有数据，则仅扫描索引树。在这种情况下，`Extra`列显示`Using index`。仅索引扫描通常比`ALL`快，因为索引的大小通常小于表数据。

2.  使用索引中的读取执行全表扫描，以按索引顺序查找数据行。使用索引不会出现在`Extra`列中。

    当查询仅使用属于单个索引的列时，MySQL可以使用此连接类型。

- All

对前面表格中的每个行组合进行全表扫描。如果表是第一个没有标记为`const`的表，这通常是不好的，并且在所有其他情况下通常非常糟糕。通常，您可以通过添加索引来避免`ALL`，这些索引根据以前表中的常量值或列值从表中启用行检索。

### 5.2.3 解释额外信息(EXPLAIN Extra Information)

































