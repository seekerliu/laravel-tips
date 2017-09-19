# Laravel Container (容器) 概念详解 (上)

>本文翻译自 `Symfony` 作者 Fabien Potencier 的 [《Dependency Injection in general and the implementation of a Dependency Injection Container in PHP》](http://fabien.potencier.org/what-is-dependency-injection.html) 系列文章。

* [Part 1: What is Dependency Injection?](http://fabien.potencier.org/article/11/what-is-dependency-injection)
* [Part 2: Do you need a Dependency Injection Container?](http://fabien.potencier.org/article/12/do-you-need-a-dependency-injection-container)
* [Part 3: Introduction to the Symfony Service Container](http://fabien.potencier.org/article/13/introduction-to-the-symfony-service-container)
* [Part 4: Symfony Service Container: Using a Builder to create Services](http://fabien.potencier.org/article/14/symfony-service-container-using-a-builder-to-create-services)
* [Part 5: Symfony Service Container: Using XML or YAML to describe Services](http://fabien.potencier.org/article/15/symfony-service-container-using-xml-or-yaml-to-describe-services)
* [Part 6: The Need for Speed](http://fabien.potencier.org/article/16/symfony-service-container-the-need-for-speed)

>专有名词翻译成中文后会变得不利于理解，后续文章中将改用括号+中文备注的形式。


上文我通过一些示例讲解了 `Dependency Injection` ，本文将接着介绍 `Dependency Injection Containers (容器)` 的概念。

首先记住这句话：

>大多数时候，`Dependency Injection` 并不需要 `Container`。

只有当你需要管理一大堆具有很多依赖关系的不同对象时，`Container` 才会非常有用（例如框架中）。 

上文书，创建 `User` 对象需要先创建 `SessionStorate` 对象。这里的有个瑕疵，创建对象时需要提前知道它所有的依赖项：
```php
$storage = new SessionStorage('SESSION_ID');
$user = new User($storage);
```

以 `Zend Framework` 中 `Zend_Mail` 库发送邮件过程为例：
```php
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

>请把这个例子看做一个大系统中的一小部分，因为这种简单的例子当然没必要用 `Container` 。

`Dependency Injection Container` 是一个“知道如何实例化和配置对象”的对象（工厂模式的升华）。为了做到这点，它需要知道构造函数的参数、以及对象之间的关系。

下面是一个写死 `Zend_Mail` 的 `Container`：
```php
class Container
{
  public function getMailTransport()
  {
    return new Zend_Mail_Transport_Smtp('smtp.gmail.com', [
      'auth'     => 'login',
      'username' => 'foo',
      'password' => 'bar',
      'ssl'      => 'ssl',
      'port'     => 465,
    ]);
  }

  public function getMailer()
  {
    $mailer = new Zend_Mail();
    $mailer->setDefaultTransport($this->getMailTransport());

    return $mailer;
  }
}
```

这个 `Container` 用起来就相当简单了：
```php
$container = new Container();
$mailer = $container->getMailer();
```

我们只管向 `Container` 要 `mailer` 对象就行，完全不用管 `mailer` 怎么创建。创建 `mailer` 对象的“杂活”是嵌入在 `Container` 中的。
`Container` 通过 `getMailTransport()` 方法，把 `Zend_Mail_Transport_Smtp` 这个依赖自动注入到了 `Zend_Mail` 中。

细心的网友可能已经发现，这里的 `Container` 把什么都写死了。我们可以完善一下：
```php
class Container
{
  protected $parameters = array();

  public function __construct(array $parameters = [])
  {
    $this->parameters = $parameters;
  }

  public function getMailTransport()
  {
    return new Zend_Mail_Transport_Smtp('smtp.gmail.com', [
      'auth'     => 'login',
      'username' => $this->parameters['mailer.username'],
      'password' => $this->parameters['mailer.password'],
      'ssl'      => 'ssl',
      'port'     => 465,
    ]);
  }

  public function getMailer()
  {
    $mailer = new Zend_Mail();
    $mailer->setDefaultTransport($this->getMailTransport());

    return $mailer;
  }
}
```

现在就可以随时更改 `username` 和 `password` 了：
```php
$container = new Container([
  'mailer.username' => 'foo',
  'mailer.password' => 'bar',
]);
$mailer = $container->getMailer();
```

如果需要更改 `mailer` 类，把类名也当参数传入就行：
```php
class Container
{
  // ...

  public function getMailer()
  {
    $class = $this->parameters['mailer.class'];

    $mailer = new $class();
    $mailer->setDefaultTransport($this->getMailTransport());

    return $mailer;
  }
}

$container = new Container([
  'mailer.username' => 'foo',
  'mailer.password' => 'bar',
  'mailer.class'    => 'Zend_Mail',
]);
$mailer = $container->getMailer();
```

如果想每次获取同一个 `mailer` 实例，可以用 `单例模式`：
```php
class Container
{
  static protected $shared = [];

  // ...

  public function getMailer()
  {
    if (isset(self::$shared['mailer']))
    {
      return self::$shared['mailer'];
    }

    $class = $this->parameters['mailer.class'];

    $mailer = new $class();
    $mailer->setDefaultTransport($this->getMailTransport());

    return self::$shared['mailer'] = $mailer;
  }
}
```

这就包含了 `Dependency Injection Containers` 的基本功能：
* `Container` 管理对象实例化到配置的过程
* 对象本身不知道自己是由 `Container` 管理的，对 `Container` 一无所知。 

这就是为什么 `Container` 能够管理任何 PHP 对象。 对象使用 `DI` 来管理依赖关系非常好，但不是必须的。

`Container` 很容易实现，但手工维护各种乱七八糟的对象还是很麻烦。下一章我将介绍 `Laravel` 中 `Container` 的实现方式。

>作者下一章原文中讲的是 `Container` 在 `Symfony 2` 中的实现，我会把它换成 `Laravel`。






