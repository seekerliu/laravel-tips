# Laravel 依赖注入 (Dependency Injection) 概念详解

>本文翻译自 `Symfony` 作者 Fabien Potencier 的 [《Dependency Injection in general and the implementation of a Dependency Injection Container in PHP》](http://fabien.potencier.org/what-is-dependency-injection.html) 系列文章。

* [Part 1: What is Dependency Injection?](http://fabien.potencier.org/article/11/what-is-dependency-injection)
* [Part 2: Do you need a Dependency Injection Container?](http://fabien.potencier.org/article/12/do-you-need-a-dependency-injection-container)
* [Part 3: Introduction to the Symfony Service Container](http://fabien.potencier.org/article/13/introduction-to-the-symfony-service-container)
* [Part 4: Symfony Service Container: Using a Builder to create Services](http://fabien.potencier.org/article/14/symfony-service-container-using-a-builder-to-create-services)
* [Part 5: Symfony Service Container: Using XML or YAML to describe Services](http://fabien.potencier.org/article/15/symfony-service-container-using-xml-or-yaml-to-describe-services)
* [Part 6: The Need for Speed](http://fabien.potencier.org/article/16/symfony-service-container-the-need-for-speed)


`依赖注入` 设计模式非常简单，但又很难解释清楚。造成这个现象的主要原因是，别的介绍 `依赖注入` 的文章里太多废话，让人混淆。下面我将通过一些更适合 PHP 的例子来讲解它。

HTTP 协议是无状态的，我们的 Web 应用程序如果需要在请求之间存储用户信息，可以通过 `COOKIE` 或 `SESSION` ：

```php
$_SESSION['language'] = 'fr';
```

上述代码中，我们将 `language` 存储在全局变量 `$_SESSION` 中，因此可以这样获取它：

```php
$user_language = $_SESSION['language'];
```

只能在 `OOP` 开发时中才会遇到 `依赖注入` ，因此假设我们有一个封装 `SESSION` 的 `SessionStorage` 类：

```php
class SessionStorage
{
    function __construct($cookieName='PHPSESSID')
    {
        session_name($cookieName);
        session_start();
    }

    function set($key, $value)
    {
        $_SESSION[$key] = $value;
    }

    function get($key)
    {
        return $_SESSION[$key];
    }

    // ...
}
```

以及一个更高层的 `User` 类:

```php
class User
{
    protected $storage;

    function __construct()
    {
        $this->storage = new SessionStorage();
    }

    function setLanguage($language)
    {
        $this->storage->set('language', $language);
    }

    function getLanguage()
    {
        return $this->storage->get('language');
    }

    // ...
}
```

这两个类很简单，并且用起来也很方便：

```php
$user = new User();
$user->setLanguage('fr');
$user_language = $user->getLanguage();
```

这种方式看起来很完美，但是并不够灵活。比如：现在想修改会话的 `COOKIE` 名称（默认为 `PHPSESSID`） ，怎么办？这时有一大堆办法：

* 把会话的 `COOKIE` 名称写死在 `User` 类中 `SessionStorage` 的构造函数中 (Hardcode)：

```php
class User
{
    function __construct()
    {
        $this->storage = new SessionStorage('SESSION_ID');
    }

    // ...
}
```

* 或者在 `User` 类外面定义一个常量：

```php
class User
{
    function __construct()
    {
        $this->storage = new SessionStorage(SESSION_COOKIE_NAME);
    }

    // ...
}

define('SESSION_COOKIE_NAME', 'SESSION_ID');
```

* 或者把会话的 `COOKIE` 名称作为 `User` 类构造函数的一个参数传进去：

```php
class User
{
    function __construct($cookieName)
    {
        $this->storage = new SessionStorage($cookieName);
    }

    // ...
}

$user = new User('SESSION_ID');
```

* 或者给 `SessionStorage` 类加个选项数组：

```php
class User
{
    function __construct($storageOptions)
    {
        $this->storage = new SessionStorage($storageOptions['cookie_name']);
    }

    // ...
}

$user = new User(['cookie_name' => 'SESSION_ID']);
```

上述方法都很糟糕：
* 把会话的 `COOKIE` 名称写死的话，每次想再改名，都要修改 `User` 类
* 使用常量的话，`User` 类的变化将取决于设置的常量
* 使用参数或者选项数组看起来很灵活，但它把与 `User` 本身无关的东西掺杂在了构造函数中

### 通过构造函数，把一个外部的 `SessionStorage` 实例"注入"进 `User` 实例内部，而不是在 `User` 实例内部创建 `SessionStorage` 实例，就是 `依赖注入`。

```php
class User
{
    function __construct($storage)
    {
        $this->storage = $storage;
    }

    // ...
}
```

很清爽吧！只需先创建 `SessionStorage` 实例，再创建 `User` 实例：

```php
$storage = new SessionStorage('SESSION_ID');
$user = new User($storage);
```

用这个方法，配置 `SessionStorage` 很简单，给 `User` 替换 `$storage` 类型也很简单，都不需要去修改 `User` 类。这就实现了解耦。

`依赖注入` 并不仅于构造函数：

* Constructor Injection:
```php
class User
{
    function __construct($storage)
    {
        $this->storage = $storage;
    }

    // ...
}
```

* Setter Injection:
```php
class User
{
    function setSessionStorage($storage)
    {
        $this->storage = $storage;
    }

    // ...
}
```

* Property Injection:
```php
class User
{
    public $sessionStorage;
}

$user->sessionStorage = $storage;
```

作为经验， `Constructor 注入` 最适合必须的依赖关系，比如示例中的情况； `Setter 注入` 最适合可选依赖关系，比如缓存一个对象示例。

现在，大多数现代 PHP 框架都大量使用依赖注入来提供一组 `去耦` 但 `粘合` 的组件：

```php
// symfony: A constructor injection example
$dispatcher = new sfEventDispatcher();
$storage = new sfMySQLSessionStorage([
    'database' => 'session',
    'db_table' => 'session',
]);
$user = new sfUser($dispatcher, $storage, ['default_culture' => 'en']);

// Zend Framework: A setter injection example
$transport = new Zend_Mail_Transport_Smtp('smtp.gmail.com', [
  'auth'     => 'login',
  'username' => 'foo',
  'password' => 'bar',
  'ssl'      => 'ssl',
  'port'     => 465,
]);

$mailer = new Zend_Mail();
$mailer->setDefaultTransport($transport);
```

如果你有兴趣了解更多有关 `依赖注入` 的信息，我强烈建议你阅读 [Martin Fowler 的介绍](http://www.martinfowler.com/articles/injection.html) 或 [Jeff Moore 的 PPT](http://www.procata.com/talks/phptek-may2007-dependency.pdf)。你还可以看看我去年关于 `依赖注入` 的 [PPT](http://fabien.potencier.org/talk/19/decouple-your-code-for-reusability-ipc-2008)，其中我详细了介绍本文中所讨论的示例。  

希望本文能让你更好的理解 `依赖注入`，在本系列的下一部分中，我将讨论 `依赖注入容器` (Dependency Injection Containers)。


