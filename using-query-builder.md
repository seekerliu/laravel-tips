# Laravel Query Builder 原理及用法

>本文翻译自 [《Laravel - My first framework》](https://leanpub.com/laravel-first-framework/)

从 `CURD` 到 `排序` 和 `过滤`，`Query Builder` 提供了方便的操作符来处理数据库中的数据。这些操作符大多数可以组合在一起，以充分利用单个查询。

`Laravel` 一般使用 `DB` facade 来进行数据库查询。当我们执行 `DB` 的「命令」(、或者说「操作符」)时，`Query Builder` 会构建一个 SQL 查询，该查询将根据 `table()` 方法中指定的表执行查询。

![Executing database operations using Query Builder](./figures/using-query-builder-1.png)

该查询将使用 `app/config/database.php` 文件中指定的数据库连接执行。 查询执行的结果将返回：检索到的记录、布尔值或一个空结果集。

下表中是 `Query Builder` 的常用操作符：

<table>
    <tr>
        <td>操作符</td>
        <td>描述</td>
    </tr>
    <tr>
        <td>insert(array(...))</td>
        <td>接收包含字段名和值的数组，插入数据至数据库</td>
    </tr>
    <tr>
        <td>find($id)</td>
        <td>检索一个主键 id 等于给定参数的记录</td>
    </tr>
    <tr>
        <td>update(array(...))</td>
        <td>接收含有字段名和值的数组，更新已存在的记录</td>
    </tr>
    <tr>
        <td>delete()</td>
        <td>删除一条记录</td>
    </tr>
    <tr>
        <td>get()</td>
        <td>返回一个 Illuminate\Support\Collection 结果，其中每个结果都是一个 PHP StdClass 对象的实例，实例中包含每行记录中的列名及其值</td>
    </tr>
    <tr>
        <td>take($number)</td>
        <td>限制查询结果数量</td>
    </tr>
</table>

接下来，将讲解 `Query Builder` 的各种操作。

# Inserting records - 插入
`insert` 操作符将新行（记录）插入到现有表中。我们可以通过提供数据数组作为 `insert` 运算符的参数来指定要插入到表中的数据。

假设有一个 `orders` 表：
<table>
    <tr>
        <td>Key</td>
        <td>Column</td>
        <td>Type</td>
    </tr>
    <tr>
        <td>primary</td>
        <td>id</td>
        <td>int (11), auto-incrementing</td>
    </tr>
    <tr>
        <td></td>
        <td>price</td>
        <td>int (11)</td>
    </tr>
    <tr>
        <td></td>
        <td>product</td>
        <td>varchar(255)</td>
    </tr>
</table>

## 插入单行数据
把数据数组传给 `insert` 操作符，来告诉 `Query Builder` 插入新行：
```php
DB::table('orders')->insert(
    [
        'price' => 200, // 设置 price 字段值
        'product' => 'Console', // 设置 product 字段值
    ]
);
```

`Query Builder`将 `insert` 命令转换为特定于 `database.php` 配置文件中指定的数据库的 SQL 查询。 作为参数传递给 `insert` 命令的数据将以参数的形式放入 SQL 查询中。 然后，SQL 查询将在指定的表 "orders" 上执行，执行结果将返回给调用者。  
下图说明了整个过程：

![Behind the scenes process of running “insert” operator](./figures/using-query-builder-2.png)

>可以看到，Laravel 使用 PDO 来执行 SQL 语句。通过为数据添加占位符来使用准备好的语句可以增强 SQL 注入的保护性，并增加数据插入和更新的安全性。

## 插入多行数据
`Query Builder` 的 `insert` 操作符同样可用于插入多行数据。 传递一个包含数组的数组可以插入任意数量的行：

```php
DB::table('orders')->insert(
    [
        ['price' => 400, 'product' => 'Laptop'],
        ['price' => 200, 'product' => 'Smartphone'],
        ['price' => 50, 'product' => 'Accessory'],
    ]
);
```

此 `insert` 语句将创建三个新记录，Laravel 构建的 SQL 查询是：
```
insertinto`orders`(`price`,`product`)values(?,?),(?,?),(?,?)
```

>可以看到， Laravel 聪明地在一个查询中插入三行数据，而不是运行三个单独的查询。 

# Retrieving records - 检索

`Query Builder` 提供了多种从数据库获取数据的操作符，以灵活的适应许多不同的情况，例如：

*   检索单个记录
*   检索表中的所有记录
*   仅检索表中所有记录的特定列
*   检索表中有限数量的记录

## 检索单个记录

可以使用 `find` 操作符从表中检索单个记录。只需提供要检索的记录的主键的值作为 `find` 的参数，Laravel 将返回该记录作为对象。如果未找到该记录，则返回 `NULL`。

```php
$order = DB::table('orders')->find(3); 

/*
object(stdClass)#157 (3) {
    ["id"]=>string(1) "3"
    ["price"]=>string(3) "200"
    ["product"]=>string(10) "Smartphone"
 }
 */
```

> 注意： `find` 操作符以 `id` 作为主键进行查询，如想使用别的主键，请使用其它操作符。

Laravel 构建的 SQL 查询是：
```
select * from `orders` where `id` = ? limit 1
```

## 检索表中的所有记录
要从表中检索所有记录，可以使用 `get` 操作符而不用任何参数。在指定的表上运行 `get` (前面没有别的操作符) 将会将该表中的所有记录作为对象数组返回。

```php
$orders = DB::table('orders')->get();

/*
array(4) { [0]=>
    object(stdClass)#157 (3) { 
        ["id"]=>string(1) "1"
        ["price"]=>string(3) "200"
        ["product"]=>string(7) "Console"
    }
    
    ... 3 more rows returned as objects ...
}
*/
```

Laravel 构建的 SQL 查询是：
```
select * from `orders`
```

## 检索仅包含特定列的所有记录
将所需的列名作为参数数组传递给 `get` 运算符，可获得表中所有记录的特定列。

```php
$orders = DB::table('orders')->get(['id','price']);

/*
array(4) { [0]=>
    object(stdClass)#157 (2) {
        ["id"]=>string(1) "1"
        ["price"]=>string(3) "200"
    }
    ... 3 more rows returned as objects ... 
}
*/
```

Laravel 构建的 SQL 查询是：
```
select `id`, `price` from `orders`
```

## 检索表中有限数量的记录
要指定要从表中获取的最大记录数，可以使用 `take` 操作符，并将 `get` 附加到查询中。

```php
$orders = DB::table('orders')->take(50)->get();
```

$orders 数组中最多有 50 条数据。

# Updating records - 更新

使用 `Query Builder` 更新记录与创建新记录非常相似。要更新现有记录或一组记录的数据，可以将操作符 `update` 附加到查询中，并将一个新数据数组作为参数传递给它。同时可以使用查询链定位要更新的特定记录。

## 更新特定记录

使用 `where` 操作符来指定特定记录并更新：
```php
DB::table('orders')
    ->where('price','>','50')
    ->update(['price' => 100]);
```

Laravel 构建的 SQL 查询是：
```
update `orders` set `price` = ? where `price` > ?
```

## 更新所有记录
如果不限定条件直接使用 `update` ，将更新表中所有记录：

```php
DB::table('orders')->update(['product'=>'Headphones']);
```

Laravel 构建的 SQL 查询是：
```
update `orders` set `product` = ?
```

# Deleting records - 删除
使用 `Query Builder` 从表中删除记录遵循与更新记录相同的模式。 可以使用 `delete` 操作符删除与某些条件匹配的特定记录或删除所有记录。

## 删除特定记录
使用 `where` 操作符来指定要删除的特定记录：

```php
DB::table('orders')
    ->where('product','=','Smartphone')
    ->delete();
```

Laravel 构建的 SQL 查询是：
```
delete from `orders` where `product` = ?
```

