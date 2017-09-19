# Laravel 获取 Route Parameters (路由参数) 的 5 种方法

> Laravel 获取路由参数的方式有很多，并且有个小坑，汇总如下。

### 假设我们设置了一个路由参数：
```php
/**
* 定义路由参数名称分别为： param1，param2
*/
Route::get('/{param1}/{param2}', 'TestController@index');
```
### 现在我们访问 `http://test.dev/1/2`
### 在 `TestController` 中：
```php
/**
* 路由参数获取方法
*
* @param Illuminate\Http\Request $request    依赖注入 Request 实例，放在参数中什么位置都可以自动加载
* @param mixed $arg2    要获取的路由参数
* @param mixed $arg1    要获取的路由参数
*/

public function index(Request $request, $arg2, $arg1)
{

    /**
    * 方法一：按照 URL 中路由参数先后顺序来获取
    * 注意：此种方式有个小坑，获取的值只与顺序有关，与名称无关
    */
    echo $arg2;    //结果为 1 ，因为 $arg2 在第一位，获取的是第一个路由参数 param1 的值
    echo $arg1;    //结果为 2 ，因为 $arg1 在第二位，获取的是第二个路由参数 param2 的值

    /**
    * 方法二：按照路由参数名称来获取
    * 注意：此处名称是 Route 中定义的参数名，非上面方法中的参数名 
    */
    $request->route('param1');      //结果为 1 ，获取的是第一个路由参数
    $request->route('param2');      //结果为 2 ，获取的是第二个路由参数

    /**
    * 方法三：使用 request() 辅助函数来获取，效果同方法二
    */
    request()->route('param1');     //结果为 1 ，如果不带路由参数名则返回当前的Route对象
    request()->route('param2');     //结果为 2 ，如果不带路由参数名则返回当前的Route对象

    /**
    * 方法四：使用 Route Facade
    */
    \Route::input('param1');     //结果为 1 ，该方法必须带路由参数名
    \Route::input('param2');     //结果为 2 ，该方法必须带路由参数名

    /**
    * 方法五：使用 Illuminate\Http\Request 实例动态属性
    */
    $request->param1;   //结果为 1 ，Laravel 5.4+ 可用
    $request->param2;   //结果为 2 ，Laravel 5.4+ 可用
        
    // 或者
    request()->param1;   //结果为 1 ，Laravel 5.4+ 可用
    request()->param2;   //结果为 2 ，Laravel 5.4+ 可用
        
    //或者
    request('param1');   //结果为 1 ，Laravel 5.4+ 可用
    request('param2');   //结果为 2 ，Laravel 5.4+ 可用
        
    /**
    * 注意：Laravel 在处理动态属性的优先级是，先从请求的数据（POST/GET）中查找，没有的话再到路由参数中找。
    * 例如：URL ： http://test.dev/1/2?param1=a&param2=b
    * $request->param1; request()->param1; request('param1');    //结果为 a
    * $request->param2; request()->param2; request('param2');    //结果为 b
    */
}
```
以上就是 Laravel 获取路由参数的 5 种方法。