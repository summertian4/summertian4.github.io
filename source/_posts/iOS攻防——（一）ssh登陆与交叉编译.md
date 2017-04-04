title: iOS攻防——（一）ssh登陆与交叉编译
date: 2016-07-11 20:49:11
tags:
  - iOS
  - iOS攻防
categories:
  - iOS
  - iOS攻防
---

# 简介
iOS攻防系列大家耳熟能详的是我们iOS女神念茜的系列文章。博主在看了之后也进行了一系列的学习和尝试。念茜的文章写的比较早，有很多文章中提到的东西已经不再适合现在使用，写的也不算详细，很多地方一笔带过，却不是那么好探索。在中间也有很多摸索的过程。
所以本系列文章算是对念茜iOS攻防系列的一个补充，中间细节的地方也会写的更加详尽。

# 你需要一部越狱手机
首先要做的事，找一部越狱后的iPhone，攻防方面的探索很多需要借助越狱手机的帮助。作为平常的消遣和研究你也应该有一部越狱手机，我的越狱手机是iPhone4，比较古老，但是研究够用了。

# 前期准备
首先是手机（前面说过了）。其次，越狱手机大家都知道一个app——Cydia，在上面可以下载所有的越狱APP，相当于越狱后的app store。对于iOS攻防，首先需要以下软件：
1. openSSH
2. LLVM+Clang

可能还需要的软件：
1. Cydia Translations
2. Cydia Substrate
3. Cydia Installer

这些软件都可以在Cydia下载到，如果你搜索不到，那么你需要添加一些源
在Cydia的软件源中点击『编辑』->『添加』，依次添加以下源
> 1. http://yuan.duowan.com - 多玩源
> 2. http://apt.thebigboss.org/repofiles/cydia/

<!--more-->

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E6%94%BB%E9%98%B2%E2%80%94%E2%80%94%EF%BC%88%E4%B8%80%EF%BC%89ssh%E7%99%BB%E9%99%86%E4%B8%8E%E4%BA%A4%E5%8F%89%E7%BC%96%E8%AF%91-01.PNG-w375)
![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E6%94%BB%E9%98%B2%E2%80%94%E2%80%94%EF%BC%88%E4%B8%80%EF%BC%89ssh%E7%99%BB%E9%99%86%E4%B8%8E%E4%BA%A4%E5%8F%89%E7%BC%96%E8%AF%91-02.PNG-w375)
![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E6%94%BB%E9%98%B2%E2%80%94%E2%80%94%EF%BC%88%E4%B8%80%EF%BC%89ssh%E7%99%BB%E9%99%86%E4%B8%8E%E4%BA%A4%E5%8F%89%E7%BC%96%E8%AF%91-03.PNG-w375)

# hacking
## 1. ssh登陆手机
ssh登陆，大家应该不算陌生，如果你第一次听说，你可以简单理解成『远程登录』，可以通过一台设备远程登陆另一台设备。

1. 保证你的Mac和iPhone在同一网段
2. 确定iPhone的IP
3. 远程登陆

在你mac的Terminal输入

```
ssh root@xxx.xxx.xxx.xxx
```

接着会提醒你是否连接，输入yes继续，输入密码，初始密码是`alpine`。
建议你将改密码改掉，因为这样很不安全，在默认密码的情况下，任何人都可以尝试登陆你的设备。
在登录之后，你可以更改你的密码：

```
passwd root
```

ssh登陆后，可以试着看看手机的目录结构
![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E6%94%BB%E9%98%B2%E2%80%94%E2%80%94%EF%BC%88%E4%B8%80%EF%BC%89ssh%E7%99%BB%E9%99%86%E4%B8%8E%E4%BA%A4%E5%8F%89%E7%BC%96%E8%AF%91-05.png)
 
## 2. 交叉编译
首先解释一下什么是交叉编译。交叉编译指在一个平台上生成另一个平台上的可执行代码。
我们将会在MAC上写代码，但要生成的可执行文件需要在iPhone上运行。
编译是由编译器完成的，所以我们首先要找到合适的编译器。

我们本次的任务就是写一个简单的C程序，能在iPhone上跑的C程序。

念茜所提到的arm-apple-darwin10-llvm-gcc-4.2我是没有找到的，因为这东西好像是在Xcode5的时候才有。

我用的是Clang，所以我们需要下载一下Clang。Clang是一个C语言、C++、Objective-C、C++语言的轻量级编译器。[这里是传送门](http://clang.llvm.org/get_started.html)

### 1. 写经典HelloWorld

```c
#include                                                                                               
int main(){   
       printf("Hello world !!!\n");   
       return 0;   
} 
```

### 2. 编译
命令台编译：

```
touch helloworld.c
open helloworld.c
...（写一下代码）
xcrun -sdk iphoneos clang -arch armv7 -o helloworld helloworld.c
```

用过自动打包ipa的同学都对xcrun和xcodebuild很熟悉。这与打包过程类似。

格式：`xcrun -sdk iphoneos clang -arch armv7 -o [目标文件名] [源文件名]`

生成可在iPhone平台运行的二进制可执行文件

### 3. 放到iPhone中
通过ssh文件传输将helloworld传到iPhone中

```
scp helloworld root@192.168.31.152:helloworld
```

再看一下iPhone文件目录，是不是已经有了helloworld
![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E6%94%BB%E9%98%B2%E2%80%94%E2%80%94%EF%BC%88%E4%B8%80%EF%BC%89ssh%E7%99%BB%E9%99%86%E4%B8%8E%E4%BA%A4%E5%8F%89%E7%BC%96%E8%AF%91-06.png)

运行一下：
![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E6%94%BB%E9%98%B2%E2%80%94%E2%80%94%EF%BC%88%E4%B8%80%EF%BC%89ssh%E7%99%BB%E9%99%86%E4%B8%8E%E4%BA%A4%E5%8F%89%E7%BC%96%E8%AF%91-07.png)

Bingo！

如果你继续玩，你会发现其实你的iPhone就是一个类Linux系统（本来就是Unix~~），你可以随便玩。

# 补充知识
## 1. 重置ssh登陆密码
如果你不幸忘记了ssh密码，可以在Cydia中下载ifile软件，通过ifile找到/private/etc/master.password文件，文件中会有以下一段：

> root:xxxxxxxxxxxxx:0:0::0:0:System
> Administrator:/var/root:/bin/sh
> mobile:xxxxxxxxxxxxx:501:501::0:0:Mobile
> User:/var/mobile:/bin/sh

将root:及mobile:后面的13个x字符处修改成`/smx7MYTQIi2M`，修改后保存此文件，你iphone的ssh密码就重新回到默认的alpine

# 下期内容
只是写个HelloWorld是不是很无聊，下一期内容将会教你如何**非法窃取iTunesstore信息**以及**Cycript修改运行时**。

----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)

