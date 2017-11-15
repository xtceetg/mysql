# InnoDB Locking

## Shared and Exclusive Locks(共享和独占锁定)

InnoDB在有两种类型的锁，共享（S）锁和独占（X）锁的情况下实现标准行级锁定。

- shared 共享（S）锁允许持有锁的事务读取一行。
- exclusive 独占（X）锁允许持有锁的事务更新或删除一行。

如果事务T1持有行r上的共享（S）锁，那么来自某个不同事务T2的对行r上的锁的请求将按如下方式处理：

- T2对S锁的请求可以立即被授予。 结果，T1和T2在r上持有一个S锁。
- T2对X锁的请求不能立即授予。

如果一个事务T1在行r上保存一个排它（X）锁，那么不能立即授予来自某个不同事务T2的对r上任一类型的锁的请求。 相反，事务T2必须等待事务T1释放其在行r上的锁定。

## Intention Locks (意图锁定)

InnoDB支持多粒度锁定，允许在整个表上共存行级锁和锁。 为了实现多个粒度级别的锁定，使用额外类型的锁，称为意向锁。 意图锁是InnoDB中的表级锁，用于指示该表中某一行的事务需要哪种类型的锁（共享或排除）。 在InnoDB中使用意向锁有两种类型（假设事务T已经在表t上请求指定类型的锁）：

- Intention share 意图共享（IS）：事务T意图在表t中的单个行上设置S锁。
- Intention exclusive 意图排他（IX）：事务T意图在这些行上设置X锁。

例如，SELECT ... LOCK IN SHARE MODE设置一个IS锁，SELECT ... FOR UPDATE设置一个IX锁。

意向锁定协议如下：

- 在一个事务可以在表t中的一行上获得一个S锁之前，它必须首先获得一个IS或更强的锁。
- 在一个事务可以在一行上获得一个X锁之前，它必须首先在t上获得一个IX锁。

这些规则可以通过下面的锁类型兼容性矩阵方便地总结。

|      | *X*      | *IX*       | *S*        | *IS*       |
| ---- | -------- | ---------- | ---------- | ---------- |
| *X*  | Conflict | Conflict   | Conflict   | Conflict   |
| *IX* | Conflict | Compatible | Conflict   | Compatible |
| *S*  | Conflict | Conflict   | Compatible | Compatible |
| *IS* | Conflict | Compatible | Compatible | Compatible |

如果请求事务与现有的锁定兼容，则授予锁定，但如果与现有的锁定冲突，则该锁定不会被授予。 事务一直等到冲突的现有锁被释放。 如果锁定请求与现有的锁定发生冲突，并且由于会导致死锁而无法被授予，则会发生错误。

因此，意图锁不会阻塞除了全表请求（例如，LOCK TABLES ... WRITE）之外的任何事情。 IX和IS锁的主要目的是显示某人正在锁定一行，或者要锁定表中的一行。

意图锁定的交易数据在SHOW ENGINE INNODB STATUS和InnoDB监视器输出中显示类似于以下内容：

```mysql
TABLE LOCK table `test`.`t` trx id 10080 lock mode IX
```

## Record Locks (记录锁定)

记录锁是索引记录上的锁。 例如，SELECT c1 FROM t WHERE c1 = 10 FOR UPDATE; 防止任何其他事务插入，更新或删除t.c1的值为10的行。

记录锁总是锁定索引记录，即使没有索引定义表。 对于这种情况，InnoDB创建一个隐藏的聚集索引，并使用这个索引进行记录锁定。 请参见第14.8.2.1节“集群索引和二级索引”。

记录锁定的事务数据在SHOW ENGINE INNODB STATUS和InnoDB监视器输出中显示类似于以下内容：

```mysql
RECORD LOCKS space id 58 page no 3 n bits 72 index `PRIMARY` of table `test`.`t` 
trx id 10078 lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 00000000274f; asc     'O;;
 2: len 7; hex b60000019d0110; asc        ;;
```

