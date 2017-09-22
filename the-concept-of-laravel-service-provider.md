# Laravel Service Provider 概念详解

我们知道， `Container` 有很多种 「绑定」 的姿势，比如 `bind()` , `extend()` , `singleton()` , `instance()` 等等，那么 `Laravel` 中怎样「注册」这些「绑定」呢？那就是 `Service Provider`。

先看下 `Laravel` [文档](https://laravel.com/docs/5.5/providers)中这句话：

> Service providers are the central place of all Laravel application bootstrapping. Your own application, as well as all of Laravel's core services are bootstrapped via service providers.

`Service Providers (服务提供者)` 是 `Laravel` 「引导」过程的核心。这个「引导」过程可以理解成「电脑从按下开机按钮到完全进入桌面」这段时间系统干的事。

## 概览
`Service Provider` 有两个重要的方法：
```php
namespace App\Providers;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    /**
     * 注册服务.
     *
     * @return void
     */
    public function register()
    {
        //
    }

    /**
     * 引导服务。
     *
     * @return void
     */
    public function boot()
    {
        //
    }
}
```

`Laravel` 在「引导」过程中干了两件重要的事：
1. 通过 `Service Provider` 的 `register()` 方法注册「绑定」
2. 所有 `Servier Provider` 的 `register()` 都执行完之后，再通过它们 `boot()` 方法，干一些别的事。

## 过程分析
这个「先后顺序」可以在 `Laravel` 的启动过程中找到：
1. 首先，生成核心 `Container` : `$app` (实例化过程中还注册了一大堆基本的「绑定])
```php
// public/index.php
/*
|--------------------------------------------------------------------------
| Turn On The Lights
|--------------------------------------------------------------------------
|
*/
$app = require_once __DIR__.'/../bootstrap/app.php';
```

```php
// bootstrap/app.php
$app = new Illuminate\Foundation\Application(
    realpath(__DIR__.'/../')
);
```

2. 接下来注册 `Http\Kernel` , `Console\Kernel` , `Debug\ExecptionHandler` 三个「单例」绑定：
```php
// bootstrap/app.php
$app->singleton(
    Illuminate\Contracts\Http\Kernel::class,
    App\Http\Kernel::class
);

$app->singleton(
    Illuminate\Contracts\Console\Kernel::class,
    App\Console\Kernel::class
);

$app->singleton(
    Illuminate\Contracts\Debug\ExceptionHandler::class,
    App\Exceptions\Handler::class
);
```
3. 然后「启动」应用：
```php
// public/index.php

/*
|--------------------------------------------------------------------------
| Run The Application
|--------------------------------------------------------------------------
|
*/
$kernel = $app->make(Illuminate\Contracts\Http\Kernel::class);

$response = $kernel->handle(
    $request = Illuminate\Http\Request::capture()
);
```
4. 由于以前的「绑定」，`$kernel` 获取的其实是 `App\Http\Kernel` 类的实例，`App\Http\Kernel` 类又继承了 `Illuminate\Foundation\Http\Kernel` 类。
其 `handle()` 方法执行了 `sendRequestThroughRouter()` 方法：
```php
// Illuminate\Foundation\Http
class Kernel implements KernelContract
{
    public function handle($request)
        try {
            //...
            $response = $this->sendRequestThroughRouter($request);

        } catch (Exception $e) {
            //...
        }
    }
}
```
5. 这个 `sendRequestThroughRouter()` 方法执行了一系列 `bootstrappers (引导器)`：
```php
// Illuminate\Foundation\Http
class Kernel implements KernelContract
{
    protected function sendRequestThroughRouter($request)
    {
        //...

        // 按顺序执行每个 bootstrapper
        $this->bootstrap();

        //...
    }
}
```
6. `bootstrap()` 方法中又调用了 `Illuminate\Foundation\Application` 类的 `bootstrapWith()` 方法：
```php
// Illuminate\Foundation\Http
class Kernel implements KernelContract
{
    public function bootstrap()
    {
        if (! $this->app->hasBeenBootstrapped()) {
            //
            $this->app->bootstrapWith($this->bootstrappers());
        }
    }
}
```

