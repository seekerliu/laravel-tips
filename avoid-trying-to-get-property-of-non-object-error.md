# Laravel 避免 Trying to get property of non-object 错误的四种方法

在使用链式操作的时候，例如：

```
return $user->avatar->url;
```

如果 `$user->avatar` 为 `null`，就会引起 `(E_ERROR)
Trying to get property 'url' of non-object` 错误。

## 1. 常规方法是使用 `isset` 加以判断：
```
if(isset($user->avatar->url)
    return $user->avatar->url;
else
    return 'defaultUrl';
```

如果在 `blade` 模板的 `echo` 中，可以使用：

```
{{ $user->avatar->url or 'defaultUrl' }}
```

上述代码会被 `Blade` 引擎解析为：

```
echo e(isset($user->avatar->url) ? $user->avatar->url : 'defaultUrl');
```

## 2. `PHP7` 可以使用 `?? (NULL 合并操作符)` :
```
// 如果 $user->avatar->url 为 null, 返回 'defaultUrl'
return $user->avatar->url ?? 'defaultUrl';
```

## 3. `Laravel 5.5` 及以上可以使用 `optional` 辅助函数：
```
/**
 * 如果给定的对象是 null ， 那么属性和方法会简单地返回 null 而不是产生一个错误：
 */
return optional($user->avatar)->url;
```

详见 [https://laravel-china.org/docs/laravel/5.7/helpers/1320#method-optional](https://laravel-china.org/docs/laravel/5.7/helpers/1320#method-optional)

`Laravel 5.7` 中，`optional` 函数还可以接受 `匿名函数` 作为第二个参数：
```
/**
 * 如果第一个参数不为 null, 则调用闭包
 */
return optional(User::find($id), function ($user) {
    return new DummyUser;
});
```

详见 [https://laravel.com/docs/5.7/helpers#method-optional](详见 https://laravel.com/docs/5.7/helpers#method-optional)

## 4.除此之外，还可以使用 `Null Object Pattern(空对象模式)`:
[《點燈坊:如何實現 Null Object Pattern ?》](https://oomusou.io/design-pattern/nullobject/)

感谢群里大佬 @盒子 和 @Outshine 提供的姿势。:kissing: :kissing: :kissing:

