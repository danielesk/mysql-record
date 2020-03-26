# 3.1 数据类型概述

## 3.1.1 (Numeric)数值类型概述

下面是数字数据类型的摘要。

`M`表示整数类型的最大显示宽度。最大显示宽度为255.显示宽度与类型可包含的值范围无关，如第11.2节“数值类型”中所述。对于浮点和定点类型，`M`是可以存储的总位数。

如果为数字列指定`ZEROFILL`，MySQL会自动将`UNSIGNED`属性添加到列中。

允许`UNSIGNED`属性的数字数据类型也允许`SIGNED`。但是，默认情况下会对这些数据类型进行签名，因此`SIGNED`属性不起作用。

`SERIAL`是`BIGINT UNSIGNED NOT NULL AUTO_INCREMENT UNIQUE`的别名。

整数列定义中的`SERIAL DEFAULT VALUE`是`NOT NULL AUTO_INCREMENT UNIQUE`的别名。

`警告`

在整数值之间使用减法时，其中一个类型为`UNSIGNED`，除非启用了`NO_UNSIGNED_SUBTRACTION` SQL模式，否则结果为无符号。

• `BIT[(M)]`

位值类型。 M表示每个值的位数，从1到64.如果省略M，则默认值为1。

• `TINYINT [(M)]` `[UNSIGNED]` `[ZEROFILL]`

一个非常小的整数。有符号范围是-128到127.无符号范围是0到255。

• `BOOL`，`BOOLEAN`

这些类型是TINYINT（1）的同义词。值为零被视为false。非零值被认为是真的：

```mysql
mysql> SELECT IF(0, 'true', 'false');
+------------------------+
| IF(0, 'true', 'false') |
+------------------------+
| false                  |
+------------------------+

mysql> SELECT IF(1, 'true', 'false');
+------------------------+
| IF(1, 'true', 'false') |
+------------------------+
| true                   |
+------------------------+

mysql> SELECT IF(2, 'true', 'false');
+------------------------+
| IF(2, 'true', 'false') |
+------------------------+
| true                   |
+------------------------+
```

然而，`TRUE`和`FALSE`值只是分别为`1`和`0`的别名，如下所示:

```mysql
mysql> SELECT IF(0 = FALSE, 'true', 'false');
+--------------------------------+
| IF(0 = FALSE, 'true', 'false') |
+--------------------------------+
| true                           |
+--------------------------------+

mysql> SELECT IF(1 = TRUE, 'true', 'false');
+-------------------------------+
| IF(1 = TRUE, 'true', 'false') |
+-------------------------------+
| true                          |
+-------------------------------+

mysql> SELECT IF(2 = TRUE, 'true', 'false');
+-------------------------------+
| IF(2 = TRUE, 'true', 'false') |
+-------------------------------+
| false                         |
+-------------------------------+

mysql> SELECT IF(2 = FALSE, 'true', 'false');
+--------------------------------+
| IF(2 = FALSE, 'true', 'false') |
+--------------------------------+
| false                          |
+--------------------------------+
```

最后两个语句显示了结果，因为`2`既不等于`1`也不等于`0`。

• `SMALLINT [(M)] [UNSIGNED] [ZEROFILL]`

一个小整数。有符号范围是`-32768`到`32767`.无符号范围是`0`到`65535`。

• `MEDIUMINT [(M)] [UNSIGNED] [ZEROFILL]`

一个中等大小的整数。有符号范围是`-8388608`到`8388607`.无符号范围是`0`到`16777215`。

• `INT [(M)] [UNSIGNED] [ZEROFILL]`

正常大小的整数。有符号范围是`-2147483648`到`2147483647`.无符号范围是`0`到`4294967295`。

• `INTEGER [(M)] [UNSIGNED] [ZEROFILL]`

此类型是`INT`的同义词。

• `BIGINT [(M)] [UNSIGNED] [ZEROFILL]`

