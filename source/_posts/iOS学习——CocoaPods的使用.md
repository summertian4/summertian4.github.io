title: iOS学习——CocoaPods的使用
date: 2015-10-27 09:58:46
tags:
  - iOS
categories:
  - iOS
---

# CocoaPods 
现在做任何开发都免不了使用第三方库，iOS开发也不例外，每天去逛一逛github成了不可少的必修课。
通常，一些简单的框架拖入相应的.h、.m文件就可以使用了。比如[ProgressHUD](https://github.com/relatedcode/ProgressHUD)、[JSONKit](https://github.com/johnezang/JSONKit)。但是有时候会出现这样的情况：

1. `三方库需要依赖系统的一些framework，需要手动的添加`
2. `三方库又使用了三方库，需要再去添加`

这时候，添加库就变成了十分麻烦的事。
使用 CocoaPods可以省去这些麻烦，只需要把你想添加的库写在一个文件里，CocoaPods可以帮你一键添加、配置。

----

# 安装
CocoaPods安装需要ruby环境，但是 Mac 本身已经安装好了 ruby

需要注意的是，ruby 的软件源 https://rubygems.org 被墙了，所以需要更新 ruby 的源换成源替换成国内淘宝的源 `https://ruby.taobao.org/` ，在网上看到很多人写成了http://ruby.taobao.org/，`注意这里一定是https`。

```
gem sources --remove https://rubygems.org/
gem sources -a https://ruby.taobao.org/
```

再执行：

```
gem sources -l
```

如果出现了下图，就说明替换成功了
![成功替换](http://img.blog.csdn.net/20151027101252947)

那么，你只需要在终端：

```
sudo gem install cocoapods
pod setup
```

等待一会CocoaPods就安装好了。

----

# 使用

新建一个名为 Podfile 的纯文本文件，没有后缀名，放到你项目的根目录。在文件中写入你需要加载的三方库

```
pod 'JSONKit',       '~> 1.4'
pod 'Masonry'
pod 'FMDB'
```
一般在三方库的readme文件中也会告诉你需要写什么，比如：
![第三方库Masonry的reademe文件](http://img.blog.csdn.net/20151027101816304)

然后在终端输入

```
cd "your project home"
pod install
```

等待一会，CocoaPods会把你需要的所有第三方库下载并且设置好编译参数和依赖。这时会发现项目多了一个.xcworkspace文件，`记住`，以后都要使用.xcworkspace 文件来打开工程。

# 更新
如果你又添加新的三方库，可以在Podfile文件中继续添加，然后再终端输入下面的命令CocoaPods就会重新帮你导入。

```
pod update
```

----

# 其他问题
之前遇到一个前辈，他们公司刚刚使用CocoaPods（别问我为啥，我也奇了怪了）。然后他们`pod update`之后，出现了大量的错误。原因是他们之前使用的三方库太老了，早就已经更新，连方法名都变了。为了预防这一类错误，在Podfile中可以指定版本，像这样：

```
pod 'JSONKit',       '~> 1.4'
```

----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)


