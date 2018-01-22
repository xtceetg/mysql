https://dev.mysql.com/doc/refman/5.5/en/server-logs.html

目前线上`mysql`的版本是5.5，所以先学习5.5做记录

MySQL服务器有几个日志可以帮助你找出正在发生的事情。

| Log Type                           | Information Written to Log               |
| ---------------------------------- | ---------------------------------------- |
| Error log(错误日志)                    | 在启动，运行或停止`mysqld`时遇到的问题 [**mysqld**](https://dev.mysql.com/doc/refman/5.5/en/mysqld.html) |
| General query log(一般查询日志)          | 建立客户关系和客户的声明                             |
| Binary log(二进制日志)                  | 改变数据的语句（也用于复制）                           |
| Relay log(中继日志)                    | 从复制主服务器接收的数据更改                           |
| Slow query log(慢查询日志)              | 查询花费的时间超过了[`long_query_time`](https://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html#sysvar_long_query_time)秒 |
| DDL log (metadata log)DDL日志（元数据日志） | 元数据操作由DDL语句执行                            |

### 选择一般查询和慢速查询日志输出目的地

如果启用了这些日志，则MySQL服务器可以灵活地控制输出到普通查询日志和慢速查询日志的目标。 日志条目的可能目标是日志文件或mysql数据库中的general_log和slow_log表。 可以选择任一个或两个目的地。

--log--output[=TABLE,FILE,NONE] 日志输出到文件，表 ，无 默认是FILE

general_log=[1,0] 一般查询日志 1启用，0禁用

general_log_file 指定除文件记录默认值以外的文件名

slow_query_log 慢查询

slow_query_log_file 设置慢查询文件名路径

### Binary Log

二进制日志包含描述数据库更改的“事件”，如表创建操作或对表数据的更改。 它还包含可能可能进行更改的语句的事件（例如，不匹配任何行的DELETE），除非使用基于行的日志记录。 二进制日志还包含每条语句花费多长时间更新数据的信息。 二进制日志有两个重要目的：

- 对于复制，主复制服务器上的二进制日志提供了要发送到从服务器的数据更改记录。 主服务器将包含在其二进制日志中的事件发送到其从服务器，从服务器执行这些事件以便在主服务器上进行相同的数据更改。 请参见第17.2节[“复制实现”。](https://dev.mysql.com/doc/refman/5.5/en/replication-implementation.html)
- 某些数据恢复操作需要使用二进制日志。 备份恢复后，重新执行备份后记录的二进制日志中的事件。 这些事件从备份的角度来说是最新的数据库。 [请参见第7.5节“使用二进制日志进行时间点恢复（增量）”](https://dev.mysql.com/doc/refman/5.5/en/point-in-time-recovery.html)。

二进制日志不用于SELECT或SHOW等不修改数据的语句。 要记录所有语句（例如，识别问题查询），请使用常规查询日志。 请参见“常规查询日志”。

运行启用二进制日志记录的服务器会使性能稍微降低。 但是，二进制日志的优点使您能够设置复制和还原操作，这通常会超过这个较小的性能下降。

二进制日志应该被保护，因为日志语句可能包含密码。 [请参见第6.1.2.3节“密码和记录”](https://dev.mysql.com/doc/refman/5.5/en/password-logging.html)。

以下讨论描述了影响二进制日志记录操作的一些服务器选项和变量。 有关完整列表，[请参见第17.1.3.4节“二进制日志选项和变量”](https://dev.mysql.com/doc/refman/5.5/en/replication-options-binary-log.html)。

要启用二进制日志，请使用--log-bin [= base_name]选项启动服务器。 如果没有给出base_name值，则默认名称是pid-file选项（默认为主机名称）的值，后跟-bin。 如果给出了基本名称，则服务器将该文件写入数据目录，除非基本名称以前导绝对路径名给出，以指定不同的目录。 建议您明确指定一个基本名称，而不是使用主机名称的缺省值; [请参阅第B.5.7节“MySQL中的已知问题”](https://dev.mysql.com/doc/refman/5.5/en/bugs.html)。

如果您在日志名称中提供扩展名（例如，--log-bin = base_name.extension），则会以静默方式删除并忽略扩展名。

`mysqld`将数字扩展名附加到二进制日志基本名称以生成二进制日志文件名称。 每次服务器创建一个新的日志文件时，这个数字都会增加，从而创建一系列有序的文件。 每次启动或刷新日志时，服务器都会在系列中创建一个新文件。 当前日志大小达到`max_binlog_size`后，服务器还会自动创建一个新的二进制日志文件。 如果使用大型事务，则二进制日志文件可能会大于`max_binlog_size`，因为事务是一次写入文件的，不会在文件之间进行分割。

要跟踪哪些二进制日志文件已被使用，`mysqld`还会创建一个二进制日志索引文件，其中包含所有使用的二进制日志文件的名称。 默认情况下，它与二进制日志文件具有相同的基本名称，扩展名为“.index”。 您可以使用`--log-bin-index [= file_name]`选项更改二进制日志索引文件的名称。 在`mysqld`运行时，不应该手动编辑这个文件。 这样做会混淆`mysqld`。

术语“二进制日志文件”通常表示包含数据库事件的单独编号的文件。 术语“二进制日志”共同表示编号的二进制日志文件加上索引文件的集合。

具有SUPER权限的客户端可以使用SET sql_log_bin = 0语句来禁用其自己的语句的二进制日志记录。 [请参见第5.1.5节“服务器系统变量”](https://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html)。

记录在二进制日志中的事件的格式取决于二进制记录格式。 支持三种格式类型，基于行的日志记录，基于语句的日志记录和基于混合的日志记录。 二进制日志记录格式取决于MySQL版本。 有关日志格式的一般说明，[请参见第5.4.4.1节“二进制日志记录格式”](https://dev.mysql.com/doc/refman/5.5/en/binary-log-formats.html)。 有关二进制日志格式的详细信息，[请参阅MySQL内部：二进制日志。](https://dev.mysql.com/doc/internals/en/binary-log.html)

服务器以与`--replicate-do-db`和`--replicate-ignore-db`选项相同的方式评估`--binlog-do-db和--binlog-ignore-db`选项。 有关如何完成的信息，[请参见第17.2.3.1节“数据库级复制和二进制日志记录选项的评估”](https://dev.mysql.com/doc/refman/5.5/en/replication-rules-db-options.html)。	

如果要从NDB群集复制到独立的MySQL服务器，则应该知道NDB存储引擎对某些二进制日志记录选项（包括特定于NDB的选项（如`--ndb-log-update-as-write` ）与其他存储引擎所使用的不同。 如果不纠正，这些差异可能导致主从机的二进制日志发散。 有关更多信息，请参阅从NDB到其他存储引擎的复制。 特别是，如果您在从服务器上使用非事务性存储引擎（例如`MyISAM`），[请参阅从NDB到非事务性存储引擎的复制](https://dev.mysql.com/doc/refman/5.5/en/mysql-cluster-replication-issues.html#mysql-cluster-replication-ndb-to-nontransactional)。

默认情况下，复制从服务器不会将从复制主服务器接收到的任何数据修改写入其自己的二进制日志。 要记录这些修改，除了使用--log-bin选项（[参见第17.1.3.3节“复制从属选项和变量”](https://dev.mysql.com/doc/refman/5.5/en/replication-options-slave.html)）外，还要使用`--log-slave-updates`选项启动从属。 当一个从设备也作为链接复制中的其他从设备的主设备时，这是完成的。

您可以使用RESET MASTER语句或其中的一部分使用PURGE BINARY LOGS删除所有二进制日志文件。 [请参见第13.7.6.6节“RESET语法”](https://dev.mysql.com/doc/refman/5.5/en/reset.html)和[第13.4.1.1节“PURGE BINARY LOGS语法”](https://dev.mysql.com/doc/refman/5.5/en/purge-binary-logs.html)。

如果您正在使用复制，则不应删除主服务器上的旧二进制日志文件，直到您确定没有从服务器仍然需要使用它们。 例如，如果您的从站运行超过三天，一天一次，您可以在主服务器上执行`mysqladmin flush-logs`，然后删除超过三天的任何日志。 您可以手动删除这些文件，但最好使用`PURGE BINARY LOGS`，它也可以安全地为您更新二进制日志索引文件（并且可以使用日期参数）。 [请参见第13.4.1.1节“PURGE BINARY LOGS语法”](https://dev.mysql.com/doc/refman/5.5/en/purge-binary-logs.html)。

您可以使用[mysqlbinlog](https://dev.mysql.com/doc/refman/5.5/en/mysqlbinlog.html)实用程序显示二进制日志文件的内容。 当您想要重新处理日志中的语句以进行恢复操作时，这可能很有用。 例如，您可以从二进制日志中更新MySQL服务器，如下所示：

```
shell> mysqlbinlog log_file | mysql -h server_name
```

`mysqlbinlog`也可以用来显示复制从属中继日志文件的内容，因为它们使用与二进制日志文件相同的格式写入。 有关`mysqlbinlog`实用程序以及如何使用它的更多信息，[请参见第4.6.7节“mysqlbinlog - 处理二进制日志文件的实用程序”](https://dev.mysql.com/doc/refman/5.5/en/mysqlbinlog.html)。 有关二进制日志和恢复操作的更多信息，请参见第7.5节“使用二进制日志进行时间点恢复（增量）”。

二进制日志记录是在语句或事务完成后，但在任何锁释放或任何提交完成之前立即完成的。 这可确保日志以提交顺序进行记录。

非事务表的更新在执行后立即存储在二进制日志中。

在未提交的事务中，更改事务表（例如InnoDB表）的所有更新（UPDATE，DELETE或INSERT）都将被缓存，直到服务器接收到COMMIT语句。 此时，mysqld在执行COMMIT之前将整个事务写入二进制日志。

对非事务表的修改无法回滚。 如果回滚的事务包括对非事务表的修改，则最后使用ROLLBACK语句记录整个事务，以确保对这些表的修改得到复制。

当处理事务的线程开始时，它将缓冲区语句分配给`binlog_cache_size`。 如果一个语句比这个大，那么线程将打开一个临时文件来存储事务。 线程结束时临时文件被删除。

可以使用`max_binlog_cache_size`系统变量（默认4GB，也是最大值）来限制用于缓存多语句事务的总大小。 如果事务大于这么多字节，它将失败并回滚。 最小值是4096。

如果您正在使用二进制日志和基于行的日志记录，并发插入将转换为正常插入的CREATE ... SELECT或INSERT ... SELECT语句。 这样做是为了确保您可以通过在备份操作期间应用日志来重新创建表的精确副本。 如果您使用基于语句的日志记录，则原始语句将写入日志。

二进制日志格式有一些已知的限制，可能会影响从备份恢复。 [请参见第17.4.1节“复制功能和问题”](https://dev.mysql.com/doc/refman/5.5/en/replication-features.html)。

存储程序的二进制日志记录按[第20.7节“存储程序的二进制日志记录”中所述完成](https://dev.mysql.com/doc/refman/5.5/en/stored-programs-logging.html)。

请注意，由于复制的增强，二进制日志格式在MySQL 5.5与以前版本的MySQL中有所不同。 请参见[第17.4.2节“MySQL版本之间的复制兼容性”](https://dev.mysql.com/doc/refman/5.5/en/replication-compatibility.html)。

写入二进制日志文件和二进制日志索引文件的处理方式与写入`MyISAM`表的方式相同。 请参见[第B.5.3.4节“MySQL如何处理整个磁盘”](https://dev.mysql.com/doc/refman/5.5/en/full-disk.html)。

默认情况下，二进制日志在每次写入时都不会同步到磁盘。所以如果操作系统或者机器（不仅仅是MySQL服务器）崩溃，那么二进制日志的最后一个语句就有可能丢失。为了防止这种情况发生，可以使用`sync_binlog`系统变量在每次N写入二进制日志后使二进制日志同步到磁盘。[请参见第5.1.5节“服务器系统变量”](https://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html)。 1是`sync_binlog`最安全的值，也是最慢的。即使将`sync_binlog`设置为1，在发生崩溃的情况下，表内容和二进制日志内容之间仍有可能出现不一致。例如，如果您使用的是`InnoDB`表，并且MySQL服务器处理COMMIT语句，则会将整个事务写入二进制日志，然后将此事务提交到`InnoDB`。如果服务器在这两个操作之间崩溃，则事务在重新启动时由`InnoDB`回退，但仍然存在于二进制日志中。为了解决这个问题，你应该把`--innodb_support_xa`设置为1.虽然这个选项与`InnoDB`中对XA事务的支持有关，但是它也确保了二进制日志和`InnoDB`数据文件的同步。

为了提供更高的安全性，MySQL服务器还应配置为在每次事务时将二进制日志和`InnoDB`日志同步到磁盘。 `InnoDB`日志默认是同步的，`sync_binlog` = 1可以用来同步二进制日志。这个选项的作用是在崩溃后重新启动，在事务回滚之后，MySQL服务器扫描最新的二进制日志文件以收集事务`xid`值并计算二进制日志文件中的最后一个有效位置。然后，MySQL服务器告诉`InnoDB`完成所有已成功写入二进制日志的事务，并将二进制日志截断到最后一个有效位置。这确保了二进制日志反映了`InnoDB`表的确切数据，所以从服务器与主服务器保持同步（没有接收到已经回滚的语句）。

如果MySQL服务器在崩溃恢复时发现二进制日志比它本来的短，它至少缺少一个成功提交的`InnoDB`事务。这应该不会发生，如果`sync_binlog` = 1，并且磁盘/文件系统在请求（某些时不会）时执行实际同步，则服务器会打印一条错误消息。二进制日志file_name比预期的大小要短。在这种情况下，这个二进制日志不正确，应该从主数据的新快照重新开始复制。

在解析二进制日志时，下列系统变量的会话值被写入到二进制日志中，并由复制从服务器记录：

- [`sql_mode`](https://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html#sysvar_sql_mode) (除非NO_DIR_IN_CREATE模式不被复制; [请参见第17.4.1.38节“复制和变量”](https://dev.mysql.com/doc/refman/5.5/en/replication-features-variables.html))
- [`foreign_key_checks`](https://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html#sysvar_foreign_key_checks)
- [`unique_checks`](https://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html#sysvar_unique_checks)
- [`character_set_client`](https://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html#sysvar_character_set_client)
- [`collation_connection`](https://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html#sysvar_collation_connection)
- [`collation_database`](https://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html#sysvar_collation_database)
- [`collation_server`](https://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html#sysvar_collation_server)
- [`sql_auto_is_null`](https://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html#sysvar_sql_auto_is_null)

#### Binary Logging Formats

服务器使用多种记录格式在二进制日志中记录信息。 确切的格式取决于正在使用的MySQL版本。 有三种日志格式：

- MySQL中的复制功能最初是基于从主机到从机的SQL语句的传播。 这被称为基于语句的日志记录。 您可以通过使用-`-binlog-format` = STATEMENT启动服务器来使用此格式。
- 在基于行的日志记录中，主服务器将事件写入二进制日志，指出单个表行如何受到影响。 您可以通过以`--binlog-format` = ROW启动服务器来使用基于行的日志记录。
- 第三个选项也是可用的：混合记录。 对于混合日志记录，默认情况下使用基于语句的日志记录，但是在某些情况下，日志记录模式将自动切换为基于行的，如下所述。 您可以通过使用`--binlog-format` = MIXED选项启动`mysqld`来显式使用混合日志记录。

在MySQL 5.5中，默认的二进制日志记录格式是STATEMENT。

日志格式也可以由正在使用的存储引擎设置或限制。 这有助于消除在使用不同存储引擎的主服务器和从服务器之间复制某些语句时的问题。

使用基于语句的复制，复制非确定性语句可能存在问题。 在决定给定语句是否对基于语句的复制是安全的时，MySQL决定它是否可以保证可以使用基于语句的日志记录来复制语句。 如果MySQL不能做出这样的保证，它会将该语句标记为可能不可靠并发出警告，Statement可能不是安全的登录语句格式。

您可以通过使用MySQL的基于行的复制来避免这些问题。

#### Setting The Binary Log Format

您可以通过使用`--binlog-format` = type启动MySQL服务器来明确地选择二进制日志记录格式。 支持的类型值是：

- `STATEMENT` 基于语句的日志记录。
- `ROW` 基于行日志记录。
- `MIXED` 基于混合格式日志记录。

在MySQL 5.5中，默认的二进制日志记录格式是STATEMENT。 这包括MySQL NDB集群7.2.1和更高版本的MySQL NDB集群7.2发行版，这些发行版基于MySQL 5.5。

日志格式也可以在运行时切换。 设置`binlog_format`系统变量的全局值，以指定更改后连接的客户端的格式：

```mysql
mysql> SET GLOBAL binlog_format = 'STATEMENT';
mysql> SET GLOBAL binlog_format = 'ROW';
mysql> SET GLOBAL binlog_format = 'MIXED';
```

单个客户端可以通过设置`binlog_format`的会话值来控制自己的语句的日志格式：

```mysql
mysql> SET SESSION binlog_format = 'STATEMENT';
mysql> SET SESSION binlog_format = 'ROW';
mysql> SET SESSION binlog_format = 'MIXED';
```

> 注意
> 每个MySQL服务器都可以设置自己的和唯一的二进制日志记录格式（无论是使用全局还是会话范围设置`binlog_format`）。 这意味着更改复制主服务器上的日志格式不会导致从服务器更改其日志记录格式以匹配。 （使用STATEMENT模式时，不会复制`binlog_format`系统变量;使用MIXED或ROW日志记录模式时，它将被复制，但被从属设备忽略）。在复制正在进行时更改主站上的二进制记录格式，或者不会更改 在从服务器上它可能导致复制失败，如错误执行行事件：'不能执行语句：不可能写入二进制日志，因为语句是行格式和BINLOG_FORMAT = STATEMENT。

要更改全局或会话`binlog_format`值，您必须具有SUPER权限。

客户端可能需要按每个会话设置二进制日志记录有几个原因：

- 会对数据库进行很多较小更改的会话可能需要使用基于行的日志记录。
- 执行与WHERE子句中的许多行匹配的更新的会话可能需要使用基于语句的日志记录，因为记录几条语句比多行记录更有效。
- 某些语句需要在主服务器上执行很多时间，但只会导致修改几行。 因此使用基于行的日志记录复制它们可能是有益的。

在运行时无法切换复制格式的情况有例外：

- 从存储的函数或触发器中
- 如果NDBCLUSTER存储引擎已启用
- 如果会话当前处于基于行的复制模式并且已打开临时表

试图在这些情况下切换格式会导致错误。

如果使用的是`InnoDB`表，事务隔离级别是READ COMMITTED或READ UNCOMMITTED，则只能使用基于行的日志记录。可以将日志记录格式更改为STATEMENT，但是在运行时会这样做，导致错误很快，因为`InnoDB`不能再执行插入操作。

当存在任何临时表时，不建议在运行时切换复制格式，因为只有在使用基于语句的复制时记录临时表，而使用基于行的复制时则不记录它们。通过混合复制，通常会记录临时表;用户定义函数（UDF）和UUID（）函数发生异常。

将二进制日志格式设置为ROW时，会使用基于行的格式将许多更改写入二进制日志。但是，有些更改仍然使用基于语句的格式。示例包括所有DDL（数据定义语言）语句，如CREATE TABLE，ALTER TABLE或DROP TABLE。

`--binlog-row-event-max-size`选项可用于能够进行基于行复制的服务器。行的大小以字节为单位存储在二进制日志中，不超过此选项的值。该值必须是256的倍数。默认值是1024。

> 警告
> 当使用基于语句的日志记录进行复制时，如果语句的设计方式使得数据修改不具有确定性，则主服务器和从服务器上的数据可能会有所不同; 也就是说，这是留给查询优化器的意愿的。 一般来说，即使在复制之外，这也不是一个好的做法。 有关此问题的详细说明，[请参见第B.5.7节“MySQL中的已知问题”](https://dev.mysql.com/doc/refman/5.5/en/bugs.html)。

#### Mixed Binary Logging Format

- 当一个DML语句更新一个NDBCLUSTER表时。

- 当一个函数包含UUID（）时。

- 当一个或多个具有AUTO_INCREMENT列的表被更新并且触发器或存储函数被调用时。 像所有其他不安全的语句一样，如果`binlog_format = STATEMENT`，则会生成警告。

  有关更多信息，[请参见第17.4.1.1节“复制和AUTO_INCREMENT”](https://dev.mysql.com/doc/refman/5.5/en/replication-features-auto-increment.html)。

- 当执行任何INSERT DELAYED时。

- 当视图的主体需要基于行的复制时，创建视图的语句也使用它。 例如，创建视图的语句使用UUID（）函数时会发生这种情况。

- 涉及到UDF的调用时。

- 如果语句按行记录，并且执行该语句的会话具有任何临时表，则按行记录将用于所有后续语句（除了访问临时表的那些语句之外），直到删除该会话使用的所有临时表为止。

  无论是否实际记录临时表，情况都是如此。

  临时表不能使用基于行的格式进行记录; 因此，一旦使用基于行的日志记录，使用该表的所有后续语句都是不安全的。 服务器通过将在会话期间执行的所有语句视为不安全，直到会话不再保存任何临时表，来近似这种情况。

- 当使用[`FOUND_ROWS（）`](https://dev.mysql.com/doc/refman/5.5/en/information-functions.html#function_found-rows)或[`ROW_COUNT（）`](https://dev.mysql.com/doc/refman/5.5/en/information-functions.html#function_row-count)时。 （错误＃12092，错误＃30244）

- 当使用 [`FOUND_ROWS()`](https://dev.mysql.com/doc/refman/5.5/en/information-functions.html#function_found-rows) 或 [`ROW_COUNT()`](https://dev.mysql.com/doc/refman/5.5/en/information-functions.html#function_row-count) 时。 (Bug #12092, Bug #30244)

- When [`USER()`](https://dev.mysql.com/doc/refman/5.5/en/information-functions.html#function_user), [`CURRENT_USER()`](https://dev.mysql.com/doc/refman/5.5/en/information-functions.html#function_current-user), or [`CURRENT_USER`](https://dev.mysql.com/doc/refman/5.5/en/information-functions.html#function_current-user) is used. (Bug #28086)

- 当一个语句引用一个或多个系统变量时。 （Bug＃31168）

  例外。 以下系统变量与会话作用域（仅）一起使用时，不会导致记录格式切换：

  - [`auto_increment_increment`](https://dev.mysql.com/doc/refman/5.5/en/replication-options-master.html#sysvar_auto_increment_increment)
  - [`auto_increment_offset`](https://dev.mysql.com/doc/refman/5.5/en/replication-options-master.html#sysvar_auto_increment_offset)
  - [`character_set_client`](https://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html#sysvar_character_set_client)
  - [`character_set_connection`](https://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html#sysvar_character_set_connection)
  - [`character_set_database`](https://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html#sysvar_character_set_database)
  - [`character_set_server`](https://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html#sysvar_character_set_server)
  - [`collation_connection`](https://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html#sysvar_collation_connection)
  - [`collation_database`](https://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html#sysvar_collation_database)
  - [`collation_server`](https://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html#sysvar_collation_server)
  - [`foreign_key_checks`](https://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html#sysvar_foreign_key_checks)
  - [`identity`](https://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html#sysvar_identity)
  - [`last_insert_id`](https://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html#sysvar_last_insert_id)
  - [`lc_time_names`](https://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html#sysvar_lc_time_names)
  - [`pseudo_thread_id`](https://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html#sysvar_pseudo_thread_id)
  - [`sql_auto_is_null`](https://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html#sysvar_sql_auto_is_null)
  - [`time_zone`](https://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html#sysvar_time_zone)
  - [`timestamp`](https://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html#sysvar_timestamp)
  - [`unique_checks`](https://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html#sysvar_unique_checks)

  有关确定系统变量作用域的信息，[请参见第5.1.6节“使用系统变量”](https://dev.mysql.com/doc/refman/5.5/en/using-system-variables.html)。

  有关复制如何处理`sql_mode`的信息，[请参见第17.4.1.38节“复制和变量”](https://dev.mysql.com/doc/refman/5.5/en/replication-features-variables.html)。

- 当涉及的其中一个表是`mysql`数据库中的日志表。

- 当使用LOAD_FILE（）函数时。 （错误＃39701）

  > 注意
  > 如果您尝试使用基于语句的日志记录执行语句，则应该使用基于行的日志记录来生成警告。 该警告显示在客户端（在SHOW WARNINGS的输出中）以及`mysqld`错误日志中。 每次执行这样的语句时，都会向SHOW WARNINGS表中添加警告。 但是，只有为每个客户机会话生成警告的第一条语句才会写入错误日志，以防止日志溢出。

除了上面的决定之外，单独的引擎还可以确定在更新表中的信息时使用的记录格式。 单个引擎的记录功能可以定义如下：

- 如果一个引擎支持基于行的日志记录，那么这个引擎可以说是行日志记录功能。
- 如果一个引擎支持基于语句的日志记录，则该引擎被认为是语句记录功能。

给定的存储引擎可以支持一种或两种日志格式。 下表列出了每个引擎支持的格式。

| Storage Engine                           | Row Logging Supported | Statement Logging Supported              |
| ---------------------------------------- | --------------------- | ---------------------------------------- |
| `ARCHIVE`                                | Yes                   | Yes                                      |
| `BLACKHOLE`                              | Yes                   | Yes                                      |
| `CSV`                                    | Yes                   | Yes                                      |
| `EXAMPLE`                                | Yes                   | No                                       |
| `FEDERATED`                              | Yes                   | Yes                                      |
| `HEAP`                                   | Yes                   | Yes                                      |
| `InnoDB`                                 | Yes                   | 当事务隔离级别是`REPEATABLE READ`或`SERIALIZABLE`时，可以。 没有其他的。 |
| `MyISAM`                                 | Yes                   | Yes                                      |
| `MERGE`                                  | Yes                   | Yes                                      |
| [`NDBCLUSTER`](https://dev.mysql.com/doc/refman/5.5/en/mysql-cluster.html) | Yes                   | No                                       |

是否要记录语句，并根据语句的类型（安全，不安全或二进制注入）确定要使用的记录模式，二进制记录格式（STATEMENT，ROW或MIXED）以及日志记录功能 存储引擎（语句能力，行能力，两者兼得或不兼容）。 （二进制注入是指记录必须使用ROW格式记录的更改。）

报表可以记录或不报警; 不会记录失败的语句，但会在日志中生成错误。 这在下面的决策表中显示，其中SLC代表“有语句记录能力”，而RLC代表“有行记录能力”。



| Condition     | Action                                   |      |      |                                          |             |
| ------------- | ---------------------------------------- | ---- | ---- | ---------------------------------------- | ----------- |
| Type          | [`binlog_format`](https://dev.mysql.com/doc/refman/5.5/en/replication-options-binary-log.html#sysvar_binlog_format) | SLC  | RLC  | Error / Warning                          | Logged as   |
| *             | `*`                                      | No   | No   | 错误：无法执行语句：由于至少涉及一个行不可行和不能声明的引擎，因此二进制日志记录是不可能的。 | `-`         |
| Safe          | `STATEMENT`                              | Yes  | No   | -                                        | `STATEMENT` |
| Safe          | `MIXED`                                  | Yes  | No   | -                                        | `STATEMENT` |
| Safe          | `ROW`                                    | Yes  | No   | 错误：无法执行语句：自BINLOG_FORMAT = ROW以来，无法进行二进制日志记录，并且至少有一个表使用的存储引擎无法进行基于行的日志记录。 | `-`         |
| Unsafe        | `STATEMENT`                              | Yes  | No   | 警告：由于BINLOG_FORMAT = STATEMENT，所以不安全语句以声明格式被`binlogged` | `STATEMENT` |
| Unsafe        | `MIXED`                                  | Yes  | No   | 错误：无法执行语句：即使BINLOG_FORMAT = MIXED，存储引擎仅限于基于语句的日志记录，也不可能执行不安全语句的二进制日志记录。 | `-`         |
| Unsafe        | `ROW`                                    | Yes  | No   | 错误：无法执行语句：自BINLOG_FORMAT = ROW以来，无法进行二进制日志记录，并且至少有一个表使用的存储引擎无法进行基于行的日志记录。 | -           |
| Row Injection | `STATEMENT`                              | Yes  | No   | 错误：无法执行行注入：二进制日志记录是不可能的，因为至少有一个表使用不能进行基于行记录的存储引擎。 | -           |
| Row Injection | `MIXED`                                  | Yes  | No   | 错误：无法执行行注入：二进制日志记录是不可能的，因为至少有一个表使用不能进行基于行记录的存储引擎。 | -           |
| Row Injection | `ROW`                                    | Yes  | No   | 错误：无法执行行注入：二进制日志记录是不可能的，因为至少有一个表使用不能进行基于行记录的存储引擎。 | -           |
| Safe          | `STATEMENT`                              | No   | Yes  | 错误：无法执行语句：因为BINLOG_FORMAT = STATEMENT，所以不可能执行二进制日志记录，并且至少有一个表使用不支持基于语句的日志记录的存储引擎。 | `-`         |
| Safe          | `MIXED`                                  | No   | Yes  | -                                        | `ROW`       |
| Safe          | `ROW`                                    | No   | Yes  | -                                        | `ROW`       |
| Unsafe        | `STATEMENT`                              | No   | Yes  | 错误：无法执行语句：因为BINLOG_FORMAT = STATEMENT，所以不可能执行二进制日志记录，并且至少有一个表使用不支持基于语句的日志记录的存储引擎。 | -           |
| Unsafe        | `MIXED`                                  | No   | Yes  | -                                        | `ROW`       |
| Unsafe        | `ROW`                                    | No   | Yes  | -                                        | `ROW`       |
| Row Injection | `STATEMENT`                              | No   | Yes  | 错误：无法执行行注入：因为BINLOG_FORMAT = STATEMENT，所以无法进行二进制日志记录。 | `-`         |
| Row Injection | `MIXED`                                  | No   | Yes  | -                                        | `ROW`       |
| Row Injection | `ROW`                                    | No   | Yes  | -                                        | `ROW`       |
| Safe          | `STATEMENT`                              | Yes  | Yes  | -                                        | `STATEMENT` |
| Safe          | `MIXED`                                  | Yes  | Yes  | -                                        | `STATEMENT` |
| Safe          | `ROW`                                    | Yes  | Yes  | -                                        | `ROW`       |
| Unsafe        | `STATEMENT`                              | Yes  | Yes  | 警告：自从BINLOG_FORMAT = STATEMENT以来，语句格式中的不安全语句被`binlogged`。 | `STATEMENT` |
| Unsafe        | `MIXED`                                  | Yes  | Yes  | -                                        | `ROW`       |
| Unsafe        | `ROW`                                    | Yes  | Yes  | -                                        | `ROW`       |
| Row Injection | `STATEMENT`                              | Yes  | Yes  | 错误：无法执行行注入：因为BINLOG_FORMAT = STATEMENT，所以无法进行二进制日志记录。 | -           |
| Row Injection | `MIXED`                                  | Yes  | Yes  | -                                        | `ROW`       |
| Row Injection | `ROW`                                    | Yes  | Yes  | -                                        | `ROW`       |

处理MySQL 5.5.2及更早版本中的混合格式日志。 由于Bug＃39934的修复，二进制日志的决策过程在MySQL 5.5.3中发生了变化。 在MySQL 5.5.3之前，在确定要使用的日志记录模式时，将受事件影响的所有表的能力合并，然后根据这些规则标记受影响的表集：

- 如果表是行记录功能但不具有语句记录功能，则将一组表定义为限制行记录。
- 一组表被定义为限制语句日志记录，如果这些表是语句记录能力但不能行记录能力。

一旦语句所需的可能日志记录格式的确定完成，就将其与当前`binlog_format`设置进行比较。 下面的表在MySQL 5.5.2和更早的版本中用来决定如何在二进制日志中记录信息，或者如果合适的话，是否引发错误。 在表中，安全操作被定义为确定性操作。

在MySQL 5.5.2和更早版本中，有几条规则决定语句是否是确定性的，如下表所示，其中SLR代表“限制语句记录”，RLR代表“限制行记录”。 语句是声明日志记录受限，如果它访问的一个或多个表不能记录行。 类似地，如果语句访问的任何表不具有语句记录功能，则语句是行记录受限的。

| Condition   | Action                                   |      |      |         |             |
| ----------- | ---------------------------------------- | ---- | ---- | ------- | ----------- |
| Safe/unsafe | [`binlog_format`](https://dev.mysql.com/doc/refman/5.5/en/replication-options-binary-log.html#sysvar_binlog_format) | SLR  | RLR  | 错误/警告   | Logged as   |
| Safe        | `STATEMENT`                              | Yes  | Yes  | 错误：不可记录 |             |
| Safe        | `STATEMENT`                              | Yes  | No   |         | `STATEMENT` |
| Safe        | `STATEMENT`                              | No   | Yes  | 错误：不可记录 |             |
| Safe        | `STATEMENT`                              | No   | No   |         | `STATEMENT` |
| Safe        | `MIXED`                                  | Yes  | Yes  | 错误：不可记录 |             |
| Safe        | `MIXED`                                  | Yes  | No   |         | `STATEMENT` |
| Safe        | `MIXED`                                  | No   | Yes  |         | `ROW`       |
| Safe        | `MIXED`                                  | No   | No   |         | `STATEMENT` |
| Safe        | `ROW`                                    | Yes  | Yes  | 错误：不可记录 |             |
| Safe        | `ROW`                                    | Yes  | No   | 错误：不可记录 |             |
| Safe        | `ROW`                                    | No   | Yes  |         | `ROW`       |
| Safe        | `ROW`                                    | No   | No   |         | `ROW`       |
| Unsafe      | `STATEMENT`                              | Yes  | Yes  | 错误：不可记录 |             |
| Unsafe      | `STATEMENT`                              | Yes  | No   | 警告：不安全  | `STATEMENT` |
| Unsafe      | `STATEMENT`                              | No   | Yes  | 错误：不可记录 |             |
| Unsafe      | `STATEMENT`                              | No   | No   | 警告：不安全  | `STATEMENT` |
| Unsafe      | `MIXED`                                  | Yes  | Yes  | 错误：不可记录 |             |
| Unsafe      | `MIXED`                                  | Yes  | No   | 错误：不可记录 |             |
| Unsafe      | `MIXED`                                  | No   | Yes  |         | `ROW`       |
| Unsafe      | `MIXED`                                  | No   | No   |         | `ROW`       |
| Unsafe      | `ROW`                                    | Yes  | Yes  | 错误：不可记录 |             |
| Unsafe      | `ROW`                                    | Yes  | No   | 错误：不可记录 |             |
| Unsafe      | `ROW`                                    | No   | Yes  |         | `ROW`       |
| Unsafe      | `ROW`                                    | No   | No   |         | `ROW`       |

在所有的MySQL 5.5版本中，当通过确定产生警告时，产生标准的MySQL警告（并且可以使用SHOW WARNINGS）。 这些信息也被写入到`mysqld`错误日志中。 记录每个客户端连接的每个错误实例只有一个错误，以防止日志溢出。 日志消息包含尝试的SQL语句。

如果从服务器启动时启用了--log-warnings，则从设备将消息打印到错误日志中，以提供有关其状态的信息，例如二进制日志和中继日志坐标，以便在开始工作时切换到另一个日志 中继日志，断开连接后重新连接等等。

#### 日志格式更改为`mysql`数据库表

`mysql`数据库中授权表的内容可以直接修改（例如，使用INSERT或DELETE）或间接修改（例如使用GRANT或CREATE USER）。影响`mysql`数据库表的语句使用以下规则写入到二进制日志中：

- 根据`binlog_format`系统变量的设置，将直接更改`mysql`数据库表中数据的数据操作语句。这涉及到`INSERT，UPDATE，DELETE，REPLACE，DO，LOAD DATA INFILE，SELECT`和`TRUNCATE TABLE`等语句。
- 不管`binlog_format`的值如何，间接改变`mysql`数据库的语句都被记录为语句。这与`GRANT，REVOKE，SET PASSWORD，RENAME USER，CREATE`（除`CREATE TABLE ... SELECT`之外的所有表单），`ALTER`（所有表单）和`DROP`（所有表单）之类的语句有关。

`CREATE TABLE ... SELECT`是数据定义和数据操作的组合。 `CREATE TABLE`部分使用语句格式进行记录，`SELECT`部分根据`binlog_format`的值进行记录。