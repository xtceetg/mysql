### 服务器SQL模式

MySQL服务器可以在不同的SQL模式下运行，并且可以根据sql_mode系统变量的值对不同的客户端应用不同的模式。 DBA可以设置全局SQL模式以匹配站点服务器操作需求，每个应用程序可以根据自己的需要设置其会话SQL模式。

模式会影响MySQL支持的SQL语法以及数据验证检查的效果。 这使得在不同的环境中使用MySQL变得更容易，并且可以将MySQL与其他数据库服务器一起使用。

在使用InnoDB表时，请考虑innodb_strict_mode系统变量。 它为InnoDB表启用额外的错误检查。

**设置SQL模式**

默认的SQL模式是NO_ENGINE_SUBSTITUTION。

要在服务器启动时设置SQL模式，请在命令行中使用--sql-mode =“modes”选项，或者在my.cnf（Unix操作系统）等选项文件中使用sql-mode =“modes” .ini（Windows）。 模式是由逗号分隔的不同模式的列表。 要显式清除SQL模式，请在命令行中使用--sql-mode =“”或在选项文件中使用sql-mode =“”将其设置为空字符串。

> **注意**
>
> MySQL安装程序可能会在安装过程中配置SQL模式。 例如，mysql_install_db在基本安装目录中创建名为my.cnf的默认选项文件。 该文件包含设置SQL模式的行; 请参见第4.4.3节“mysql_install_db - 初始化MySQL数据目录”。
>
> 如果SQL模式与默认或预期不同，请检查服务器在启动时读取的选项文件中的设置。

要在运行时更改SQL模式，请使用SET语句设置全局或会话sql_mode系统变量：

```mysql
SET GLOBAL sql_mode = 'modes';
SET SESSION sql_mode = 'modes';
```

设置GLOBAL变量需要SUPER权限，并影响从此时开始连接的所有客户端的操作。 设置SESSION变量仅影响当前客户端。 每个客户端可以随时更改其会话sql_mode的值。

要确定当前的全局或会话sql_mode值，请使用以下语句：

```mysql
SELECT @@GLOBAL.sql_mode;
SELECT @@SESSION.sql_mode;
```

> - **重要**
>
> SQL模式和用户定义的分区。 创建数据并将其插入分区表后，更改服务器SQL模式可能会导致此类表的行为发生重大变化，并可能导致数据丢失或损坏。 强烈建议您一旦创建了使用用户定义分区的表格，就不要更改SQL模式。
>
> 在复制分区表时，主站和从站上不同的SQL模式也会导致问题。 为了获得最佳结果，您应始终在主服务器和从服务器上使用相同的服务器SQL模式。
>
> 有关更多信息，请参见第19.6节“分区的限制和限制”。

**最重要的SQL模式**

最重要的sql_mode值可能是这些：

- ANSI

  此模式更改语法和行为以更加符合标准SQL。 这是本节最后列出的特殊组合模式之一。

- STRICT_TRANS_TABLES

  如果某个值无法插入到事务表中，请中止该语句。 对于非事务性表，如果该值出现在单行语句或多行语句的第一行中，则中止该语句。 更多细节在本节后面给出。

  为事务性存储引擎启用严格的SQL模式，并在非事务性存储引擎可能的情况下启用。