一个大整数。符号范围是`-9223372036854775808`到`9223372036854775807`。无符号范围是`0`到`18446744073709551615`。`SERIAL`是`BIGINT UNSIGNED NOT NULL AUTO_INCREMENT UNIQUE`的别名。

`SERIAL`是`BIGINT UNSIGNED NOT NULL AUTO_INCREMENT UNIQUE`的别名。

关于BIGINT列，您应该注意的一些事项：

•所有算术都使用带符号的BIGINT或DOUBLE值完成，因此除了位功能外，不应使用大于`9223372036854775807（63位）`的无符号大整数！如果这样做，结果中的一些最后数字可能是错误的，因为在将BIGINT值转换为DOUBLE时出现舍入错误。

在以下情况下，MySQL可以处理BIGINT：

• 使用整数在BIGINT列中存储大的无符号值时。

• 在`MIN（col_name）`或`MAX（col_name）`中，其中`col_name`指的是`BIGINT`列。

• 使用运算符（`+`， `-` ，`*`等）时，两个操作数都是整数。

• 通过使用字符串存储，您始终可以在BIGINT列中存储精确的整数值。在这种情况下，MySQL执行字符串到数字的转换，不涉及中间双精度表示。

• 当两个操作数都是整数值时， `-` ，`+`和`*`运算符使用BIGINT算法。这意味着如果将两个大整数（或返回整数的函数的结果）相乘，则当结果大于`9223372036854775807`时，可能会得到意外结果。


• `DECIMAL [（M [，D]）] [UNSIGNED] [ZEROFILL]`

打包的“精确”定点数。 `M`是总位数（精度），`D`是小数点后的位数（刻度）。小数点和（对于负数） - 符号不计入`M`.如果`D`为`0`，则值没有小数点或小数部分。 `DECIMAL`的最大位数（`M`）为65.支持的最大小数（`D`）为30.如果省略`D`，则默认值为0.如果省略M，则默认值为10.UNSIGNED，如果指定，不允许负值。使用DECIMAL列的所有基本计算（`+`， `-` ，`*`，/）都是以65位数的精度完成的。

• `DEC [（M [，D]）] [UNSIGNED] [ZEROFILL]，NUMERIC [（M [，D]）] [UNSIGNED] [ZEROFILL]，FIXED[（M [，D]）] [UNSIGNED] [ZEROFILL ]`

这些类型是DECIMAL的同义词。 FIXED同义词可用于与其他数据库系统兼容。

• `FLOAT [（M，D）] [UNSIGNED] [ZEROFILL]`

一个小的（单精度）浮点数。允许值为`-3.402823466E+38`至`-1.175494351E-38`,`0`,`1.175494351E-38`至`3.402823466E+38`。这些是基于IEEE标准的理论限制。实际范围可能略小，具体取决于您的硬件或操作系统。
 
`M`是总位数，`D`是小数点后面的位数。如果省略`M`和`D`，则将值存储到硬件允许的限制。单精度浮点数精确到大约7位小数。 

`FLOAT（M，D）`是一个非标准的MySQL扩展。如果指定，`UNSIGNED`不允许负值。

使用`FLOAT`可能会给您带来一些意想不到的问题，因为MySQL中的所有计算都以双倍的精度完成。


• `FLOAT（p）[UNSIGNED] [ZEROFILL]`

一个浮点数。 `p`表示以位为单位的精度，但MySQL仅使用此值来确定是否对结果数据类型使用`FLOAT`或`DOUBLE`。如果p为`0`到`24`，则数据类型变为`FLOAT`，没有`M`或`D`值。如果`p`为`25`到`53`，则数据类型变为`DOUBLE`，没有`M`或`D`值。结果列的范围与本节前面介绍的单精度`FLOAT`或双精度`DOUBLE`数据类型的范围相同。为ODBC兼容性提供了`FLOAT（p）`语法。

• `DOUBLE [（M，D）] [UNSIGNED] [ZEROFILL]`

正常大小（双精度）浮点数。允许值为`-1.7976931348623157E+308`至`-2.2250738585072014E-308`,`0`,`2.2250738585072014E-308`至`1.7976931348623157E+308`。这些是基于IEEE标准的理论限制。实际范围可能略小，具体取决于您的硬件或操作系统。

