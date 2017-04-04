title: iOS攻防——（四）class-dump 与 Dumpdecrypted 使用
date: 2016-08-30 13:09:36
tags:
  - iOS
  - iOS攻防
categories:
  - iOS
  - iOS攻防
---

# 1 class dump

class dump 是一个用于检查保存在 Mach-O 文件中的 objective-c 运行时信息的工具，攻防中最常用、实用的命令行工具。

## 1.1 class dump 好玩在哪？

class dump 绝对可以满足你的好奇心。你可以通过 class dump ：

1. 查看闭源的应用、frameworks、bundles。
2. 对比一个 APP 不同版本之间的接口变化。
3. 对一些私有 frameworks 做些有趣的试验。

## 1.2 Download

当前版本: 3.5 (64 bit Intel)
需要 Mac OS X 10.8 或更高版本

[class-dump-3.5.dmg](http://stevenygard.com/download/class-dump-3.5.dmg)
[class-dump-3.5.tar.gz](http://stevenygard.com/download/class-dump-3.5.tar.gz)
[class-dump-3.5.tar.bz2](http://stevenygard.com/download/class-dump-3.5.tar.bz2)

## 1.3 Use

<!--more-->

下载好后，双击dmg文件，将其中的 class-dump 文件放到/usr/local/sbin 目录下，然后就可以在命令行中使用了。

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E6%94%BB%E9%98%B2%E2%80%94%E2%80%94%EF%BC%88%E5%9B%9B%EF%BC%89class%20dump%20%E4%BD%BF%E7%94%A8-01.png)

官方用法指南：

```
class-dump 3.5 (64 bit)
Usage: class-dump [options] <mach-o-file>

  where options are:
        -a             show instance variable offsets
        -A             show implementation addresses
        --arch <arch>  choose a specific architecture from a universal binary (ppc, ppc64, i386, x86_64)
        -C <regex>     only display classes matching regular expression
        -f <str>       find string in method name
        -H             generate header files in current directory, or directory specified with -o
        -I             sort classes, categories, and protocols by inheritance (overrides -s)
        -o <dir>       output directory used for -H
        -r             recursively expand frameworks and fixed VM shared libraries
        -s             sort classes and categories by name
        -S             sort methods by name
        -t             suppress header in output, for testing
        --list-arches  list the arches in the file, then exit
        --sdk-ios      specify iOS SDK version (will look in /Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS<version>.sdk
        --sdk-mac      specify Mac OS X version (will look in /Developer/SDKs/MacOSX<version>.sdk
        --sdk-root     specify the full SDK root path (or use --sdk-ios/--sdk-mac for a shortcut)
```
        
简单的举例：

```
class-dump -H /Applications/Calculator.app -o ~/Desktop/dump/Calculate-dump
```

`/Applications/Calculator.app` 是 Mac 上计算器应用的路径。
`~/Desktop/dump/Calculate-dump` 是指定的存放 dump 出来头文件的文件夹路径。

执行完成后可以看到指定的保存目录下已经有 dump 出来的头文件了：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E6%94%BB%E9%98%B2%E2%80%94%E2%80%94%EF%BC%88%E5%9B%9B%EF%BC%89class%20dump%20%E4%BD%BF%E7%94%A8-02.png)

打开一个 `.h` 文件可以看到相应内容：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E6%94%BB%E9%98%B2%E2%80%94%E2%80%94%EF%BC%88%E5%9B%9B%EF%BC%89class%20dump%20%E4%BD%BF%E7%94%A8-03.png)

# 2 Dumpdecrypted

class dump 虽然能帮你解析出一个 app 的头文件，但是对于 App Store 下载的 App 都是通过苹果的一层签名加密，通常我们成为『加壳』。对于已经加壳的 APP，解析后的效果就和加了代码混淆类似。

比如直接去 dump 微信，出来的结果大概是这样：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E6%94%BB%E9%98%B2%E2%80%94%E2%80%94%EF%BC%88%E5%9B%9B%EF%BC%89class%20dump%20%E4%BD%BF%E7%94%A8-04.png)

## 2.1 Download