- TRADITIONAL

  使MySQL像“传统”的SQL数据库系统一样行事。 在将不正确的值插入列时，此模式的简单描述是“给出错误而不是警告”。 这是本节最后列出的特殊组合模式之一。

  > **注意**
  >
  > 一旦发现错误，INSERT或UPDATE将立即中止。 如果您使用的是非事务性存储引擎，则这可能不是您想要的，因为在错误之前进行的数据更改可能无法回滚，从而导致“部分完成”更新。

  当本手册引用“严格模式”时，表示启用了STRICT_TRANS_TABLES或STRICT_ALL_TABLES中的一个或两个的模式。

  **SQL模式的完整列表**

  以下列表描述了所有支持的SQL模式：

  - ALLOW_INVALID_DATES

    不要执行完整的日期检查。 只检查月份是在1到12之间，日期是在1到31之间。这对于在三个不同领域获得年份，月份和日期的Web应用程序非常方便，并且您想要存储 究竟是用户插入（没有日期验证）。 此模式适用于DATE和DATETIME列。 它不适用于始终需要有效日期的TIMESTAMP列。

    服务器要求月份和日期值是合法的，而不仅仅是在1到12和1到31的范围内。 禁用严格模式后，无效日期（如“2004-04-31”）将转换为“0000-00-00”，并生成警告。 启用严格模式后，无效的日期会生成错误。 要允许这样的日期，启用ALLOW_INVALID_DATES。

  - ANSI_QUOTES

    将“标识符引用字符（如引号字符），而不是字符串引用字符。”仍然可以使用`引用标识符与启用此模式。启用ANSI_QUOTES，不能使用双引号引号字符串， 因为它被解释为一个标识符。

  - ERROR_FOR_DIVISION_BY_ZERO

    ERROR_FOR_DIVISION_BY_ZERO模式影响除零的处理，其中包括MOD（N，0）。 对于数据更改操作（INSERT，UPDATE），其效果还取决于严格SQL模式是否启用。

    - 如果这种模式没有被启用，除以零将插入NULL并且不产生警告。
    - 如果启用此模式，除零将插入NULL并产生警告。
    - 如果启用此模式和严格模式，则除以零除会产生错误，除非给出IGNORE。 对于INSERT IGNORE和UPDATE IGNORE，除以零将插入NULL并产生警告。

    对于SELECT，除以零将返回NULL。 无论是否启用严格模式，启用ERROR_FOR_DIVISION_BY_ZERO也会产生警告。

    从MySQL 5.6.17开始，不推荐使用ERROR_FOR_DIVISION_BY_ZERO，并将sql_mode的值设置为包含它将生成警告。

  - HIGH_NOT_PRECEDENCE

    NOT运算符的优先级是，诸如NOT BETWEEN b和c之类的表达式被解析为NOT（BETWEEN b AND c）。 在一些老版本的MySQL中，表达式被解析为（不是）BETWEEN b AND c。 旧的高优先级行为可以通过启用HIGH_NOT_PRECEDENCE SQL模式来获得。

    ```mysql
    mysql> SET sql_mode = '';
    mysql> SELECT NOT 1 BETWEEN -5 AND 5;
            -> 0
    mysql> SET sql_mode = 'HIGH_NOT_PRECEDENCE';
    mysql> SELECT NOT 1 BETWEEN -5 AND 5;
            -> 1
    ```

  - IGNORE_SPACE

    允许函数名和（字符）之间的空格，这使得内置的函数名被视为保留字，因此，与函数名相同的标识符必须按照第9.2节“模式对象名 “例如，因为有一个COUNT（）函数，在以下语句中将count用作表名会导致错误：

    ```mysql
    mysql> CREATE TABLE count (i INT);
    ERROR 1064 (42000): You have an error in your SQL syntax
    ```

    表名应该引用：

    ```mysql
    mysql> CREATE TABLE `count` (i INT);
    Query OK, 0 rows affected (0.00 sec)	
    ```

    IGNORE_SPACE SQL模式适用于内置函数，而不适用于用户定义的函数或存储的函数。 无论是否启用IGNORE_SPACE，总是允许在UDF或存储的函数名称后面有空格。

    有关IGNORE_SPACE的更多讨论，请参见第9.2.4节“函数名称解析和解析”。

  - NO_AUTO_CREATE_USER

    防止GRANT语句自动创建新用户，除非认证信息被指定。 该语句必须使用IDENTIFIED BY或使用IDENTIFIED WITH的身份验证插件指定非空密码。

  - NO_ENGINE_SUBSTITUTION

    默认的SQL模式。

    当语句（如CREATE TABLE或ALTER TABLE）指定已禁用或未编译的存储引擎时，控制默认存储引擎的自动替换。

    因为存储引擎在运行时可以被插入，所以不可用的引擎被以同样的方式处理：

    在禁用NO_ENGINE_SUBSTITUTION的情况下，对于CREATE TABLE，将使用默认引擎，如果所需的引擎不可用，则会发生警告。 对于ALTER TABLE，会发生警告，并且表不会被更改。

    如果启用了NO_ENGINE_SUBSTITUTION，则会发生错误，并且如果所需的引擎不可用，则不会创建或更改该表。

  - NO_AUTO_VALUE_ON_ZERO

    NO_AUTO_VALUE_ON_ZERO影响AUTO_INCREMENT列的处理。 通常，通过向其中插入NULL或0来为列生成下一个序列号。 NO_AUTO_VALUE_ON_ZERO将此行为抑制为0，以便只有NULL才会生成下一个序列号。

    如果0已被存储在表的AUTO_INCREMENT列中，则此模式可能很有用。 （顺便说一下，存储0不是推荐的做法。）例如，如果您使用mysqldump转储表并重新加载它，MySQL通常会在遇到0值时生成新的序列号，从而生成一个内容不同于 被甩的那个。 在重新加载转储文件之前启用NO_AUTO_VALUE_ON_ZERO解决了这个问题。 mysqldump现在自动在其输出中包含一个使NO_AUTO_VALUE_ON_ZERO启用的语句来避免这个问题。

  - NO_BACKSLASH_ESCAPES

    禁止使用反斜线字符（\）作为字符串中的转义字符。 启用此模式后，反斜杠将变成普通字符一样。

  - NO_DIR_IN_CREATE

    创建表格时，忽略所有INDEX DIRECTORY和DATA DIRECTORY指令。 该选项在从属复制服务器上很有用。

  - NO_FIELD_OPTIONS

    不要在SHOW CREATE TABLE的输出中打印MySQL特定的列选项。 这种模式在可移植性模式下被mysqldump使用。

  - NO_KEY_OPTIONS

    不要在SHOW CREATE TABLE的输出中打印MySQL特定的索引选项。 这种模式在可移植性模式下被mysqldump使用。

  - NO_TABLE_OPTIONS

    不要在SHOW CREATE TABLE的输出中打印MySQL特定的表选项（如ENGINE）。 这种模式在可移植性模式下被mysqldump使用。

  - NO_UNSIGNED_SUBTRACTION

    整数值之间的减法，其中一个是UNSIGNED类型，默认情况下产生一个无符号的结果。 如果结果否则会导致错误：

    ```mysql
    mysql> SET sql_mode = '';
    Query OK, 0 rows affected (0.00 sec)

    mysql> SELECT CAST(0 AS UNSIGNED) - 1;
    ERROR 1690 (22003): BIGINT UNSIGNED value is out of range in '(cast(0 as unsigned) - 1)'
    ```

    如果NO_UNSIGNED_SUBTRACTION SQL模式被启用，结果是否定的：

    ```mysql
    mysql> SET sql_mode = 'NO_UNSIGNED_SUBTRACTION';
    mysql> SELECT CAST(0 AS UNSIGNED) - 1;
    +-------------------------+
    | CAST(0 AS UNSIGNED) - 1 |
    +-------------------------+
    |                      -1 |
    +-------------------------+
    ```

    如果使用此操作的结果更新UNSIGNED整数列，则结果将被剪裁为列类型的最大值，如果启用了NO_UNSIGNED_SUBTRACTION，则剪切为0。 如果启用严格的SQL模式，则会发生错误，并且列保持不变。

    当启用NO_UNSIGNED_SUBTRACTION时，即使有任何操作数是无符号的，减法结果也是有符号的。 例如，将表t1中的列c2的类型与表t2中的列c2的类型进行比较：

    ```mysql
    mysql> SET sql_mode='';
    mysql> CREATE TABLE test (c1 BIGINT UNSIGNED NOT NULL);
    mysql> CREATE TABLE t1 SELECT c1 - 1 AS c2 FROM test;
    mysql> DESCRIBE t1;
    +-------+---------------------+------+-----+---------+-------+
    | Field | Type                | Null | Key | Default | Extra |
    +-------+---------------------+------+-----+---------+-------+
    | c2    | bigint(21) unsigned | NO   |     | 0       |       |
    +-------+---------------------+------+-----+---------+-------+

    mysql> SET sql_mode='NO_UNSIGNED_SUBTRACTION';
    mysql> CREATE TABLE t2 SELECT c1 - 1 AS c2 FROM test;
    mysql> DESCRIBE t2;
    +-------+------------+------+-----+---------+-------+
    | Field | Type       | Null | Key | Default | Extra |
    +-------+------------+------+-----+---------+-------+
    | c2    | bigint(21) | NO   |     | 0       |       |
    +-------+------------+------+-----+---------+-------+
    ```

    这意味着BIGINT UNSIGNED在所有情况下都不是100％可用的。 请参见第12.10节“演算函数和操作符”。

  - NO_ZERO_DATE

    NO_ZERO_DATE模式会影响服务器是否允许“0000-00-00”作为有效日期。 其效果也取决于是否启用严格的SQL模式。

    - 如果未启用此模式，则允许“0000-00-00”，插入不会产生警告。
    - 如果启用此模式，则允许使用“0000-00-00”，插入会产生警告。
    - 如果启用了此模式和严格模式，则不允许使用“0000-00-00”，插入会产生错误，除非给出IGNORE。 对于INSERT IGNORE和UPDATE IGNORE，“0000-00-00”是允许的，插入会产生警告。

    从MySQL 5.6.17开始，不建议使用NO_ZERO_DATE，并且将sql_mode值设置为包含它将生成警告。

  - NO_ZERO_IN_DATE

    NO_ZERO_IN_DATE模式会影响服务器是否允许年份不为零，但月份或日期部分为0的日期。（此模式会影响“2010-00-01”或“2010-01-00”等日期，但不影响日期 '0000-00-00'。要控制服务器是否允许'0000-00-00'，请使用NO_ZERO_DATE模式。）NO_ZERO_IN_DATE的效果也取决于是否启用严格的SQL模式。

    - 如果此模式未启用，则允许使用零部件的日期，插入不会产生警告。
    - 如果启用此模式，则零部件的日期被插入为'0000-00-00'并产生警告。
    - 如果启用了此模式和严格模式，则不允许带有零部件的日期，插入会产生错误，除非给出IGNORE。 对于INSERT IGNORE和UPDATE IGNORE，零部件的日期被插入为'0000-00-00'并产生一个警告。

    从MySQL 5.6.17开始，不建议使用NO_ZERO_IN_DATE，并且将sql_mode值设置为包含它会生成警告。

  - ONLY_FULL_GROUP_BY

    拒绝选择列表，HAVING条件或ORDER BY列表引用未在GROUP BY子句中命名的非聚合列的查询。

    标准SQL的MySQL扩展允许在HAVING子句中引用选择列表中的别名表达式。 启用ONLY_FULL_GROUP_BY将禁用此扩展，因此需要使用非混淆表达式来写入HAVING子句。

    有关其他讨论和示例，请参见第12.18.3节“MySQL处理GROUP BY”。

  - PAD_CHAR_TO_FULL_LENGTH

    默认情况下，尾部空格在检索时从CHAR列值中删除。 如果启用了PAD_CHAR_TO_FULL_LENGTH，则不会发生修剪，并将检索的CHAR值填充到其全长。 此模式不适用于VARCHAR列，在检索时保留了尾随空格。

    ```mysql
    mysql> CREATE TABLE t1 (c1 CHAR(10));
    Query OK, 0 rows affected (0.37 sec)

    mysql> INSERT INTO t1 (c1) VALUES('xy');
    Query OK, 1 row affected (0.01 sec)

    mysql> SET sql_mode = '';
    Query OK, 0 rows affected (0.00 sec)

    mysql> SELECT c1, CHAR_LENGTH(c1) FROM t1;
    +------+-----------------+
    | c1   | CHAR_LENGTH(c1) |
    +------+-----------------+
    | xy   |               2 |
    +------+-----------------+
    1 row in set (0.00 sec)

    mysql> SET sql_mode = 'PAD_CHAR_TO_FULL_LENGTH';
    Query OK, 0 rows affected (0.00 sec)

    mysql> SELECT c1, CHAR_LENGTH(c1) FROM t1;
    +------------+-----------------+
    | c1         | CHAR_LENGTH(c1) |
    +------------+-----------------+
    | xy         |              10 |
    +------------+-----------------+
    1 row in set (0.00 sec)		
    ```

  - PIPES_AS_CONCAT

    对待|| 作为字符串连接运算符（与CONCAT（）相同）而不是OR的同义词。

  - REAL_AS_FLOAT

    将REAL视为FLOAT的同义词。 默认情况下，MySQL将REAL作为DOUBLE的同义词。

  - STRICT_ALL_TABLES

    为所有存储引擎启用严格的SQL模式。 无效的数据值被拒绝。 有关详细信息，请参阅严格SQL模式。

  - STRICT_TRANS_TABLES

    为事务性存储引擎启用严格的SQL模式，并在非事务性存储引擎可能的情况下启用。 有关详细信息，请参阅严格SQL模式。

  **组合SQL模式**

  提供以下特殊模式作为上述列表中模式值组合的简写。

  - ANSI

    相当于REAL_AS_FLOAT，PIPES_AS_CONCAT，ANSI_QUOTES，IGNORE_SPACE。

    ANSI模式还会导致服务器返回一个查询错误，其中具有外部引用S（outer_ref）的集合函数S不能在已解析外部引用的外部查询中聚合。 这是一个这样的问题：

    ```mysql
    SELECT * FROM t1 WHERE t1.a IN (SELECT MAX(t1.b) FROM t2 WHERE ...);
    ```

    这里，MAX（t1.b）不能在外部查询中聚合，因为它出现在该查询的WHERE子句中。 标准SQL在这种情况下需要一个错误。 如果ANSI模式未启用，则服务器将以与解释S（const）相同的方式在这样的查询中处理S（outer_ref）。

    请参阅第1.7节“MySQL标准合规性”。

  - DB2

    相当于PIPES_AS_CONCAT，ANSI_QUOTES，IGNORE_SPACE，NO_KEY_OPTIONS，NO_TABLE_OPTIONS，NO_FIELD_OPTIONS。

  - MAXDB

    相当于PIPES_AS_CONCAT，ANSI_QUOTES，IGNORE_SPACE，NO_KEY_OPTIONS，NO_TABLE_OPTIONS，NO_FIELD_OPTIONS，NO_AUTO_CREATE_USER。

  - MSSQL

    相当于PIPES_AS_CONCAT，ANSI_QUOTES，IGNORE_SPACE，NO_KEY_OPTIONS，NO_TABLE_OPTIONS，NO_FIELD_OPTIONS。

  - MYSQL323

    相当于MYSQL323，HIGH_NOT_PRECEDENCE。 这意味着HIGH_NOT_PRECEDENCE加上一些特定于MYSQL323的SHOW CREATE TABLE行为：

    - TIMESTAMP列显示不包括在MySQL 4.1中引入的DEFAULT或ON UPDATE属性。
    - 字符串列显示不包括在MySQL 4.1中引入的字符集和整理属性。 对于CHAR和VARCHAR列，如果排序规则是二进制的，则将BINARY附加到列类型。
    - ENGINE = engine_name表格选项显示为TYPE = engine + name。
    - 对于MEMORY表，存储引擎显示为HEAP。

  - MYSQL40

    相当于MYSQL40，HIGH_NOT_PRECEDENCE。 这意味着HIGH_NOT_PRECEDENCE加上一些特定于MYSQL40的行为。 除了SHOW CREATE TABLE不显示HEAP作为MEMORY表的存储引擎外，它们与MYSQL323相同。

  - ORACLE

    相当于PIPES_AS_CONCAT，ANSI_QUOTES，IGNORE_SPACE，NO_KEY_OPTIONS，NO_TABLE_OPTIONS，NO_FIELD_OPTIONS，NO_AUTO_CREATE_USER。

  - POSTGRESQL

    相当于PIPES_AS_CONCAT，ANSI_QUOTES，IGNORE_SPACE，NO_KEY_OPTIONS，NO_TABLE_OPTIONS，NO_FIELD_OPTIONS。

  - TRADITIONAL

    相当于STRICT_TRANS_TABLES，STRICT_ALL_TABLES，NO_ZERO_IN_DATE，NO_ZERO_DATE，ERROR_FOR_DIVISION_BY_ZERO，NO_AUTO_CREATE_USER和NO_ENGINE_SUBSTITUTION。

  **严格的SQL模式**

  严格模式控制MySQL如何处理数据更改语句（如INSERT或UPDATE）中的无效或缺失值。 由于以下原因，值可能无效。 例如，该列可能具有错误的数据类型，或者可能超出范围。 如果要插入的新行没有包含定义中没有显式DEFAULT子句的非NULL列的值，则缺少一个值。 （对于NULL列，如果值缺失，则插入NULL。）严格模式也会影响DDL语句，如CREATE TABLE。

  如果严格模式没有生效，MySQL将插入调整后的值作为无效值或缺失值，并产生警告（参见第13.7.5.41节“SHOW WARNINGS语法”）。 在严格模式下，您可以使用INSERT IGNORE或UPDATE IGNORE来产生此行为。

  对于不改变数据的SELECT等语句，无效值将在严格模式下生成警告，而不是错误。

  从MySQL 5.6.11开始，严格模式会尝试创建超出最大密钥长度的密钥。 以前，这会导致警告并截断最大密钥长度的密钥（与严格模式未启用时相同）。

  严格模式不会影响是否检查外键约束。 foreign_key_checks可以用于这个。 （请参见第5.1.5节“服务器系统变量”。）

  如果启用了STRICT_ALL_TABLES或STRICT_TRANS_TABLES，则严格的SQL模式有效，但这些模式的效果有所不同：

  - 对于事务表，当启用STRICT_ALL_TABLES或STRICT_TRANS_TABLES时，数据更改语句中的无效值或缺失值会发生错误。 该声明被中止并回滚。
  - 对于非事务性表，如果在要插入或更新的第一行中出现错误值，则对于任一模式，行为都是相同的：语句被中止，表保持不变。 如果语句插入或修改多行，并且第二行或更后一行出现错误值，则结果取决于启用了哪个严格模式：
    - 对于STRICT_ALL_TABLES，MySQL返回一个错误，并忽略其余的行。 但是，由于先前的行已被插入或更新，所以结果是部分更新。 为了避免这种情况，可以使用单行语句，可以在不更改表的情况下中止。
    - 对于STRICT_TRANS_TABLES，MySQL将无效值转换为列的最接近的有效值并插入调整后的值。 如果缺少一个值，MySQL将插入列数据类型的隐式默认值。 无论哪种情况，MySQL都会生成警告而不是错误，并继续处理语句。 第11.6节“数据类型默认值”中介绍了隐式默认值。(如查表字段设置不为空，而且没有给默认值，那么在执行insert语句的时候会报错[Err] 1364 - Field 'content' doesn't have a default value)

  严格模式还影响日期中由零，零日期和零处理除法，结合ERROR_FOR_DIVISION_BY_ZERO，NO_ZERO_DATE和NO_ZERO_IN_DATE模式。 有关详细信息，请参阅这些模式的说明。