M是总位数，D是小数点后面的位数。如果省略M和D，则将值存储到硬件允许的限制。双精度浮点数精确到大约15位小数。

`DOUBLE（M，D）`是一个非标准的MySQL扩展。

如果指定，`UNSIGNED`不允许负值。

• `DOUBLE PRECISION[(M,D)] [UNSIGNED] [ZEROFILL]，REAL [（M，D）] [UNSIGNED] [ZEROFILL]`

这些类型是`DOUBLE`的同义词。例外：如果启用了`REAL_AS_FLOAT` SQL模式，则`REAL`是`FLOAT`的同义词，而不是`DOUBLE`。

## 3.1.2 日期和时间类型概述(Date and Time Type Overview)

下面简要介绍时态数据类型。

对于`DATE`和`DATETIME`范围描述，“支持”意味着尽管早期值可能有效，但无法保证。

MySQL允许`TIME`，`DATETIM`E和`TIMESTAMP`值的小数秒，精度高达微秒（6位）。要定义包含小数秒部分的列，请使用语法`type_name（fsp）`，其中`type_name`为`TIME`，`DATETIME`或`TIMESTAMP`，`fsp`是小数秒精度。例如：

```mysql
CREATE TABLE t1 (t TIME(3), dt DATETIME(6));
```

如果给定，则`fsp`值必须在0到6之间。值为0表示没有小数部分。如果省略，默认精度为0。(为了兼容以前的MySQL版本，这与标准SQL默认值6不同。)

表中的任何`TIMESTAMP`或`DATETIME`列都可以具有自动初始化和更新属性。

- `DATE`

date。支持的范围是`'1000-01-01'`到`'9999-12-31'`。 MySQL以`'YYYY-MM-DD'`格式显示`DATE`值，但允许使用字符串或数字将值分配给`DATE`列。

- `DATETIME [（fsp）]`

日期和时间组合。支持的范围是`'1000-01-01 00：00：00.000000'`到`'9999-12-31 23：59：59.999999'`。 MySQL显示`DATETIME`值`'YYYY-MM-DD hh：mm：ss [.fraction]'`格式，但允许使用字符串或数字将值分配给`DATETIME`列。

可以给出0到6范围内的可选`fsp`值以指定小数秒精度。值为0表示没有小数部分。如果省略，则默认精度为0。

可以使用`DEFAULT`和`ON UPDATE`列定义子句指定`DATETIME`列的当前日期和时间的自动初始化和更新，如第11.3.5节“TIMESTAMP和DATETIME的自动初始化和更新”中所述。


- `TIMESTAMP [（fsp）]`

`timestamp`。范围是`'1970-01-01 00：00：01.000000'`UTC到`'2038-01-19 03：14：07.999999'`UTC。 `TIMESTAMP`值存储为自纪元以来的秒数（`'1970-01-01 00:00:00'`UTC）。 `TIMESTAMP`不能表示值`'1970-01-01 00:00:00'`，因为它相当于纪元的0秒，并且值0保留为
表示`'0000-00-00 00:00:00'`，“零”`TIMESTAMP`值。

可以给出0到6范围内的可选`fsp`值以指定小数秒精度。值0表示没有小数部分。如果省略，则默认精度为0。

服务器处理`TIMESTAM`P定义的方式取决于`explicit_defaults_for_timestamp`系统变量的值（请参见第5.1.7节“服务器系统变量”）。

如果启用了`explicit_defaults_for_timestamp`，则不会将`DEFAULT CURRENT_TIMESTAMP`或`ON UPDATE CURRENT_TIMESTAMP`属性自动分配给任何`TIMESTAMP`列。它们必须明确包含在列定义中。此外，任何未显式声明为`NOT NULL`的`TIMESTAMP`都允许`NULL`值。

如果禁用`explicit_defaults_for_timestamp`，则服务器将按如下方式处理`TIMESTAMP`：

除非另有说明，否则表中的第一个`TIMESTAMP`列被定义为自动设置为最近修改的日期和时间（如果未明确赋值）。这使得`TIMESTAMP`可用于记录`INSERT`或`UPDATE`操作的时间戳。您还可以通过为任何`TIMESTAMP`列指定`NULL`值来将其设置为当前日期和时间，除非已使用`NULL`属性定义该值以允许`NULL`值。

可以使用`DEFAULT CURRENT_TIMESTAMP`和`ON UPDATE CURRENT_TIMESTAMP`列定义子句指定自动初始化和更新到当前日期和时间。默认情况下，第一个`TIMESTAMP`列具有这些属性，如前所述。但是，可以将表中的任何`TIMESTAMP`列定义为具有这些属性。


- `TIME [（fsp）]`

`time`。范围是`'-838：59：59.000000'`到`'838：59：59.000000'`。 MySQL以`'hh：mm：ss [.fraction]'`格式显示`TIME`值，但允许将值分配给`TIME`列使用字符串或数字。

可以给出0到6范围内的可选`fsp`值以指定小数秒精度。值为0表示没有小数部分。如果省略，则默认精度为0。


- `YEAR[（4）]`
  
一年四位数格式。 MySQL以`YYYY`格式显示`YEAR`值，但允许使用字符串或数字将值分配给`YEAR`列。值显示为`1901`到`2155`和`0000`。

`注意：不推荐使用YEAR（2）数据类型，并在MySQL 5.7.5中删除对它的支持。要将YEAR（2）列转换为YEAR（4），`


`SUM()`和`AVG()`聚合函数不适用于时间值。 （它们将值转换为数字，在第一个非数字字符后丢失所有内容。）要解决此问题，请转换为数字单位，执行聚合操作，然后转换回时间值。例子：

```mysql
SELECT SEC_TO_TIME(SUM(TIME_TO_SEC(time_col))) FROM tbl_name;
SELECT FROM_DAYS(SUM(TO_DAYS(date_col))) FROM tbl_name;
```

`注意`

可以在启用`MAXDB` SQL模式的情况下运行MySQL服务器。在这种情况下，`TIMESTAMP`与`DATETIME`相同。如果在创建表时启用此模式，则会将`TIMESTAMP`列创建为`DATETIME`列。因此，此类列使用`DATETIME`显示格式，具有相同的值范围，并且不会自动初始化或更新到当前日期和时间。

从MySQL 5.7.22开始，不推荐使用MAXDB。它将在MySQL的未来版本中被删除。

`注意`

## 3.1.3 字符串类型概述

下面是字符串数据类型的摘要。

在某些情况下，MySQL可能会将字符串列更改为与CREATE中给定的类型不同的类型表或ALTER TABLE语句。

MySQL在字符单位的字符列定义中解释长度规范。这适用于`CHAR`、`VARCHAR`和`TEXT`类型。

字符串数据类型`CHAR`，`VARCHAR`，`TEXT`类型，`ENUM`，`SET`和任何同义词的列定义可以指定列字符集和排序规则：

- `CHARACTER SET`指定字符集。如果需要，可以使用`COLLATE`属性以及任何其他属性指定字符集的排序规则。例如：

```mysql
CREATE TABLE t
(
c1 VARCHAR(20) CHARACTER SET utf8,
c2 TEXT CHARACTER SET latin1 COLLATE latin1_general_cs
);
```

此表定义创建一个名为`c1`的列，其字符集为`utf8`，具有该字符集的默认排序规则，以及一个名为`c2`的列，其字符集为`latin1`，区分大小写的排序规则。

第10.3.5节“列字符集和排序规则”中介绍了在缺少`CHARACTER SET`和`COLLATE`属性之一或两者时分配字符集和排序规则的规则。

`CHARSET`是`CHARACTER SET`的同义词。 

