# Laravel Container (容器) 深入理解 (下)

>本文大部分翻译自 DAVE JAMES MILLER 的 [《Laravel’s Dependency Injection Container in Depth》](https://davejamesmiller.com/2017/06/15/laravel-illuminate-container-in-depth) 。

上文介绍了 `Dependency Injection Containers (容器)` 的基本概念，现在接着深入讲解 `Laravel` 的 `Container`。

`Laravel` 中实现的 `Inversion of Control (IoC) / Dependency Injection (DI) Container` 非常强悍，但文档中很低调的没有细讲它。

>本文中示例基于 `Laravel 5.5` ，其它版本差不多。

# 准备工作
## 1.`Dependency Injection`

关于 `DI` 请看这篇 [《Laravel Dependency Injection (依赖注入) 概念详解》](https://github.com/seekerliu/laravel-tips/blob/master/do-you-need-a-dependency-injection-container.md)，这里不再赘述。

## 2. 初识 `Container`
`Laravel` 中有一大堆访问 `Container` 实例的姿势，比如最简单的：
```php
$container = app();
```
但我们还是先关注下 `Container` 类本身。

>`Laravel` 官方文档中一般使用 `$this->app` 代替 `$container`。它是 `Application` 类的实例，而 `Application` 类继承自 `Container` 类。

## 3. 在 `Laravel` 之外使用 `Illuminate\Container`
如果在 `Laravel` 之外
```bash
mkdir container && cd container
composer require illuminate/container
```

```php
// 新建一个 container.php，文件名随便取
<?php
include './vendor/autoload.php';

use Illuminate\Container\Container;
$container = Container::getInstance();
```

# `Container` 的技能们
## 技能Q. 基本用法，用`type hint (类型提示)` `注入` 依赖：
只需要在自己类的构造函数中使用 `type hint` 就实现 `DI`：
```php
class MyClass
{
    private $dependency;

    public function __construct(AnotherClass $dependency)
    {
        $this->dependency = $dependency;
    }
}
```

接下来用 `Container` 的 `make` 方法来代替 `new MyClass`:
```php
$instance = $container->make(MyClass::class);
```

`Container` 会自动实例化依赖的对象，所以它等同于：
```php
$instance = new MyClass(new AnotherClass());
```

如果 `AnotherClass` 也有 `依赖`，那么 `Container` 会递归注入它所需的依赖。

>`Container` 使用 `[Reflection (反射)](http://php.net/manual/zh/book.reflection.php)` 来找到并实例化构造函数参数中的那些类，实现起来并不复杂，以后的文章里再介绍。

### 实战
下面是 `[PHP-DI 文档](http://php-di.org/doc/getting-started.html)` 中的一个例子，它分离了「用户注册」和「发邮件」的过程：
```php
class Mailer
{
    public function mail($recipient, $content)
    {
        // Send an email to the recipient
        // ...
    }
}
```

```php
class UserManager
{
    private $mailer;

    public function __construct(Mailer $mailer)
    {
        $this->mailer = $mailer;
    }

    public function register($email, $password)
    {
        // 创建用户账户
        // ...

        // 给用户的邮箱发个 “hello" 邮件
        $this->mailer->mail($email, 'Hello and welcome!');
    }
}
```

```php
use Illuminate\Container\Container;

$container = Container::getInstance();

$userManager = $container->make(UserManager::class);
$userManager->register('dave@davejamesmiller.com', 'MySuperSecurePassword!');
```

## 技能W. Binding Interfaces to Implementations (绑定接口到实现)
用`Container` 可以轻松地写一个接口，然后在运行时实例化一个具体的实例。 首先定义接口：

```php
interface MyInterface { /* ... */ }
interface AnotherInterface { /* ... */ }
```

然后声明实现这些接口的具体类。下面这个类不但实现了一个接口，还依赖了实现另一个接口的类实例：
```php
class MyClass implements MyInterface
{
    private $dependency;

    // 依赖了一个实现 AnotherInterface 接口的类的实例
    public function __construct(AnotherInterface $dependency)
    {
        $this->dependency = $dependency;
    }
}
```

现在用 `Container` 的 `bind()` 方法来让每个 `接口` 和实现它的类一一对应起来：
```php
$container->bind(MyInterface::class, MyClass::class);
$container->bind(AnotherInterface::class, AnotherClass::class);
```

最后，用 `接口名` 而不是 `类名` 来传给 `make()`:
```php
$instance = $container->make(MyInterface::class);
```

>注意：如果你忘记绑定它们，会导致一个 `Fatal Error`:"Uncaught ReflectionException: Class MyInterface does not exist"。

### 实战
下面是可封装的 `Cache` 层：
```php
interface Cache
{
    public function get($key);
    public function put($key, $value);
}
```

```php
class Worker
{
    private $cache;

    public function __construct(Cache $cache)
    {
        $this->cache = $cache;
    }

    public function result()
    {
        // 去缓存里查询
        $result = $this->cache->get('worker');

        if ($result === null) {
             // 如果缓存里没有，就去别的地方查询，然后再放进缓存中
            $result = do_something_slow();

            $this->cache->put('worker', $result);
        }

        return $result;
    }
}
```

```php
use Illuminate\Container\Container;

$container = Container::getInstance();
$container->bind(Cache::class, RedisCache::class); 

$result = $container->make(Worker::class)->result();
```
这里用 `Redis` 做缓存，如果改用其他缓存，只要把 `RedisCache` 换成别的就行了，easy!

## 技能E：Binding Abstract & Concret Classes （绑定抽象类和具体类）：

绑定还可以用在抽象类：
```php
$container->bind(MyAbstract::class, MyConcreteClass::class);
```

或者继承的类中：
```php
$container->bind(MySQLDatabase::class, CustomMySQLDatabase::class);
```

## 技能R：自定义绑定
如果类需要一些附加的配置项，可以把 `bind()` 方法中的第二个参数换成 `Closure (闭包函数)`：
```php
$container->bind(Database::class, function (Container $container) {
    return new MySQLDatabase(MYSQL_HOST, MYSQL_PORT, MYSQL_USER, MYSQL_PASS);
});
```

闭包也可用于定制 `具体类` 的实例化方式：
```php
$container->bind(GitHub\Client::class, function (Container $container) {
    $client = new GitHub\Client;
    $client->setEnterpriseUrl(GITHUB_HOST);
    return $client;
});
```

## 技能T：Resolving Callbacks (回调)
可用 `resolveing()` 方法来注册一个 `callback (回调函数)`，而不是直接覆盖掉之前的 `绑定`。 这个函数会在绑定的类解析完成之后调用。
```php
$container->resolving(GitHub\Client::class, function ($client, Container $container) {
    $client->setEnterpriseUrl(GITHUB_HOST);
});
```

如果有一大堆 `callbacks`，他们全部都会被调用。对于 `接口` 和 `抽象类` 也可以这么用：
```php
$container->resolving(Logger::class, function (Logger $logger) {
    $logger->setLevel('debug');
});

$container->resolving(FileLogger::class, function (FileLogger $logger) {
    $logger->setFilename('logs/debug.log');
});

$container->bind(Logger::class, FileLogger::class);

$logger = $container->make(Logger::class);
```

更 `diao` 的是，还可以注册成「什么类解析完之后都调用」：
```php
$container->resolving(function ($object, Container $container) {
    // ...
});
```

但这个估计只有 `logging` 和 `debugging` 才会用到。

## 技能Y：Extending a Class (扩展一个类)
使用 `extend()` 方法，可以封装一个类然后返回一个不同的对象 (装饰模式)：
```php
$container->extend(APIClient::class, function ($client, Container $container) {
    return new APIClientDecorator($client);
});
```

注意：这两个类要实现相同的 `接口`，不然用类型提示的时候会出错：
```php
interface Getable
{
    public function get();
}
```

```php
class APIClient implements Getable
{
    public function get()
    {
        return 'yes!';
    }
}
```

```php
class APIClientDecorator implements Getable
{
    private $client;

    public function __construct(APIClient $client)
    {
        $this->client = $client;
    }

    public function get()
    {
        return 'no!';
    }
}
```

```php
class User
{
    private $client;

    public function __construct(Getable $client)
    {
        $this->client = $client;
    }
}
```

```php
$container->extend(APIClient::class, function ($client, Container $container) {
    return new APIClientDecorator($client);
});
//
$container->bind(Getable::class, APIClient::class);

// 此时 $instance 的 $client 属性已经是 APIClentDecorator 类型了
$instance = $container->make(User::class);
```

## 技能U：单例
使用 `bind()` 方法绑定后，每次解析时都会新实例化一个对象(或重新调用闭包)，如果想获取 `单例` ，则用 `signleton()` 方法代替 `bind()`：
```php
$container->singleton(Cache::class, RedisCache::class);
```
绑定单例 `闭包`：
```php
container->singleton(Database::class, function (Container $container) {
    return new MySQLDatabase('localhost', 'testdb', 'user', 'pass');
});
```
绑定 `具体类` 的时候，不需要第二个参数：
```php
$container->singleton(MySQLDatabase::class);
```

在每种情况下，`单例` 对象将在第一次需要时创建，然后在后续重复使用。
如果你已经有一个 `实例` 并且想重复使用，可以用 `instance()` 方法。 `Laravel` 就是用这种方法来确保每次获取到的都是同一个 `Container` 实例：
```php
$container->instance(Container::class, $container);
```

## 技能I：Arbitrary Binding Names (任意绑定名称)
`Container` 还可以绑定任意字符串而不是类/接口名称。但这种情况下不能使用类型提示，并且只能用 `make()` 来获取实例。
```php
$container->bind('database', MySQLDatabase::class);
$db = $container->make('database');
```
为了同时支持类/接口名称和短名称，可以使用 `alias()`：
```php
$container->singleton(Cache::class, RedisCache::class);
$container->alias(Cache::class, 'cache');

$cache1 = $container->make(Cache::class);
$cache2 = $container->make('cache');

assert($cache1 === $cache2);
```

## 技能O：保存任何值
`Container` 还可以用来保存任何值，例如 `configuration` 数据：
```php
$container->instance('database.name', 'testdb');
$db_name = $container->make('database.name');
```
它支持数组访问语法，这样用起来更自然：
```php
$container['database.name'] = 'testdb';
$db_name = $container['database.name'];
```
>这是因为 `Container` 实现了 PHP 的 `[ArrayAccess](http://php.net/manual/zh/class.arrayaccess.php)` 接口。

当处理 `Closure` 绑定的时候，你会发现这个方式非常好用：
```php
$container->singleton('database', function (Container $container) {
    return new MySQLDatabase(
        $container['database.host'],
        $container['database.name'],
        $container['database.user'],
        $container['database.pass']
    );
});
```
>`Laravel` 自己没有用这种方式来处理配置项，它使用了一个单独的 `Config` 类本身。 `PHP-DI` 用了。

数组访问语法还可以代替 `make()` 来实例化对象：
```php
$db = $container['database'];
```

## 技能P：Dependency Injection for Functions & Methods (给函数或方法注入依赖)
除了给构造函数注入依赖，`Laravel` 还可以往任意函数中注入：
```php
function do_something(Cache $cache) { /* ... */ }
$result = $container->call('do_something');
```
函数的附加参数可以作为索引或关联数组传递：
```php
function show_product(Cache $cache, $id, $tab = 'details') { /* ... */ }

// show_product($cache, 1)
$container->call('show_product', [1]);
$container->call('show_product', ['id' => 1]);

// show_product($cache, 1, 'spec')
$container->call('show_product', [1, 'spec']);
$container->call('show_product', ['id' => 1, 'tab' => 'spec']);
```
除此之外，`闭包`：
```php
$closure = function (Cache $cache) { /* ... */ };
$container->call($closure);
```
`静态方法`：
```php
class SomeClass
{
    public static function staticMethod(Cache $cache) { /* ... */ }
}
```

```php
$container->call(['SomeClass', 'staticMethod']);
// or:
$container->call('SomeClass::staticMethod');
```
`实例的方法`：
```php
class PostController
{
    public function index(Cache $cache) { /* ... */ }
    public function show(Cache $cache, $id) { /* ... */ }
}
```

```php
$controller = $container->make(PostController::class);

$container->call([$controller, 'index']);
$container->call([$controller, 'show'], ['id' => 1]);
```

都可以注入。

## 技能A: 调用实例方法的快捷方式
使用 `ClassName@methodName` 语法可以快捷调用实例中的方法：
```php
$container->call('PostController@index');
$container->call('PostController@show', ['id' => 4]);
```

因为`Container` 被用来实例化类。意味着：
1. `依赖` 被注入进构造函数（或者方法）；
2. 如果需要复用实例，可以定义为单例；
3. 可以用接口或任何名称来代替具体类。

所以这样调用也可以生效：
```php
class PostController
{
    public function __construct(Request $request) { /* ... */ }
    public function index(Cache $cache) { /* ... */ }
}
```

```php
$container->singleton('post', PostController::class);
$container->call('post@index');
```
最后，还可以传一个「默认方法」作为第三个参数。如果第一个参数是没有指定方法的类名称，则将调用默认方法。 `Laravel` 用这种方式来处理 `[event handlers](https://laravel.com/docs/5.5/events#registering-events-and-listeners)` :
```php
$container->call(MyEventHandler::class, $parameters, 'handle');

// 相当于:
$container->call('MyEventHandler@handle', $parameters);
```

## 技能S：Method Call Bindings (方法调用绑定)
`bindMethod()` 方法可用来覆盖方法，例如用来传递其他参数：
```php
$container->bindMethod('PostController@index', function ($controller, $container) {
    $posts = get_posts(...);

    return $controller->index($posts);
});
```
下面的方式都有效，调用闭包来代替调用原始的方法：
```php
$container->call('PostController@index');
$container->call('PostController', [], 'index');
$container->call([new PostController, 'index']);
```

但是，`call()` 的任何其他参数都不会传递到闭包中，因此不能使用它们。
```php
$container->call('PostController@index', ['Not used :-(']);
```
>注意：这种方式不是 `[Container 接口](https://github.com/laravel/framework/blob/5.5/src/Illuminate/Contracts/Container/Container.php)` 的一部分，只有在它的实现类 `[Container](https://github.com/laravel/framework/blob/5.4/src/Illuminate/Container/Container.php)` 才有。在这个 `[PR](https://github.com/laravel/framework/pull/16800)` 里可以看到它加了什么以及为什么参数被忽略。

## 技能D：Contextual Bindings (上下文绑定)
有时候你想在不同的地方给接口不同的实现。这里有 `[Laravel 文档](https://laravel.com/docs/5.5/container#contextual-binding)` 里的一个例子：
```php
$container
    ->when(PhotoController::class)
    ->needs(Filesystem::class)
    ->give(LocalFilesystem::class);

$container
    ->when(VideoController::class)
    ->needs(Filesystem::class)
    ->give(S3Filesystem::class);
```
现在 `PhotoController` 和 `VideoController` 都依赖了 `Filesystem` 接口，但是收到了不同的实例。

可以像 `bing()` 那样，给 `give()` 传闭包：
```php
    ->when(VideoController::class)
    ->needs(Filesystem::class)
    ->give(function () {
        return Storage::disk('s3');
    });
```

或者短名称：
```php
$container->instance('s3', $s3Filesystem);

$container
    ->when(VideoController::class)
    ->needs(Filesystem::class)
    ->give('s3');
```

## 技能F：Binding Parameters to Primitives (绑定初始数据)
当有一个类不仅需要接受一个注入类，还需要注入一个基本值（比如整数）。
还可以通过将变量名称 (而不是接口) 传递给 `needs()` 并将值传递给 `give()` 来注入需要的任何值 (字符串、整数等) ：

```php
$container
    ->when(MySQLDatabase::class)
    ->needs('$username')
    ->give(DB_USER);
```

还可以使用闭包实现延时加载，只在需要的时候取回这个 `值` 。
```php
$container
    ->when(MySQLDatabase::class)
    ->needs('$username')
    ->give(function () {
        return config('database.user');
    });
```

这种情况下，不能传递类或命名的依赖关系（例如，give('database.user')），因为它将作为字面值返回。所以需要使用闭包：
```php
$container
    ->when(MySQLDatabase::class)
    ->needs('$username')
    ->give(function (Container $container) {
        return $container['database.user'];
    });
```

## 技能G: Tagging (标记)
`Container` 可以用来「标记」有关系的绑定：
```php
$container->tag(MyPlugin::class, 'plugin');
$container->tag(AnotherPlugin::class, 'plugin');
```

这样会以数组的形式取回所有「标记」的实例：
```php
foreach ($container->tagged('plugin') as $plugin) {
    $plugin->init();
}
```

`tag()` 方法的两个参数都可以接受数组：
```php
$container->tag([MyPlugin::class, AnotherPlugin::class], 'plugin');
$container->tag(MyPlugin::class, ['plugin', 'plugin.admin']);
```

## 技能H：Rebinding (重新绑定)
>这个功能很少用到，可以跳过，仅供参考。

在绑定或实例被使用之后又发生了变化，将调用一个 `rebinding` 方法。 下例中， `Auth` 使用 `Session` 类后，`Session` 类将被替换，此时需要通知 `Auth` 类这个变动：
```php
$container->singleton(Auth::class, function (Container $container) {
    $auth = new Auth;
    $auth->setSession($container->make(Session::class));

    $container->rebinding(Session::class, function ($container, $session) use ($auth) {
        $auth->setSession($session);
    });

    return $auth;
});

$container->instance(Session::class, new Session(['username' => 'dave']));
$auth = $container->make(Auth::class);
echo $auth->username(); // dave
$container->instance(Session::class, new Session(['username' => 'danny']));

echo $auth->username(); // danny
```

`Rebinding` 的更多信息可以看这两个链接：  
[https://stackoverflow.com/questions/38974593/laravels-ioc-container-rebinding-abstract-types](https://stackoverflow.com/questions/38974593/laravels-ioc-container-rebinding-abstract-types)  
[https://code.tutsplus.com/tutorials/digging-in-to-laravels-ioc-container--cms-22167](https://code.tutsplus.com/tutorials/digging-in-to-laravels-ioc-container--cms-22167)

还有一个 `refresh()` 方法来处理这种模式：
```php
$container->singleton(Auth::class, function (Container $container) {
    $auth = new Auth;
    $auth->setSession($container->make(Session::class));

    $container->refresh(Session::class, $auth, 'setSession');

    return $auth;
});
```
它还返回现有的实例或绑定（如果有的话），所以可以这样做：
```php
// This only works if you call singleton() or bind() on the class
$container->singleton(Session::class);

$container->singleton(Auth::class, function (Container $container) {
    $auth = new Auth;
    $auth->setSession($container->refresh(Session::class, $auth, 'setSession'));
    return $auth;
});
```

>注意：这种方式不是 `[Container 接口](https://github.com/laravel/framework/blob/5.5/src/Illuminate/Contracts/Container/Container.php)` 的一部分，只有在它的实现类 `[Container](https://github.com/laravel/framework/blob/5.4/src/Illuminate/Container/Container.php)` 才有。

## 技能J：Overriding Constructor Parameters (重写构造函数参数)
`makeWith` 方法允许将附加参数传递给构造函数。它忽略任何现有的实例或单例，可以用于创建具有不同参数的类的多个实例，同时仍然注入依赖关系：
```php
class Post
{
    public function __construct(Database $db, int $id) { /* ... */ }
}
```

```php
$post1 = $container->makeWith(Post::class, ['id' => 1]);
$post2 = $container->makeWith(Post::class, ['id' => 2]);
```

>注意：`Laravel 5.3` 及以下使用 `make($class, $parameters)`。`Laravel 5.4` 中移除了此方法，但是在 `5.4.16` 以后又重新加回来了 `makeWith()` 。

## 技能K：其它
这涵盖了我认为有用的所有方法，但仅仅是简介，不然这篇文章就写不完了。。。

### bound()
如果一个类/名称已经被 `bind()` , `signleton()` ，`instance()` 或 `alias()` 绑定，那么 `bound()` 方法返回 `true`。
```php
if (! $container->bound('database.user')) {
    // ...
}
```
还可以使用数组访问语法和 `isset()`：
```php
if (! isset($container['database.user'])) {
    // ...
}
```

可以使用 `unset()` 来重置它，这会删除指定的绑定/实例/别名。
```php
unset($container['database.user']);
var_dump($container->bound('database.user')); // false
```

### bindIf()

`bindIf()` 和 `bind()` 功能类似，差别在于只有在现有绑定不存在的情况下才注册绑定。 它一般被用在 `package` 中注册一个可被用户重写的默认绑定。
```php
$container->bindIf(Loader::class, FallbackLoader::class);
```
不过并没有 `singletonIf()` 方法，只能用 `bindIf($abstract, $concrete, true)` 来实现。
```php
$container->bindIf(Loader::class, FallbackLoader::class, true);
```
它等同于:
```php
if (! $container->bound(Loader::class)) {
    $container->singleton(Loader::class, FallbackLoader::class);
}
```

### resolved()
如果一个类已经被解析，`resolved()` 方法会返回 `true`。
```php
var_dump($container->resolved(Database::class)); // false
$container->make(Database::class);
var_dump($container->resolved(Database::class)); // true
```

我也不太确定他用在什么地方。。。
如果用过 `unset()` 之后它会被重置：
```php
unset($container[Database::class]);
var_dump($container->resolved(Database::class)); // false
```

### factory()
`factory()` 方法返回一个不需要参数并调用 `make()` 的闭包。
```php
$dbFactory = $container->factory(Database::class);
$db = $dbFactory();
```

这个东西我也不知道有什么用。。。

### wrap()
`wrap` 方法包装一个闭包，以便在执行时依赖关系被注入。 它接受一个数组参数; 返回的闭包不带参数：
The wrap() method wraps a closure so that its dependencies will be injected when it is executed. The wrap method accepts an array of parameters; the returned closure takes no parameters:
```php
$cacheGetter = function (Cache $cache, $key) {
    return $cache->get($key);
};

$usernameGetter = $container->wrap($cacheGetter, ['username']);

$username = $usernameGetter();
```

我也不知道它有啥用，因为返回的闭包没带回参数。。。

>注意：这个方法不是 `[Container 接口](https://github.com/laravel/framework/blob/5.5/src/Illuminate/Contracts/Container/Container.php)` 的一部分，只有在它的实现类 `[Container](https://github.com/laravel/framework/blob/5.4/src/Illuminate/Container/Container.php)` 才有。

### afterResolving()
`afterResolving()` 方法作用与 `resolving()` 完全相同，不同之处是 调用 「resolving」回调之后再调用 「afterResolving」回调。  
不知道什么时候会用到它。。。

# 最后再附几个

`isShared()` – 确定一个给定的类型是一个 signleton/instance
`isAlias()` – 确定给定的字符串是否是已注册的 `别名`
`hasMethodBinding()` - 确定容器是否具有给定的 `method binding`
`getBindings()` - 取回所有已注册绑定的原始数组
`getAlias($abstract)` - 获取基础类/绑定名称的别名
`forgetInstance($abstract)` - 清除单个实例对象
`forgetInstances()` - 清除所有实例对象
`flush()` - 清除所有绑定和实例，有效地重置容器
`setInstance()` - 替换 `getInstance()` 使用的实例 (提示：使用 setInstance(null)来清除它，这样下一次它将生成一个新的实例)

>注意：这些方法不是 `[Container 接口](https://github.com/laravel/framework/blob/5.5/src/Illuminate/Contracts/Container/Container.php)` 的一部分，只有在它的实现类 `[Container](https://github.com/laravel/framework/blob/5.4/src/Illuminate/Container/Container.php)` 才有。