## Gap Locks(间隙锁)

间隙锁定是索引记录之间的间隙的锁定，也可以是最后一个索引记录之前或之后间隙的锁定。 例如，SELECT c1 FROM t WHERE c1 BETWEEN 10 and 20 FOR UPDATE; 防止其他事务向列t.c1中插入15的值，无论该列中是否已经有任何这样的值，因为该范围内的所有现有值之间的间隙被锁定。

缺口可能跨越单个索引值，多个索引值，甚至是空的。

间隙锁是性能和并发性之间的折衷的一部分，在某些事务隔离级别中使用，而不是在其他级别中使用。

对于使用唯一索引锁定行以搜索唯一行的语句，不需要使用间隙锁定。 （这不包括搜索条件仅包含多列唯一索引中的某些列的情况;在这种情况下，会发生间隙锁定）。例如，如果id列具有唯一索引，则以下语句仅使用 对于id值为100的行的索引记录锁定，其他会话是否在上述间隔中插入行并不重要：

```mysql
SELECT * FROM child WHERE id = 100;	
```

如果id没有编入索引或者有一个不唯一的索引，那么语句确实会锁定前面的差距。

这里也值得注意的是，不同的交易可以在差距上保持冲突的锁定。 例如，事务A可以在间隙上保持共享的间隙锁（间隙S锁），而事务B在同一间隙上保持独占间隙锁（间隙X锁）。 允许冲突间隙锁定的原因是，如果从索引中清除记录，则必须合并由不同事务记录保持的间隙锁定。

InnoDB中的缺口锁是“纯粹的抑制性”，这意味着它们只能阻止其他事务插入到缺口中。 它们并不妨碍不同的交易在同样的差距上进行差距锁定。 因此，间隙X锁具有与间隙S锁相同的效果。

可以显式禁用间隙锁定。 如果将事务隔离级别更改为READ COMMITTED或启用innodb_locks_unsafe_for_binlog系统变量（现在已弃用），则会发生这种情况。 在这种情况下，对于搜索和索引扫描禁用间隙锁定，仅用于外键约束检查和重复键检查。

还有使用READ COMMITTED隔离级别或启用innodb_locks_unsafe_for_binlog的其他效果。 在MySQL评估WHERE条件之后，释放对不匹配行的记录锁定。 对于UPDATE语句，InnoDB执行“半连续”读取，以便将最新的提交版本返回给MySQL，以便MySQL可以确定该行是否匹配UPDATE的WHERE条件。

## Next-Key锁

下一个键锁是索引记录上的记录锁与索引记录之前的间隙上的间隙锁的组合。

InnoDB以这样一种方式执行行级锁定，即当它搜索或扫描表索引时，它会在遇到的索引记录上设置共享锁或排它锁。 因此，行级锁实际上是索引记录锁。 索引记录上的下一个键锁定也会影响该索引记录之前的“间隙”。 也就是说，下一个键锁是一个索引记录锁，并在索引记录之前的间隙上加上一个间隙锁。 如果一个会话在索引中具有记录R上的共享或排它锁定，则另一个会话不能在索引顺序中紧接在R之前的间隙中插入新的索引记录。

假设索引包含值10,11,13和20.此索引的可能下一个键锁定涵盖以下区间，其中圆括号表示排除区间端点，方括号表示包含端点：

```mysql
(negative infinity, 10]
(10, 11]
(11, 13]
(13, 20]
(20, positive infinity)
```

对于最后一个时间间隔，下一个键锁定将索引中的最大值之上的间隙锁定，并且“上游”伪记录的值高于实际在索引中的任何值。 上确界不是真实的索引记录，所以实际上，这个下一个键锁只锁定了跟随最大索引值的间隙。

默认情况下，InnoDB以REPEATABLE READ事务隔离级别运行。 在这种情况下，InnoDB使用next-key锁进行搜索和索引扫描，防止幻像行（请参阅第14.5.4节“幻像行”）。

下一个键锁定的事务数据在SHOW ENGINE INNODB STATUS和InnoDB监视器输出中出现类似于以下内容：

```mysql
RECORD LOCKS space id 58 page no 3 n bits 72 index `PRIMARY` of table `test`.`t` 
trx id 10080 lock_mode X
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 00000000274f; asc     'O;;
 2: len 7; hex b60000019d0110; asc        ;;	
```

## Insert Intention Locks(插入意向锁)

插入意图锁定是插入行之前由INSERT操作设置的一种间隙锁定。 这个锁定以插入到同一索引间隙中的多个事务如果没有插入间隙中的相同位置时不需要等待对方的方式表示插入的意图。 假设有索引记录的值为4和7.分别尝试插入5和6的值的事务分别在获得对插入行的排它锁之前用插入意向锁来锁定4和7之间的间隔， 但不要相互阻塞，因为行是非冲突的。

以下示例演示在获取插入记录的排它锁之前，插入意向锁的事务。 这个例子涉及两个客户，A和B.

客户端A创建一个包含两个索引记录（90和102）的表，然后启动一个事务处理，对ID大于100的索引记录进行排它锁。排它锁包括记录102之前的间隔锁：

```mysql
mysql> CREATE TABLE child (id int(11) NOT NULL, PRIMARY KEY(id)) ENGINE=InnoDB;
mysql> INSERT INTO child (id) values (90),(102);

mysql> START TRANSACTION;
mysql> SELECT * FROM child WHERE id > 100 FOR UPDATE;
+-----+
| id  |
+-----+
| 102 |
+-----+
```

客户B开始一个交易，将记录插入到缺口中。 该事务在等待获得排他锁的同时获取插入意图锁。

```mysql
mysql> START TRANSACTION;
mysql> INSERT INTO child (id) VALUES (101);
```

插入意图锁定的事务数据在SHOW ENGINE INNODB STATUS和InnoDB监视器输出中显示类似于以下内容：

```mysql
RECORD LOCKS space id 31 page no 3 n bits 72 index `PRIMARY` of table `test`.`child`
trx id 8731 lock_mode X locks gap before rec insert intention waiting
Record lock, heap no 3 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 80000066; asc    f;;
 1: len 6; hex 000000002215; asc     " ;;
 2: len 7; hex 9000000172011c; asc     r  ;;...
```

## AUTO-INC Locks

AUTO-INC锁是一个特殊的表级锁，通过将事务插入带AUTO_INCREMENT列的表中。 在最简单的情况下，如果一个事务正在向表中插入值，则任何其他事务都必须等待自己插入到该表中，以便第一个事务插入的行接收连续的主键值。

innodb_autoinc_lock_mode配置选项控制用于自动增量锁定的算法。 它允许您选择如何在可预测的自动增量值序列和插入操作的最大并发之间进行权衡。

有关更多信息，请参见第14.8.1.5节“InnoDB中的AUTO_INCREMENT处理”。

## Predicate Locks for Spatial Indexes 谓词锁定空间索引

InnoDB支持包含空间列的列的SPATIAL索引（参见第11.5.8节“优化空间分析”）。

要处理涉及SPATIAL索引的操作的锁定，下一个键锁定不能很好地支持REPEATABLE READ或SERIALIZABLE事务隔离级别。 在多维数据中没有绝对的排序概念，因此不清楚哪个是“下一个”键。

为了支持具有SPATIAL索引的表的隔离级别，InnoDB使用谓词锁。 SPATIAL索引包含最小边界矩形（MBR）值，因此InnoDB通过对用于查询的MBR值设置谓词锁定来强制执行对索引的一致读取。 其他事务不能插入或修改匹配查询条件的行。

# InnoDB Transaction Model

## 事务隔离级别

事务隔离是数据库处理的基础之一。 隔离是I中的首字母ACID; 隔离级别是在多个事务同时进行更改和执行查询时，对结果的性能和可靠性，一致性和可重复性之间的平衡进行微调的设置。

