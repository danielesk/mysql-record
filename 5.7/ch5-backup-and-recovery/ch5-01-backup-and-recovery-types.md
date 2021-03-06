# 5.1 Backup and Recovery Types

本节描述不同类型备份的特性。

## 5.1.1 物理(原始)备份与逻辑备份(Physical (Raw) Versus Logical Backups)

物理备份包括存储数据库内容的目录和文件的原始副本。这种类型的备份适用于大型、重要的数据库，这些数据库在出现问题时需要快速恢复。

逻辑备份保存以逻辑数据库结构（`CREATE DATABASE`,`CREATE TABLE`语句）和内容（`INSERT`语句或分隔文本文件）表示的信息。这种类型的备份适用于较小的数据量，您可以在其中编辑数据值或表结构，或者在不同的计算机体系结构上重新创建数据。

物理备份方法具有以下特点：

- 备份由数据库目录和文件的精确副本组成。通常这是MySQL数据目录的全部或部分副本。

- 物理备份方法比逻辑备份方法更快，因为它们只涉及不转换的文件复制。

- 输出比逻辑备份更紧凑。

- 由于备份速度和紧凑性对于繁忙、重要的数据库很重要，MySQL企业备份产品执行物理备份。有关mysql企业备份产品的概述，请参阅第29.2节“mysql企业备份概述”。

- 备份和恢复粒度范围从整个数据目录到单个文件。这可能提供也可能不提供表级粒度，具体取决于存储引擎。例如，InnoDB表可以各自位于单独的文件中，或者与其他InnoDB表共享文件存储；每个MyISAM表唯一对应一组文件。

- 除了数据库之外，备份还可以包括任何相关文件，如日志或配置文件。

- 内存表中的数据很难用这种方式备份，因为它们的内容并不存储在磁盘上。(MySQL企业备份产品有一个特性，您可以在备份期间从内存表中检索数据。



P1214








































































































































































































