- 为字符串数据类型指`定CHARACTER SET binary` 属性会导致将列创建为相应的二进制字符串数据类型：`CHAR`变为`BINARY`，`VARCHAR`变为`VARBINARY`，`TEXT`变为`BLOB`。对于`ENUM`和`SET`数据类型，不会发生这种情况;它们是按声明创建的。假设您使用此定义指定表：

```mysql
CREATE TABLE t
(
c1 VARCHAR(10) CHARACTER SET binary,
c2 TEXT CHARACTER SET binary,
c3 ENUM('a','b','c') CHARACTER SET binary
);
```

得到的表定义如下:

```mysql
CREATE TABLE t
(
c1 VARBINARY(10),
c2 BLOB,
c3 ENUM('a','b','c') CHARACTER SET binary
);
```

- `BINARY`属性是指定表缺省字符集和该字符集的二进制（`_bin`）归类的简写。在这种情况下，比较和排序基于数字字符代码值。

- `ASCII`属性是`CHARACTER SET latin1`的简写。

- `UNICODE`属性是`CHARACTER SET ucs2`的简写。字符列比较和排序基于分配给列的排序规则。对于`CHAR`，`VARCHAR`，`TEXT`，`ENUM`和`SET`数据类型，可以声明具有二进制（`_bin`）排序规则或`BINARY`属性的列，以使比较和排序使用基础字符代码值而不是词法排序。

- `[NATIONAL] CHAR[(M)] [CHARACTER SET charset_name] [COLLATE collation_name]`

一个固定长度的字符串，在存储时始终用空格填充指定的长度。 `M`表示以字符为单位的列长度。 `M`的范围是0到255.如果省略`M`，则长度为1。

`注意：除非启用PAD_CHAR_TO_FULL_LENGTH SQL模式，否则在检索CHAR值时将删除尾随空格。`

`CHAR`是`CHARACTER`的简写。 `NATIONAL CHAR`（或其等效的简写形式，`NCHAR`）是定义`CHAR`列应使用某些预定义字符集的标准SQL方法。 MySQL使用`utf8`作为此预定义字符集。

`CHAR BYTE`数据类型是`BINARY`数据类型的别名。这是兼容性功能。


MySQL允许您创建`CHAR（0）`类型的列。当您必须符合依赖于列的存在但实际上不使用其值的旧应用程序时，这非常有用。当你需要一个只能有两个值的列时，`CHAR（0）`也很好：一个定义为`CHAR（0）`的列只占用一位，只能取值`NULL`和`''`（空字符串） 。

- `[NATIONAL] VARCHAR（M）[CHARACTER SET charset_name] [COLLATE collat​​ion_name]`

可变长度的字符串。 `M`表示字符的最大列长度。 `M`的范围是`0`到`65,535`。 `VARCHAR`的有效最大长度取决于最大行大小（`65,535`字节，在所有列之间共享）和使用的字符集。例如，`utf8`字符每个字符最多可能需要`三`个字节，因此使用`utf8`字符集的`VARCHAR`列可以声明为最多`21,844`个字符。

MySQL将`VARCHAR`值存储为`1`字节或`2`字节长度前缀加数据。长度前缀表示值中的字节数。如果值不超过`255`个字节，则`VARCHAR`列使用一个长度字节;如果值可能需要超过`255`个字节，则使用`两`个长度字节。

`注意：MySQL遵循标准SQL规范，不会从VARCHAR值中删除尾随空格。`


`VARCHAR`是`CHARACTER VARYING`的简写。 `NATIONAL VARCHAR`是定义`VARCHAR`列应使用某些预定义字符集的标准SQL方法。 MySQL使用utf8作为此预定义字符集。第10.3.7节“国家字符集”。 NVARCHAR是NATIONAL VARCHAR的简写。

- `BINARY [（M）]`

`BINARY`类型类似于`CHAR`类型，但存储二进制字节字符串而不是非二进制字符串。可选长度`M`表示以字节为单位的列长度。如果省略，`M`默认为`1`。
- `VARBINARY（M）`

`VARBINARY`类型类似于`VARCHAR`类型，但存储二进制字节字符串而不是非二进制字符串。 `M`表示以字节为单位的最大列长度。