InnoDB提供了SQL：1992标准描述的所有四个事务隔离级别：READ UNCOMMITTED，READ COMMITTED，REPEATABLE READ和SERIALIZABLE。 InnoDB的默认隔离级别是REPEATABLE READ。

用户可以使用SET TRANSACTION语句更改单个会话或所有后续连接的隔离级别。 要为所有连接设置服务器的默认隔离级别，请在命令行或选项文件中使用--transaction-isolation选项。 有关隔离级别和级别设置语法的详细信息，请参见第13.3.6节“SET TRANSACTION语法”。

InnoDB使用不同的锁定策略支持这里描述的每个事务隔离级别。 您可以使用默认的REPEATABLE READ级别强制执行高度一致性操作，以便在对ACID合规性非常重要的关键数据上进行操作。 或者，在批量报告等情况下，可以放宽一致性规则，即READ COMMITTED或READ UNCOMMITTED。在这种情况下，精确的一致性和可重复的结果不如最小化锁定开销的数量重要。 SERIALIZABLE强制执行比REPEATABLE READ更严格的规则，主要用于特殊情况，例如XA事务以及排除并发和死锁问题。

以下列表描述了MySQL如何支持不同的事务级别。 该列表从最常用的级别到最少使用的级别。

- REPEATABLE READ

  这是InnoDB的默认隔离级别。 在同一事务中的一致读取读取由第一次读取建立的快照。这意味着如果在同一个事务中发出几个纯的（非锁定的）SELECT语句，这些SELECT语句也是相互一致的。 请参见第14.5.2.3节“一致性非锁定读取”。

  对于锁定读取（SELECT FOR WITH UPDATE或LOCK IN SHARE MODE），UPDATE和DELETE语句，锁定取决于语句是使用具有唯一搜索条件的唯一索引还是范围类型搜索条件。

  - 对于具有唯一搜索条件的唯一索引，InnoDB只锁定找到的索引记录，而不是锁定之前的间隔。
  - 对于其他搜索条件，InnoDB锁定扫描的索引范围，使用间隙锁或下一个键锁来阻止其他会话插入到范围所覆盖的间隙中。 有关间隙锁和下一个键锁的信息，请参见第14.5.1节“InnoDB锁定”。

