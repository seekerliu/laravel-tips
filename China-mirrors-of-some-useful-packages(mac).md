众所周知，在使用各种包管理工具的时候，下载的速度简直慢得跟蜗牛一样，甚至还有可能出现中断错误的情况，这个时候使用国内的镜像源是一个不错的选择。

本文档试用与Mac OS，终端是zsh

## Homebrew使用国内镜像
(1) 使用中科大源

* 第一步：替换brew.git
```shell
cd "$(brew --repo)"
git remote set-url origin https://mirrors.ustc.edu.cn/brew.git
```
* 第二步：替换homebrew-core.git
```shell
cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core"
git remote set-url origin https://mirrors.ustc.edu.cn/homebrew-core.git
cd 
brew update
```

(2) 替换Homebrew Bottles源

Homebrew是OS X系统的一款开源的包管理器。出于节省时间的考虑，Homebrew默认从Homebrew Bottles源中下载二进制代码包安装。Homebrew Bottles是Homebrew提供的二进制代码包，目前镜像站收录了以下仓库：

```
homebrew/homebrew-core
homebrew/homebrew-dupes
homebrew/homebrew-games
homebrew/homebrew-gui
homebrew/homebrew-python
homebrew/homebrew-php
homebrew/homebrew-science
homebrew/homebrew-versions
homebrew/homebrew-x11
```
打开终端或者iterms2
```shell
echo 'export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.ustc.edu.cn/homebrew-bottles' >> ~/.zshrc
source ~/.zshrc
```

使用清华源

（1）替换默认源
* 第一步：替换现有上游
```shell
cd "$(brew --repo)"
git remote set-url origin https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/brew.git
cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core"
git remote set-url origin https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-core.git
cd 
brew update
```
* 第二步：使用homebrew-science或者homebrew-python

```shell
cd "$(brew --repo)/Library/Taps/homebrew/homebrew-science"
git remote set-url origin https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-science.git
```
或

```shell
cd "$(brew --repo)/Library/Taps/homebrew/homebrew-python"
git remote set-url origin https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-python.git
cd 
brew update
```
（2）替换Homebrew Bottles源

```shell
echo 'export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.tuna.tsinghua.edu.cn/homebrew-bottles' >> ~/.bash_profile
source ~/.bash_profile
```

3.在中科大源或清华源失效或宕机时可以切换回官方源
第一步：重置brew.git

```shell
cd "$(brew --repo)"
git remote set-url origin https://github.com/Homebrew/brew.git
```
第二步：重置homebrew-core.git

```shell
cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core"
git remote set-url origin https://github.com/Homebrew/homebrew-core.git
cd
brew update
```
第三步：注释掉配置文件里的有关Homebrew Bottles即可恢复官方源。 重启终端或iterms2重读配置文件。
## Composer使用国内镜像

* 全局配置

```shell
composer config -g repo.packagist composer https://packagist.laravel-china.org
```

* 单独使用

```shell
composer config repo.packagist composer https://packagist.laravel-china.org
```

* 取消镜像

```shell
composer config -g --unset repos.packagist
```
这时候创建laravel项目的时候用下面的命令：
```shell
composer -vvv create-project laravel/laravel blog
```
## npm使用国内镜像

使用定制的 cnpm (gzip 压缩支持) 命令行工具代替默认的 npm

* 临时使用

```shell
npm install -g cnpm --registry=https://registry.npm.taobao.org
```

* 写进alias

```shell
echo '\n#alias for cnpm\nalias cnpm="npm --registry=https://registry.npm.taobao.org \
  --cache=$HOME/.npm/.cache/cnpm \
  --disturl=https://npm.taobao.org/dist \
  --userconfig=$HOME/.cnpmrc"' >> ~/.zshrc && source ~/.zshrc
```
## iTerms2使用终端通过ss代理

前提是shadow socks已经打开到pac自动模式

直接将下面的命令复制粘贴到~/.zshrc 文件的末尾

```shell
# proxy list
alias proxy='export all_proxy=socks5://127.0.0.1:1086'
alias unproxy='unset all_proxy
```

重启iterm2或者终端生效。
测试是否生效使用下面的命令：

* 打开代理前输入下面的命令看你访问的IP地址

```shell
curl cip.cc
```

* 打开代理：

```
proxy
```

* 再看你的IP地址：

```shell
curl cip.cc
```

* 关闭代理：

```shell
unproxy
```


