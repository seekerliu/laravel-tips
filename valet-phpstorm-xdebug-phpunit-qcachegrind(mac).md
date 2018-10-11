# Laravel 配置 valet, PHPStorm, xdebug, PHPUnit, qcachegrind(mac)环境

* 可使用 `command+,` 快捷键打开 PHPStorm Preferences页面
* 在 Preferences 页面可以直接搜索要配置的项目
* 可使用 `control+alt+d` 快捷键打开 PHPStorm Debug 设置

## 一、配置单 php 文件的 Debug
```
1、PHPStorm: Preferences > Languages & Frameworks > PHP
    > PHP language level: 7.2
    > CLI interpreter: PHP 7.2.4

2、PHPStorm: Run > Edit Configurations.. > + > PHP Script, 添加 php 文件。
    或直接在要调试的 php 文件中，`control+alt+d` > xxx.php(PHP Script)
```

## 二、配置 Laravel 的 Debug
1、修改 xdebug.ini
```
xdebug.remote_autostart=1
xdebug.default_enable=1
xdebug.remote_port=9001 ;xdebug的端口，默认为9000，我习惯用9001
xdebug.remote_host=127.0.0.1
xdebug.remote_connect_back=1
xdebug.remote_enable=1
xdebug.idekey=PHPSTORM
```

如需开启性能分析，还需添加下列信息，开启后会严重拖慢 laravel 打开速度
```
zend_debugger.allow_hosts=127.0.0.1
zend_debugger.expose_remotely=always
zend_debugger.httpd_uid=-1
xdebug.trace_format = 2
xdebug.auto_trace = on
xdebug.auto_profile = on ;打开性能分析
xdebug.collect_params = on
xdebug.collect_return = on
xdebug.profiler_enable = 1
xdebug.trace_output_dir = /Users/seekerliu/Documents/xdebug/trace ;trace生成的文件目录, 需配置成自己的
xdebug.profiler_output_dir=/Users/seekerliu/Documents/xdebug ;性能分析生成的文件目录, 需配置成自己的
xdebug.trace_output_name = trace.%c.%p
xdebug.show_exception_trace = On ;开启异常跟踪
xdebug.show_local_vars = 0
 
xdebug.profiler_output_name=cachegrind.out.%s
xdebug.dump.GET = *
xdebug.dump.POST = *
xdebug.dump.COOKIE = *
xdebug.dump.SESSION = *
xdebug.var_display_max_data = 4056
xdebug.var_display_max_depth = 5
```

2、 重启 php-fpm
```
➜ valet restart
或 找到 php-fpm master progress 进程 pid (例如 2345) 后
➜ ps aux | grep php-fpm
➜ kill -USR2 2345
```

3、配置 PhpStorm
```
1) PHPStorm: Preferences > DBGp Proxy > IDE key: PHPSTROM > PORT:9001

2) Languages & Frameworks > PHP > Debug > Xdebug > Debug port: 9001

3) PHPStorm: Run > Edit Configurations.. > + > PHP Web Page
> 填写 Name, Configuration: 选择 Server, 填写 Start URL。
> 添加 Server 时，填写 Name, Host, Port, Debugger:Xdebug。
> 设置 path mappings: √ Use path mappings, 
> 编辑 File/Directory(本地项目文件) 与 Absolute path on the server(服务器上的文件绝对路径) 的映射关系，例如：
    File/Directory: /Users/seekerliu/Documents/sxmsV2
    Absolute path on the server: /Users/seekerliu/Documents/sxmsV2
```

4、对于 valet 项目，还需配置 valet 路径至 include path:
```
PHPStorm: Project Window(左边栏项目资源目录，可使用 command+1 快捷键打开) 
> Extrenal Libraries > Config PHP Include Paths
> Add 
> /Users/seekerliu/.composer/vendor/laravel/valet (换成你自己的)
```

5、监听来自页面的 xdebug 请求，即浏览器中打开网页后自动跳转到 PHPStorm 的断点上：

首先在 PHPStorm 中开启监听:
```
 Run > Start Listening for PHP Debug Connections
```

然后再 Web 端开启 XDEBUG_SESSION：

方法一：

在请求 URL 中添加 `XDEBUG_SESSION_START` 参数（可以为任意值, 例如：`XDEBUG_SESSION_START=1`)，

代码中设置断点，然后访问URL：
```
http://website.test/?XDEBUG_SESSION_START=1
```

即可进入设置过的断点。

方法二：

`cookie` 中设置 `XDEBUG_SESSION`。使用方法一就会自动设置此 `cookie` 值，有效期 1小时。

Laravel 项目中可以如下配置：
```
\App\Providers\AppServiceProvider.class.php

public function register()
{
    // 各类开发环境中使用的配置
    if ($this->app->environment() == 'local') {
        // 注册各类开发时用到的 package
        // $this->app->register('Seekerliu\ViewComposerGenerator\ServiceProvider');

        // 开启 XDEBUG_SESSION
        // PHPStorm 中需执行：Run > Start Listening for PHP Debug Connections
        if(!request()->hasCookie('XDEBUG_SESSION')) {
            $cookie = \Cookie::forever('XDEBUG_SESSION', 1);
            \Cookie::queue($cookie);
        }
    }
}
```

## 三、配置 PHP Unit
1、安装phpunit
```
➜ wget https://phar.phpunit.de/phpunit.phar
➜ chmod +x phpunit.phar
➜ php phpunit.phar --version
>> PHPUnit 6.3.0 by Sebastian Bergmann and contributors.

➜ sudo mv phpunit.phar /usr/local/bin/phpunit
➜ phpunit --version
>> PHPUnit 6.3.0 by Sebastian Bergmann and contributors.
```

2、配置 `~/.bash_profile`
```
# PHP Unit
t() {
  if [ -f vendor/bin/phpunit ]; then
    vendor/bin/phpunit "$@"
  else
    phpunit "$@"
  fi
}
```

这样在 Terminal 中可以使用 t 直接运行。
```
➜ t  --stderr
```

3、配置 PHPStorm 中 phpunit 路径
```
PHPStorm: Preferences > Test Frameworks > + > PHP Unit Local 
> Path to phpunit.phar: /usr/local/bin/phpunit
> Default configuration file: /Laravel项目目录/phpunit.xml 
> Default bootstrap file: /Laravel项目目录/vendor/autoload.php
```

这样就可以在 PHPStorm 中对 Laravel Test 文件使用 `control+alt+d` 执行 PHPUnit 了。

四、安装qcachegrind(mac下的图形化性能分析软件)
```
➜ brew install graphviz
➜ brew install qcachegrind
➜ qcachegrind
```
还可以把 qcachegrind 添加到应用程序中：
```
➜ cp -r /usr/local/Cellar/qcachegrind/17.04.1(版本号)/qcachegrind.app ~/Applications
```
解决 Call Graph 无法使用的问题：
```
sudo ln -s /usr/local/bin/dot /usr/bin/dot
```

以上命令会报错，因为mac系统有完整性保护，mac系统一些重要的系统目录禁止修改，即使是切换到root也不行。

这时，可暂时修改：
```
1、关机后，开机的同时或者在听到开始音的同时，按住Command+R键
2、打开 Terminal 窗口输入命令：csrutil disable 即可；可以查看状态：csrutil status
3、重启电脑后再执行命令：sudo ln -s /usr/local/bin/dot /usr/bin/dot
4、重复 1 步骤，2 步骤命令替换为：csrutil enable  恢复默认的mac权限
```
