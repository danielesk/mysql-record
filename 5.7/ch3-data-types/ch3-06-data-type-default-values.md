# 6.1 数据类型默认值

## 6.1.1 显式默认值的处理( Handling of Explicit Defaults )

数据类型规范中的`DEFAULT`值子句显式指示列的默认值。

例子：

```mysql
CREATE TABLE t1 (
i INT DEFAULT -1,
c VARCHAR(10) DEFAULT '',
price DOUBLE(16,2) DEFAULT '0.00'
);
```

`SERIAL DEFAULT VALUE`是一个特例。 在整数列的定义中，它是`NOT NULL AUTO_INCREMENT UNIQUE`的别名。

除了一个例外，`DEFAULT`子句中指定的默认值必须是文字常量; 它不能是一个功能或表达。 这意味着，例如，您不能将日期列的默认值设置为函数的值，例如`NOW()`或`CURRENT_DATE`。 例外情况是，对于`TIMESTAMP`和`DATETIME`列，您可以将`CURRENT_TIMESTAMP`指定为默认值。 请参见第11.3.5节“TIMESTAMP和DATETIME的自动初始化和更新”。

无法为`BLOB`，`TEXT`，`GEOMETRY`和`JSON`数据类型分配默认值。

## 6.1.2 隐式默认值的处理( Handling of Implicit Defaults )

如果数据类型规范没有包含显式`DEFAULT`，MySQL将按照以下方式确定默认值:

如果列可以接受`NULL`作为值，则使用显式的`DEFAULT NULL`子句定义列。

如果列不能将`NULL`作为值，则MySQL定义没有显式`DEFAULT`子句的列。 例外：如果列被定义为`PRIMARY KEY`的一部分但未显式为`NOT NULL`，则MySQL将其创建为`NOT NULL`列（因为PRIMARY KEY列必须为`NOT NULL`）。

对于没有显式默认子句的`NOT NULL`列的数据输入，如果`INSERT`或`REPLACE`语句不包含该列的值，或者`UPDATE`语句将该列设置为`NULL`, MySQL将根据当时有效的SQL模式处理该列：

- 如果启用了严格的SQL模式，则事务表会发生错误，并且会回滚该语句。 对于非事务性表，会发生错误，但如果多行语句的第二行或后续行发生这种情况，则会插入前面的行。

- 如果未启用严格模式，MySQL会将列设置为列数据类型的隐式默认值。

假设表`t`定义如下：

```mysql
CREATE TABLE t (i INT NOT NULL);
```

在本例中，`i`没有显式默认值，因此在严格模式下，下面的每个语句都会产生一个错误，并且没有插入任何行。当不使用严格模式时，只有第三条语句产生错误;前两个语句插入了隐式默认值，但是第三个语句失败了，因为`default (i)`不能生成值:

```mysql
INSERT INTO t VALUES();
INSERT INTO t VALUES(DEFAULT);
INSERT INTO t VALUES(DEFAULT(i));
```

对于给定的表，`SHOW CREATE TABLE`语句显示哪些列具有显式`DEFAULT`子句。

隐式默认定义如下:

- 对于数字类型，缺省值为`0`，但对于使用`AUTO_INCREMENT`属性声明的整数或浮点类型，缺省值是序列中的下一个值。

- 对于`TIMESTAMP`以外的日期和时间类型，默认值为该类型的相应“零”值。 如果启用了`explicit_defaults_for_timestamp`系统变量，则`TIMESTAMP`也是如此（请参见第5.1.7节“服务器系统变量”）。 否则，对于表中的第一个`TIMESTAMP`列，默认值为当前日期和时间。 请参见第11.3节“日期和时间类型”。

- 对于`ENUM`以外的字符串类型，默认值为空字符串。 对于`ENUM`，默认值是第一个枚举值。





























