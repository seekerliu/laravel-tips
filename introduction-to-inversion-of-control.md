# Laravel Inversion of Control (控制反转) 概念简介

>本文内容部分摘自 Wikipedia - [Inversion of Control](https://zh.wikipedia.org/wiki/%E6%8E%A7%E5%88%B6%E5%8F%8D%E8%BD%AC) .

# 概述

IoC (控制反转)，是面向对象编程中的一种设计原则，可以用来减低计算机代码之间的耦合度。

## 实现「控制反转」，有两种方式：
*   Dependency Injection (DI) - 依赖注入
*   Dependency Lookup - 依赖查找

> 两者的区别在于，前者是被动的接收对象，在类实例创建过程中即创建了依赖的对象，通过类型或名称来判断将不同的对象注入到不同的属性中，而后者是主动索取相应类型的对象，获得依赖对象的时间也可以在代码中自由控制。

## 哪些方面的「控制」被「反转」了？

依赖对象的「获得」被反转了。

## 技能描述
Class A 中用到了 Class B 的对象 b，一般情况下，需要在 A 的代码中显式的 new 一个 B 的对象。采用依赖注入技术之后，A 的代码只需要定义一个私有的B对象，不需要直接 new 来获得这个对象，而是通过相关的 `容器控制程序` 来将 B 对象在外部 new 出来并注入到 A 类里的引用中。而具体获取的方法、对象被获取时的状态由 `容器` 来指定。

假设我有一个 iPhone，我的 iPhone 依赖充电器才能充电。
```php
class iPhone
{
    // 电量
    private $power;

    // 充电
    public function charge()
    {
    }
}
```

我还有个苹果原装充电器：
```php
class AppleCharger
{
    public function charge()
    {
        return 100;
    }
}
```

在以前，iPhone 内部「控制」着只能用哪一款充电器：
```php
class iPhone
{
    // 电量
    private $power;

    // 充电，只能用原装的充电器
    public function charge()
    {
        $charger = new AppleCharger;
        $this->power = $charger->charge();
    }
}
```

```php
// 充电
$iphone = new iPhone;
$iphone->charge();
```

使用依赖注入之后，我来决定给 iPhone 用哪一款充电器：
```php
class iPhone
{
    private $power;
    private $charger;

    // 依赖注入充电器，╮(╯▽╰)╭哎算了只要是充电器就行
    public function __construct(Charger $charger)
    {
        $this->charger = $charger;
    }

    // 充电
    public function charge()
    {
        $this->power = $this->charger->charge();
    }
}
```

```php
interface Charger
{
    public function charge();
}
```

```php
// Laravel 容器
use Illuminate\Container\Container;
$container = Container::getInstance();

// 给它一个原装充电器：
$container->bind(Charger::class, AppleCharger::class);

// 或者给它其它充电器
$container->bind(Charger::class, OtherCharger::class);

// 充电
$iphone = $container->make(iPhone::class);
$iphone->charger();
```

可见，使用依赖注入之后，控制权「反转」了，由外部来决定给它什么充电器 (依赖对象)。
`Laravel` 管这个 `容器控制程序` 叫 `Service Container (服务容器)`，它来控制着各种依赖的获取方法。

> 更多关于 Laravel 依赖注入、服务容器的知识可以参考以前的文章。

# 扩展知识
### 依赖注入实现方式：
*   基于构造函数。实现特定参数的构造函数，在新建对象时传入所依赖类型的对象。
*   基于接口。实现特定接口以供外部容器注入所依赖类型的对象。
*   基于 set 方法。实现特定属性的 public set 方法，来让外部容器调用传入所依赖类型的对象。
*   基于注解。基于 Java 的注解功能，在私有变量前加 "@Autowired" 等注解，不需要显式的定义以上三种代码，便可以让外部容器传入对应的对象。该方案相当于定义了 public 的 set 方法，但是因为没有真正的 set 方法，从而不会为了实现依赖注入导致暴露了不该暴露的接口（因为 set 方法只想让容器访问来注入而并不希望其他依赖此类的对象访问）。

### 依赖查找：
依赖查找更加主动，在需要的时候通过调用框架提供的方法来获取对象，获取时需要提供相关的配置文件路径、key等信息来确定获取对象的状态。