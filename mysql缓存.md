- Query Cache SELECT Options 查询缓存选择选项

  > 注意
  >
  > 查询缓存从MySQL 5.7.20开始已被弃用，并在MySQL 8.0中被删除。

  在SELECT语句中可以指定两个与查询缓存相关的选项：

  - SQL_CACHE

    如果查询结果可缓存，并且query_cache_type系统变量的值为ON或DEMAND，则查询结果将被缓存。

  - SQL_NO_CACHE

    服务器不使用查询缓存。 它既不检查查询缓存，查看结果是否已被缓存，也不缓存查询结果。

  例如:

  ```mysql
  SELECT SQL_CACHE id, name FROM customer;
  SELECT SQL_NO_CACHE id, name FROM customer;
  ```

  ​