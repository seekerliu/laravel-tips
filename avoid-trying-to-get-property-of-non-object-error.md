# Laravel 避免 Trying to get property of non-object 错误的六种方法

在使用链式操作的时候，例如：

```
return $user->avatar->url;
```

如果 `$user->avatar` 为 `null`，就会引起 `(E_ERROR)
Trying to get property 'url' of non-object` 错误。

## 1. 常规方法是使用 `isset` 加以判断：
```
if(isset($user->avatar->url))
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

>  `Laravel 5.7` 已经取消了这个特性。详见：[https://github.com/laravel/framework/pull/23532](https://github.com/laravel/framework/pull/23532) 。感谢 [@jltxwesley](https://laravel-china.org/users/28596) 提醒。

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

## 4. 使用 `object_get` 辅助函数
```
return object_get($user->avatar, 'url', 'default');
```

这个函数原意是用来以 `.` 语法来获取对象中的属性，例如：

```
return object_get($user, 'avatar.url', 'default');
```

也可以达到避免 `non-object` 错误的效果。

```
if (! function_exists('object_get')) {
    /**
     * Get an item from an object using "dot" notation.
     *
     * @param  object  $object
     * @param  string  $key
     * @param  mixed   $default
     * @return mixed
     */
    function object_get($object, $key, $default = null)
    {
        if (is_null($key) || trim($key) == '') {
            return $object;
        }

        foreach (explode('.', $key) as $segment) {
            if (! is_object($object) || ! isset($object->{$segment})) {
                return value($default);
            }

            $object = $object->{$segment};
        }

        return $object;
    }
}
```

详见 [https://github.com/laravel/framework/blob/master/src/Illuminate/Support/helpers.php#L673](https://github.com/laravel/framework/blob/master/src/Illuminate/Support/helpers.php#L673)

> 感谢 [@lovecn](https://laravel-china.org/users/87) 提供姿势！

 
## 5. 使用 `data_get` 辅助函数
```
return data_get($user, 'avatar.url', 'default');
```

或

```
return data_get($user, ['avatar', 'url'], 'default');
```

以 `.` 语法来获取对象属性或数组元素。

```
if (! function_exists('data_get')) {
    /**
     * Get an item from an array or object using "dot" notation.
     *
     * @param  mixed   $target
     * @param  string|array  $key
     * @param  mixed   $default
     * @return mixed
     */
    function data_get($target, $key, $default = null)
    {
        if (is_null($key)) {
            return $target;
        }
        $key = is_array($key) ? $key : explode('.', $key);
        while (! is_null($segment = array_shift($key))) {
            if ($segment === '*') {
                if ($target instanceof Collection) {
                    $target = $target->all();
                } elseif (! is_array($target)) {
                    return value($default);
                }
                $result = [];
                foreach ($target as $item) {
                    $result[] = data_get($item, $key);
                }
                return in_array('*', $key) ? Arr::collapse($result) : $result;
            }
            if (Arr::accessible($target) && Arr::exists($target, $segment)) {
                $target = $target[$segment];
            } elseif (is_object($target) && isset($target->{$segment})) {
                $target = $target->{$segment};
            } else {
                return value($default);
            }
        }
        return $target;
    }
}
```

详见 [https://github.com/laravel/framework/blob/master/src/Illuminate/Support/helpers.php#L450](https://github.com/laravel/framework/blob/master/src/Illuminate/Support/helpers.php#L450)

> 感谢 [@Hachiko](https://laravel-china.org/users/22249) 提供姿势！

## 6.除此之外，还可以使用 `Null Object Pattern(空对象模式)`:
[《點燈坊:如何實現 Null Object Pattern ?》](https://old-oomusou.goodjack.tw/design-pattern/nullobject/)

感谢群里大佬 @盒子 和 @Outshine 提供的姿势。:kissing: :kissing: :kissing:

