```mysql
SELECT
    [ALL | DISTINCT | DISTINCTROW ]
      [HIGH_PRIORITY]
      [STRAIGHT_JOIN]
      [SQL_SMALL_RESULT] [SQL_BIG_RESULT] [SQL_BUFFER_RESULT]
      [SQL_CACHE | SQL_NO_CACHE] [SQL_CALC_FOUND_ROWS]
    select_expr [, select_expr ...]
    [FROM table_references
      [PARTITION partition_list]
    [WHERE where_condition]
    [GROUP BY {col_name | expr | position}
      [ASC | DESC], ... [WITH ROLLUP]]
    [HAVING where_condition]
    [ORDER BY {col_name | expr | position}
      [ASC | DESC], ...]
    [LIMIT {[offset,] row_count | row_count OFFSET offset}]
    [PROCEDURE procedure_name(argument_list)]
    [INTO OUTFILE 'file_name'
        [CHARACTER SET charset_name]
        export_options
      | INTO DUMPFILE 'file_name'
      | INTO var_name [, var_name]]
    [FOR UPDATE | LOCK IN SHARE MODE]]
```

SELECT语句中最常用的子句是：

- 每个select_expr表示您要检索的列。 必须至少有一个select_expr。

- table_references表示从中检索行的表格。 其语法在第13.2.9.2节“JOIN语法”中描述。

- SELECT支持使用PARTITION和一系列分区或者子分区（或者两者都有）在table_reference中的表名（见第13.2.9.2节“JOIN语法”）进行明确的分区选择。 在这种情况下，只从列出的分区中选择行，并忽略表中的任何其他分区。 有关更多信息和示例，请参见第22.5节“分区选择”。

  SELECT ... PARTITION使用存储引擎（如MyISAM）执行表级锁（以及分区锁）的表只锁定由PARTITION选项指定的分区或子分区。

  有关更多信息，请参见第22.6.4节“分区和锁定”。

- WHERE子句（如果给出）表示行必须满足选择的条件。 where_condition是一个表达式，对于每个要选择的行，其计算结果为true。 如果没有WHERE子句，则语句选择所有行。

  在WHERE表达式中，除了汇总（汇总）函数，您可以使用MySQL支持的任何函数和运算符。 请参见第9.5节“表达式语法”和第12章函数和操作符。

SELECT也可以用来检索计算而不参考任何表的行。

For example:

```mysql
mysql> SELECT 1 + 1;
        -> 2
```

在没有引用表的情况下，您可以将DUAL指定为虚拟表名：

```mysql
mysql> SELECT 1 + 1 FROM DUAL;
        -> 2
```

DUAL纯粹是为了方便那些要求所有SELECT语句应该有FROM和可能的其他子句的人。 MySQL可能会忽略子句。 如果没有引用表，MySQL不需要FROM DUAL。

通常，使用的子句必须按照语法描述中显示的顺序给出。 例如，HAVING子句必须位于任何GROUP BY子句之后，且位于任何ORDER BY子句之前。 例外是INTO子句可以出现在语法描述中，也可以紧跟在select_expr列表之后。 有关INTO的更多信息，请参见第13.2.9.1节“SELECT ... INTO语法”。

以下列表提供了有关其他SELECT子句的其他信息：

- select_expr可以使用AS alias_name给别名。 别名用作表达式的列名，可以在GROUP BY，ORDER BY或HAVING子句中使用。 例如：

  ```mysql
  SELECT CONCAT(last_name,', ',first_name) AS full_name
    FROM mytable ORDER BY full_name;
  ```

  使用标识符对select_expr进行别名时，AS关键字是可选的。 前面的例子可能是这样写的：

  ```mysql
  SELECT CONCAT(last_name,', ',first_name) full_name
    FROM mytable ORDER BY full_name;
  ```

  但是，因为AS是可选的，所以如果忘记两个select_expr表达式之间的逗号，就会出现一个细微的问题：MySQL将第二个解释为别名。 例如，在以下语句中，columnb被视为别名：

  ```mysql
  SELECT columna columnb FROM mytable;
  ```

  出于这个原因，在指定列别名时，习惯于明确使用AS是一种好习惯。

  在WHERE子句中引用列别名是不允许的，因为在执行WHERE子句时可能还不能确定列值。参见B.5.4.4节“列别名问题”。

  如果您使用GROUP BY，则输出行将根据GROUP BY列进行排序，就像对同一列有ORDER BY一样。 为了避免排序该GROUP BY产生的开销，添加ORDER BY NULL：

  ```mysql
  SELECT a, COUNT(b) FROM test_table GROUP BY a ORDER BY NULL;
  ```

  依靠隐式的GROUP BY排序（也就是说，在不存在ASC或DESC标识符的情况下进行排序）已被弃用。 要生成给定的排序顺序，请对GROUP BY列使用显式的ASC或DESC指示符，或者提供ORDER BY子句。

  当您使用ORDER BY或GROUP BY对SELECT中的列进行排序时，服务器仅使用max_sort_length系统变量指示的初始字节数排序值。

- HAVING子句几乎是在物品发送到客户端之前的最后一个应用，没有优化。 （在HAVING之后应用LIMIT。）

  SQL标准要求HAVING必须仅引用GROUP BY子句中的列或聚合函数中使用的列。 但是，MySQL支持此行为的扩展，并允许HAVING引用SELECT列表中的列和外部子查询中的列。

  如果HAVING子句引用不明确的列，则会发生警告。 在下面的语句中，col2是不明确的，因为它用作别名和列名：

  ```mysql
  SELECT COUNT(col1) AS col2 FROM t GROUP BY col2 HAVING col2 = 2;
  ```

  优先考虑标准的SQL行为，所以如果在GROUP BY中使用HAVING列名，并且在输出列列表中使用HAVING列名作为别名列，则优先给予GROUP BY列中的列。

  - 不要使用HAVING来处理WHERE子句中的项目。 例如，不要写下面的内容：

    ```mysql
    SELECT col_name FROM tbl_name HAVING col_name > 0;
    ```

    写这个，而不是：

    ```mysql
    SELECT col_name FROM tbl_name WHERE col_name > 0;
    ```

  - HAVING子句可以引用集合函数，WHERE子句不能：

    ```mysql
    SELECT user, MAX(salary) FROM users
      GROUP BY user HAVING MAX(salary) > 10;
    ```

    （这在一些旧版本的MySQL中不起作用。）

  - MySQL允许重复列名。 也就是说，可以有多个具有相同名称的select_expr。 这是对标准SQL的扩展。 因为MySQL也允许GROUP BY和HAVING引用select_expr值，所以这可能会导致一个不明确的地方：

    ```mysql
    SELECT 12 AS a, a FROM t GROUP BY a;
    ```

    在这个声明中，两列都有名字a。 要确保正确的列用于分组，请为每个select_expr使用不同的名称。

