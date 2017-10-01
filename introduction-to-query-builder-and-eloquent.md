# Laravel Query Builder & Eloquent ORM 介绍

>本文翻译自 [《Laravel - My first framework》](https://leanpub.com/laravel-first-framework/)

`Laravel`拥有两个功能强大的功能来执行数据库操作：
* Query Builder - 查询构造器
* Eloquent ORM

![Database operations in Laravel](./introduction-to-query-builder-and-eloquent-1.png)

我们可以单独使用这两个功能，或两者结合起来，以更灵活的进行数据库操作。下面来看看它们的区别。

## Query Builder

`Laravel` 的 `Query Builder` 为执行数据库查询提供了一个干净简单的接口。它可以用来进行各种数据库操作，例如：   

*   Retrieving records - 检索记录
*   Inserting new records - 插入记录
*   Deleting records - 删除记录
*   Updating records - 更新记录
*   Performing "Join" queries - 执行 JOIN
*   Executing raw SQL queries - 执行「原生」 SQL 语句
*   Filtering, grouping and sorting of records - 过滤、分组和排序记录

用 `DB` Facade 来使用 `Query Builder`

```php
// 给 orders 表创建几条新记录
DB::table('orders')->insert([                      
  ['price' => 400, 'product' => 'Laptop'],
  ['price' => 200, 'product' => 'Smartphone'],
  ['price' => 50,  'product' => 'Accessory'],
));                                     

// 检索 orders 表中 所有 price 大于 100 的记录
$orders = DB::table('orders')
  ->where('price', '>', 100)
  ->get();

// 获取 orders 表中 price 列的平均值
$averagePrice = DB::table('orders')->avg('price');          

// 查找 orders 表中所有 price 等于 50 的记录
// 把他们 product 字段改为 Laptop
// proce 字段改为 400
DB::table('orders')
  ->where('price', 50)
  ->update(['product' => 'Laptop', 'price' => 400]);
```

`Query Builder` 是一个非常易于使用但很强大的与数据库进行交互的方式。 
接下来看看 `Eloquent ORM`。

## Eloquent ORM
`Eloquent` 是 `Laravel` 中对 `Active Recode pattern (领域模式)` 的实现。它通过 「模型」的概念来根数据表进行交互。

> ORM, Active Record and Eloquent   
`ORM - Object/Relational Mapping (对象/关系映射)`，是随着面向对象的软件开发方法发展而产生的一种技术，用来把对象模型表示的对象映射到基于 SQL  的关系模型数据库结构中去。这样，我们在具体的操作实体对象的时候，就不需要再去和复杂的 SQL 语句打交道，只需简单的操作实体对象的属性和方法。ORM 技术是在对象和关系之间提供了一条桥梁，前台的对象型数据和数据库中的关系型的数据通过这个桥梁来相互转化。  
`Active Record` ，是一种领域模型模式，特点是一个「模型」类对应关系型数据库中的一个表，而「模型」类的一个实例对应表中的一行记录。  
流行的Web开发框架，如 `CodeIgniter` ，`Ruby on Rails` 和 `Symfony` 都使用Active Record 模式来简化数据库操作。 而 `Eloquent` 就是 `Laravel` 自己的 `Active Record` 模式的实现。

要开始使用 `Eloquent`，只需要在 `app/config/database.php` 文件中配置数据库连接设置，并创建与数据库表相对应的「模型」。 `Eloquent` 的语法与普通的 PHP 代码没有太大的不同，所以很容易阅读。  
`Eloquent`提供许多先进的功能来管理和操作数据，其中最突出的是：

*   Working with data by the use of "models" - 通过「模型」来处理数据
*   Creating relationships between data - 创建表之间的关联关系
*   Querying related data - 查询数据
*   Conversion of data to JSON and arrays - 将数据转换为 `JSON` 或数组
*   Query optimization - 优化查询
*   Automatic timestamps - 自动时间戳

`Eloquent` 涵盖了简单的数据插入，表格之间的数据关联，以及相关数据中的复杂过滤等多个方面。  
例如：

```php
// app/User.php
class User extends Eloquent {
  public function orders()
  {
    return $this->hasMany('Order');
  }
}

// app/models/Order.php
class Order extends Eloquent {}                     

// 在 users 表中创建一行
$user = new User;                               
$user->name = "John Doe";                           
$user->save();                              

// 在 orders 表中创建一行，并与 user 关联
$order = new Order;                             
$order->price   = 100;                          
$order->product = "TV";                         

$user->orders()->save($order);
```

以上示例只是 `Query Builder` 和 `Eloquent ORM` 概念的简单介绍，下一篇中我将详细讲解 `Query Builder` 的用法。