```php
// Illuminate\Foundation\Http\Kernel
class Kernel implements KernelContract
{
    protected $bootstrappers = [
        //...

        \Illuminate\Foundation\Bootstrap\RegisterProviders::class,  // 注册 Providers
        \Illuminate\Foundation\Bootstrap\BootProviders::class,  // 引导 Providers
    ];

    protected function bootstrappers()
    {
        return $this->bootstrappers;
    }
}
```

这里就能看出来 `Service Provider` 中 `register` 和 `boot` 的「先后顺序了」。

> 后续的执行过程暂时先不介绍了。

这就意味着，在 `Service Provider` `boot` 之前，已经把注册好了所有服务的「绑定」。因此， 在 `boot()` 方法中可以使用任何已注册的服务。
例如：
```php
{
namespace App\Providers;
use Illuminate\Support\ServiceProvider;

class ComposerServiceProvider extends ServiceProvider
{
    public function boot()
    {
        // 这里使用 make() 方法，可以更直观
        $this->app->make('view')->composer('view', function () {
            //
        });

        // 或者使用 view() 辅助函数
        view()->composer('view', function () {
            //
        });
    }
}
```

如果了解 `Container` 「绑定」 和 `make` 的概念，应该很容易理解上面的过程。

剩下的无非就是：
### 创建 `Service Provier`
```bash
php artisan make:provider MyServiceProvider
```
### [注册绑定](https://laravel.com/docs/5.5/providers#writing-service-providers)
```php
namespace App\Providers;
use Illuminate\Support\ServiceProvider;

class MyServiceProvider extends ServiceProvider
{
    public function register()
    {
        $this->app->bind(MyInterface::class, MyClass::class);
    }
}
```
### [引导方法](https://laravel.com/docs/5.5/providers#writing-service-providers)
```php
namespace App\Providers;
use Illuminate\Support\ServiceProvider;
use Illuminate\Contracts\Routing\ResponseFactory;

class MyServiceProvider extends ServiceProvider
{

    /**
    * boot() 方法中可以使用依赖注入。
    * 这是因为在 Illuminate\Foundation\Application 类中，
    * 通过 bootProvider() 方法中的 $this->call([$provider, 'boot']) 
    * 来执行 Service Provider 的 boot() 方法
    * Container 的 call() 方法作用，可以参考上一篇文章
    */
    public function boot(ResponseFactory $response)
    {
        $response->macro('caps', function ($value) {
            //
        });
    }
}
```

### 在 `config/app.php` 中注册 `Service Provider`
```php
'providers' => [
  
        /*
         * Laravel Framework Service Providers...
         */
  
        /*
         * Package Service Providers...
         */
  
        /*
         * Application Service Providers...
         */
        App\Providers\MyServiceProvider::class,
],
```

> 注意：`Laravel 5.5` 之后有 [package discovery](https://laravel.com/docs/5.5/packages#package-discovery) 功能的 `package` 不需要在 `config/app.php` 中注册。

### [延时加载](https://laravel.com/docs/5.5/providers#deferred-providers)
按需加载，只有当 `Container` 「make」 `ServiceProvider` 类 `providers()`方法中返回的值时，才会加载此 `ServiceProvider`。
例如：
```php
namespace Illuminate\Hashing;

use Illuminate\Support\ServiceProvider;

class HashServiceProvider extends ServiceProvider
{
    /**
     * 如果延时加载，$defer 必须设置为 true 。
     *
     * @var bool
     */
    protected $defer = true;

    /**
     * Register the service provider.
     *
     * @return void
     */
    public function register()
    {
        $this->app->singleton('hash', function () {
            return new BcryptHasher;
        });
    }

    /**
     * Get the services provided by the provider.
     *
     * @return array
     */
    public function provides()
    {
        return ['hash'];
    }
}
```
当我们「make」时，才会注册 `HashServiceProvider`，即执行它的 `register()` 方法，进行 `hash` 的绑定：
```php
class MyController extends Controller
{
    public function test()
    {
        // 此时才会注册 `HashServiceProvider`
        $hash = $this->app->make('hash');

        $hash->make('teststring');

        // 或
        \Hash::make('teststring');
    }
}
```

以上就是 `Laravel Service Provider` 概念的简单介绍，更深入的了解可以看看「點燈坊」的这篇文章：
[http://oomusou.io/laravel/laravel-service-provider/](http://oomusou.io/laravel/laravel-service-provider/) 。