[dumpdecrypted GitHub 地址](https://github.com/stefanesser/dumpdecrypted)

## 2.2 Install

Dumpdecrypted 比另一个砸壳工具 Clutch 要难用的多。但由于 Clutch 无法砸开含有兼容 WatchOs
2 的 App，所以只能使用 Dumpdecrypted。

下载后打开 `Makefile` 文件，注意第三行：

```
$ SDK=`xcrun --sdk iphoneos --show-sdk-path`
```

这里填写的 SDK 必须与你越狱的 iPhone 系统等级一致，你可以这样查看你 MAC 的 SDK ：

```
$ xcrun --sdk iphoneos --show-sdk-path
```

输出：

```
/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS9.3.sdk
```

而我的手机是 7.0 的，所以只能去这里下载对应的 SDK。

Makefile 所有需要填写的填好后：

```
$ cd dumpdecrypted
$ make
```

在当前目录下生成 dumpdecrypted.dylib 文件。

如果你觉得很麻烦，可以直接来[这里](https://github.com/DaSens/Crack-file/tree/master/crack%20file)或者[这里](http://git.oschina.net/hongyangyi/dumpdecrypted)下载。

## 2.3 Use

将 `dumpdecrypted.dylib` 放到需要砸壳 app 的 Documents 下。

查找微信的 Documents：

```
$ find / -name WeChat.app
```

得到路径：

```
/private/var/mobile/Applications/3DE3657E-0F69-45FF-928B-3DD5CD7A59FD/WeChat.app
```

scp 传输：

```
$ scp ~/Desktop/dump/dumpdecrypted/dumpdecrypted_7.dylib root@172.16.212.217:/private/var/mobile/Applications/3DE3657E-0F69-45FF-928B-3DD5CD7A59FD/Documents
```

砸壳：

```
$ cd /private/var/mobile/Applications/3DE3657E-0F69-45FF-928B-3DD5CD7A59FD/Documents
$ DYLD_INSERT_LIBRARIES=dumpdecrypted_7.dylib /private/var/mobile/Applications/3DE3657E-0F69-45FF-928B-3DD5CD7A59FD/WeChat.app/WeChat
```

> 注意 `DYLD_INSERT_LIBRARIES=` 后填写的是你刚刚传输的 .dylib 文件名，我的是 dumpdecrypted_7.dylib

砸壳成功：

```
[+] detected 32bit ARM binary in memory.
[+] offset to cryptid found: @0xfea4c(from 0xfe000) = a4c
[+] Found encrypted data at address 00004000 of length 42237952 bytes - type 1.[+] Opening /private/var/mobile/Applications/3DE3657E-0F69-45FF-928B-3DD5CD7A59FD/WeChat.app/WeChat for reading.
[+] Reading header
[+] Detecting header type[+] Executable is a FAT image - searching for right architecture[+] Correct arch is at offset 16384 in the file[+] Opening WeChat.decrypted for writing.
[+] Copying the not encrypted start of the file[+] Dumping the decrypted data into the file[+] Copying the not encrypted remainder of the file[+] Setting the LC_ENCRYPTION_INFO->cryptid to 0 at offset 4a4c
[+] Closing original file[+] Closing dump file
```

ls 查看 Documents 中文件多了一个 `WeChat.decrypted`，这就是砸壳过后的文件，scp出来：

```
$ scp WeChat.decrypted zhoulingyu@172.16.211.181:/Users/zhoulingyu/Desktop/dump/Wechat
```

# 3 class dump 砸壳后的文件

上一步我们得到了 `WeChat.decrypted`，现在可以对其进行 dump：

```
class-dump --arch armv7 WeChat.decrypted -H -o /Users/zhoulingyu/Desktop/dump/Wechat/Wechat-decrypted-dump
```

> --arch armv7 是指定架构，dumpdecrypted 只会砸你手机处理器对应的那个壳，fat binary 的其它部分仍然是有壳的，而 class-dump 的默认目标又不是被砸壳的那个部分，如果不指定架构只能导出 CDStructures.h 一个文件
> 
现在就可以看到 dump 后是明文的了：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E6%94%BB%E9%98%B2%E2%80%94%E2%80%94%EF%BC%88%E5%9B%9B%EF%BC%89class%20dump%20%E4%BD%BF%E7%94%A8-05.png)

# 4 Learn More

当你能看到 .h 文件时候，意味着你知道了这个应用程序的各种接口，除了学习别人的优雅代码之外，显然也也可以做一些更有意思的事情，通过一些猜想和试验，我们可以去尝试做一个微信的插件。

下一次iOS攻防将会介绍动态库的注入和微信插件的制作~

如果您感兴趣~请点击下方打赏支持萌妹子的原创哟~

----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)

