# Laravel Mass-Assignment (批量赋值) 的真正含义

> 初次遇到 `批量赋值` 的时候，很容易理解成 `批量添加多条数据`，实际并非如此。请看下面的例子。

假设用户表 `users` 结构如下，且通过 `is_admin` 字段值为 `1` 或 `0` 来判断用户是否为 `管理员`，其中 `is_admin` 字段默认值为 `0`：
```bash
+----+-----------+------------------+----------+--------------------------------------------------------------+
| id | name      | email            | is_admin | password                                                     |
+----+-----------+------------------+----------+--------------------------------------------------------------+
|  1 | seekerliu | me@seekerliu.com |        1 | $2y$10$RL6r.MwoJd.oOvKRYhUpmeQI6hUpoG/KgGNhA6X5JrRqfVbooCs92 |
+----+-----------+------------------+----------+--------------------------------------------------------------+
```
正常情况下，我们通过这种方式新建一个 `普通` 用户：
```php
public function store (Request $request)
{
    $user = new \App\User;
		
    // 赋值
    $user->name = $request->name;
    $user->email = $request->email;
    $user->password = bcrypt($request->password);
		
    // 新建一个用户
    $user->save();
}
```

为了方便，我们可以使用 `$request->all()` 获取用户提交的所有表单数据：
```php
public function store (Request $request)
{
    $user = new \App\User;
		
    // Mass-Assignment 批量赋值
    $data = $request->all();   
		
    // 新建一个用户
    $user->create($data);
}
```
这种情况下，如果用户提交正确的表单数据，例如： `['name' => 'liu', 'email' => 'liu@seekerliu.com', 'password' => 'test']` ，会新建一个 `普通` 用户。  
但只要用户在表单中伪造一个 `['is_admin' => 1]` 字段，就能新建一个 `管理员` 用户。
这种通过将一大堆数据同时传递给模型的 `create()` 方法来新建一行的方式就是 `Mass-Assignment (批量赋值)` 。

`Laravel` 提供了保护 `Mass-Assignment` 的方法，那就是在模型上定义 `fillable` 或 `guarded` 的属性，例如：
```php
class User extend Model
{
    protected $fillable = ['name', 'email', 'password'];
}
```
或：
```php
class User extend Model
{
    protected $guarded = ['is_admin'];
}
```
这样，在执行 `create()` 方法时，`Eloquent` 模型会先使用 `fill()` 方法对数据进行过滤，去掉 `$fillable` 以外的字段（白名单），或去掉 `$guarded` 中的字段（黑名单），来保证只获取预期的表单字段。

以上就是 `Laravel` 的 `Mass-Assignment` 。