- READ COMMITTED

  即使在同一事务中，每次一致的读取都会设置并读取自己的新快照。 有关一致读取的信息，请参见第14.5.2.3节“一致性非锁定读取”。

  对于锁定读取（SELECT FOR WITH UPDATE或LOCK IN SHARE MODE），UPDATE语句和DELETE语句，InnoDB只锁定索引记录，而不锁定之前的间隔，因此允许在锁定记录旁边自由插入新记录。 间隙锁定仅用于外键约束检查和重复键检查。

  因为禁用了间隙锁定，所以可能会出现幻影问题，因为其他会话可以将新行插入到间隙中。 有关幻影的信息，请参见第14.5.4节“幻影行”。

  如果使用READ COMMITTED，则必须使用基于行的二进制日志记录。

  使用READ COMMITTED有附加效果：

  - 对于UPDATE或DELETE语句，InnoDB仅为更新或删除的行保存锁。 在MySQL评估WHERE条件之后，释放对不匹配行的记录锁定。 这大大降低了死锁的可能性，但它们仍然可以发生。
  - 对于UPDATE语句，如果一行已被锁定，InnoDB执行“半连续”读取，将最新的提交版本返回给MySQL，以便MySQL可以确定该行是否与UPDATE的WHERE条件匹配。 如果行匹配（必须更新），则MySQL再次读取该行，这次InnoDB要么锁定它，要么等待锁定。

  考虑下面的例子，从这个表开始：

  ```mysql
  CREATE TABLE t (a INT NOT NULL, b INT) ENGINE = InnoDB;
  INSERT INTO t VALUES (1,2),(2,3),(3,2),(4,3),(5,2);
  COMMIT;
  ```

  在这种情况下，表没有索引，因此搜索和索引扫描使用隐藏的聚簇索引进行记录锁定（请参见第14.8.2.1节“聚簇索引”和“二级索引”）。

  假设一个客户端使用这些语句执行UPDATE：

  ```mysql
  SET autocommit = 0;
  UPDATE t SET b = 5 WHERE b = 3;
  ```

  假设第二个客户端通过执行这些第一个客户端的语句来执行UPDATE：

  ```mysql
  SET autocommit = 0;
  UPDATE t SET b = 4 WHERE b = 2;
  ```

  当InnoDB执行每个UPDATE时，它首先获取每行的排它锁，然后决定是否修改它。 如果InnoDB不修改该行，则释放该锁。 否则，InnoDB会保留该锁直到事务结束。 这会影响事务处理，如下所示。

  当使用默认的REPEATABLE READ隔离级别时，第一个UPDATE获取x锁并且不释放它们中的任何一个：

  ```mysql
  x-lock(1,2); retain x-lock
  x-lock(2,3); update(2,3) to (2,5); retain x-lock
  x-lock(3,2); retain x-lock
  x-lock(4,3); update(4,3) to (4,5); retain x-lock
  x-lock(5,2); retain x-lock
  ```

  第二个UPDATE尝试获取任何锁（因为第一个更新保留了所有行上的锁）就会阻塞，直到第一个UPDATE提交或回滚才会继续：

  ```mysql
  x-lock(1,2); block and wait for first UPDATE to commit or roll back
  ```

  如果使用READ COMMITTED，则第一个UPDATE获取x锁并释放那些不修改的行：

  ```mysql
  x-lock(1,2); unlock(1,2)
  x-lock(2,3); update(2,3) to (2,5); retain x-lock
  x-lock(3,2); unlock(3,2)
  x-lock(4,3); update(4,3) to (4,5); retain x-lock
  x-lock(5,2); unlock(5,2)
  ```

  对于第二次更新，InnoDB执行“半连续”读取，将每行的最新提交版本返回给MySQL，以便MySQL可以确定该行是否匹配UPDATE的WHERE条件：

  ```mysql
  x-lock(1,2); update(1,2) to (1,4); retain x-lock
  x-lock(2,3); unlock(2,3)
  x-lock(3,2); update(3,2) to (3,4); retain x-lock
  x-lock(4,3); unlock(4,3)
  x-lock(5,2); update(5,2) to (5,4); retain x-lock
  ```

  使用READ COMMITTED隔离级别的效果与启用不建议使用的innodb_locks_unsafe_for_binlog配置选项相同，但有以下例外：

  - 启用innodb_locks_unsafe_for_binlog是一个全局设置，会影响所有会话，而隔离级别可以为所有会话全局设置，也可以单独为每个会话设置。
  - innodb_locks_unsafe_for_binlog只能在服务器启动时设置，而隔离级别可以在启动时设置或在运行时更改。

  因此READ COMMITTED比innodb_locks_unsafe_for_binlog提供更好，更灵活的控制。

- READ UNCOMMITTED

  SELECT语句以非锁定方式执行，但可能会使用一个可能的早期版本的行。 因此，使用这个隔离级别，这样的读取是不一致的。 这也被称为脏读。 否则，这个隔离级别就像READ COMMITTED一样。

- SERIALIZABLE

  这个级别就像REPEATABLE READ，但是InnoDB隐式地将所有普通的SELECT语句转换成SELECT ... LOCK IN SHARE MODE，如果autocommit被禁用的话。 如果启用自动提交，则SELECT是它自己的事务。 因此，它被认为是只读的，并且如果作为一致的（非锁定的）读取来执行，则可以被序列化，并且不需要阻塞其他事务。 （如果其他事务已经修改了所选的行，强制一个普通的SELECT阻塞，请禁用自动提交。

Locking Reads(锁定读取)

