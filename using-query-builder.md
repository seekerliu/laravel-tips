# Laravel Query Builder 原理及用法

从 `CURD` 到 `排序` 和 `过滤`，`Query Builder` 提供了方便的操作符来处理数据库中的数据。这些操作符大多数可以组合在一起，以充分利用单个查询。

`Laravel` 一般使用 `DB` facade 来进行数据库查询。当我们执行 `DB` 的「命令」时，`Query Builder` 会构建一个 SQL 查询，该查询将根据 `table()` 方法中指定的表执行查询。

![Executing database operations using Query Builder](./figures/using-query-builder-1.png)

该查询将使用 `app/config/database.php` 文件中指定的数据库连接执行。 查询执行的结果将返回：检索到的记录、布尔值或一个空结果集。