- `TINYBLOB`

`BLOB`列，最大长度为`255（2 ^ 8  -  1）`个字节。每个`TINYBLOB`值使用`1`字节长度前缀存储，该前缀指示值中的字节数。

- `TINYTEXT [CHARACTER SET charset_name] [COLLATE collat​​ion_name]`

`TEXT`列，最大长度为`255（2 ^ 8  -  1）`个字符。如果值包含多字节字符，则有效最大长度较小。每个`TINYTEXT`值使用`1`字节长度前缀存储，该前缀指示值中的字节数。

- `BLOB [（M）]`

`BLOB`列，最大长度为`65,535（2 ^ 16  -  1）`个字节。每个`BLOB`值使用`2`字节长度前缀存储，该前缀指示值中的字节数。可以为此类型指定可选长度`M`。如果这样做，MySQL会将列创建为最小的`BLOB`类型，其大小足以保存`M`字节长的值。


- `TEXT[(M)] [CHARACTER SET charset_name] [COLLATE collation_name]`

一个`TEXT`列，最大长度为`65,535（2 ^ 16  -  1）`个字符。 如果值包含多字节字符，则有效最大长度较小。 每个TEXT值使用2字节长度前缀存储，该前缀指示值中的字节数。

可以为此类型指定可选长度`M`。 如果这样做，MySQL会将列创建为最小的`TEXT`类型，其大小足以容纳`M`个字符长的值。


- `MEDIUMBLOB`

`BLOB`列，最大长度为`16,777,215（2 ^ 24  -  1）`个字节。 每个`MEDIUMBLOB`值使用`3`字节长度前缀存储，该前缀指示值中的字节数。

- `MEDIUMTEXT [CHARACTER SET charset_name] [COLLATE collation_name]`

一个`TEXT`列，最大长度为`16,777,215（2 ^ 24  -  1）`个字符。 如果值包含多字节字符，则有效最大长度较小。 每个`MEDIUMTEXT`值使用`3`字节长度前缀存储，该前缀指示值中的字节数。

- `LONGBLOB`

`BLOB`列，最大长度为`4,294,967,295`或`4GB（2 ^ 32  -  1）`字节。 `LONGBLOB`列的有效最大长度取决于客户端/服务器协议中配置的最大数据包大小和可用内存。 每个`LONGBLOB`值使用`4`字节长度前缀存储，该前缀指示值中的字节数。

- `LONGTEXT [CHARACTER SET charset_name] [COLLATE collation_name]`

一个`TEXT`列，最大长度为`4,294,967,295`或`4GB（2 ^ 32  -  1）`个字符。 如果值包含多字节字符，则有效最大长度较小。 有效最大长度`LONGTEXT`列还取决于客户端/服务器协议中配置的最大数据包大小和可用内存。 每个`LONGTEXT`值使用`4`字节长度前缀存储，该前缀指示值中的字节数。

- `ENUM（'value1'，'value2'，...）[CHARACTER SET charset_name] [COLLATE collat​​ion_name]`

`enum`。一个字符串对象，只能有一个值，从值列表`'value1'`，`'value2'`，`...`，`NULL`或特殊的`''`错误值中选择。 `ENUM`值在内部表示为整数。

`ENUM`列最多可包含`65,535`个不同的元素。 （实际限制小于`3000`.）一个表在其被视为一个组的`ENUM`和`SET`列中不能超过`255`个唯一元素列表定义。有关这些限制的更多信息，请参见第C.10.5节“.frm文件结构所施加的限制”。

- `SET（'value1'，'value2'，...）[CHARACTER SET charset_name] [COLLATE collat​​ion_name]`

`set`。一个字符串对象，可以包含零个或多个值，每个值都必须从值列表`'value1'`，`'value2'`中选择，... `SET`值在内部表示为整数。

`SET`列最多可包含`64`个不同的成员。一个表在其被视为一个组的`ENUM`和`SET`列中可以包含不超过`255`个唯一元素列表定义。