如果查询数据，然后在同一事务中插入或更新相关数据，则常规SELECT语句不会提供足够的保护。 其他事务可以更新或删除刚才查询的相同行。 InnoDB支持两种类型的锁定读取，提供额外的安全性：

- SELECT ... LOCK IN SHARE_MODE

  在读取的任何行上设置共享模式锁定。 其他会话可以读取行，但在您的事务提交之前无法修改它们。 如果这些行中的任何一行已被另一个尚未提交的事务更改，则查询将等待，直到该事务结束，然后使用最新的值。

- SELECT ... FOR UPDATE

  对于搜索遇到的索引记录，锁定行和任何关联的索引条目，就像为这些行发布UPDATE语句一样。 其他事务被阻止更新这些行，从执行SELECT ... LOCK IN SHARE MODE，或者通过读取特定事务隔离级别的数据。 一致的读操作将忽略在读取视图中存在的记录上设置的任何锁。 （记录的旧版本不能被锁定;它们通过在记录的内存副本上应用撤消日志来重新构建）。

在处理树形结构或图形结构化数据时，这些子句主要是有用的，无论是在单个表中还是在多个表中拆分。 你从一个地方到另一个地方遍历边缘或树枝，同时保留返回的权利，并改变任何这些“指针”值。

当事务被提交或回滚时，由LOCK IN SHARE MODE和FOR UPDATE查询设置的所有锁都被释放。

> **注意**
>
> 使用SELECT FOR UPDATE锁定行进行更新仅适用于禁用自动提交时（通过以START TRANSACTION启动事务或将autocommit设置为0）。如果启用自动提交，则不会锁定与规范匹配的行。

##### Locking Read Examples

假设你想插入一个新的行到一个表子中，并且确保这个子行在父表中有一个父行。 您的应用程序代码可以确保贯穿这一系列操作的参照完整性。

首先，使用一致的读取来查询表PARENT并验证父行是否存在。 你可以安全地将子行插入到表CHILD中吗？ 不，因为其他会话可能会在您的SELECT和INSERT之间的那一刻删除父行，而您没有意识到这一点。

为了避免这个潜在的问题，执行 SELECT 时使用 LOCK IN SHARE MODE:

```mysql
SELECT * FROM parent WHERE NAME = 'Jones' LOCK IN SHARE MODE;
```

LOCK IN SHARE MODE查询返回父'Jones'后，可以安全地将子记录添加到CHILD表并提交事务。 任何尝试获取PARENT表中适用行的独占锁的事务都会一直等到完成，也就是说，直到所有表中的数据处于一致状态。

再举一个例子，考虑表CHILD_CODES中的整数计数器字段，用于为添加到表CHILD的每个孩子分配唯一的标识符。 不要使用一致性读取或共享模式读取来读取计数器的当前值，因为数据库的两个用户可以看到相同的计数器值，并且如果两个事务试图添加行 CHILD表的标识符相同。

在这里，LOCK IN SHARE MODE并不是一个好的解决方案，因为如果两个用户同时读取计数器，至少有一个用户在尝试更新计数器时最终死锁。

要实现读取和递增计数器，首先使用FOR UPDATE执行计数器的锁定读取，然后递增计数器。 例如：

```mysql
SELECT counter_field FROM child_codes FOR UPDATE;
UPDATE child_codes SET counter_field = counter_field + 1;
```

SELECT ... FOR UPDATE读取最新的可用数据，在其读取的每一行上设置独占锁。 因此，它设置一个搜索的SQL UPDATE在行上设置的相同的锁。

前面的描述仅仅是SELECT ... FOR UPDATE如何工作的一个例子。 在MySQL中，生成唯一标识符的具体任务实际上可以使用对表的单一访问来完成：

```mysql
UPDATE child_codes SET counter_field = LAST_INSERT_ID(counter_field + 1);
SELECT LAST_INSERT_ID();
```

SELECT语句仅检索标识符信息（特定于当前连接）。 它不访问任何表。