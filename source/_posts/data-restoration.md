---
title: 记一次删库到数据恢复
date: 2018-07-09 15:17:11
tags:
	- 后端
	- linux
	- 数据库
categories:
	- Other
---

# 记一次删库到数据恢复

熟悉的朋友可能知道，进入架构组后，今年一直在为团队做各种开发辅助工具，其中包括一个服务器。

最近这个服务器也上线有三月多了，也不断收集了很多数据，包括用户的各种行为操作、API 调用数据等等。

在最近的一次升级中，我调整了源码的目录结构，脑子一抽删除了数据库。

<!-- More -->

于是就开始为期一天的数据恢复过程，最后的结果是很悲剧，好在影响不大，可以放到后面讲。


## 文件找回

需要说明的是，因为是小微服务，采用的是最简单的 SQLite 数据库。SQLite 区分其他的数据库的明显地方是，它只提供最基本的数据服务，但并不启用端口监听，可以简单的认为这就是个文件，如果删除了，就和普通文件从系统中消失是一样的。所以通常如果使用 SQLite 保存数据需要自行定期备份。

所以这次的博文虽然叫『删库』，但实际上可以简单的理解成**文件找回**。

## rm -rf

我在误删时，是使用 rm -rf 命令，这当然是个耳熟能详的恶名远扬的命令（以至于我当时请教后端大佬的时候，后端大佬的反应是『哦，你也终于删一次库啦』）

rm 是 linux 系统的用于『删除文件或目录』的命令

`-f` 标识表示 force，即：

> 在除去有写保护的文件前不提示。

`-r` 标识表示：

> 当 File 参数为目录时允许循环的删除目录及其内容

**FYI**: [更多参考](https://www.ibm.com/support/knowledgecenter/zh/ssw_aix_71/com.ibm.aix.cmds4/rm.htm)

## extundelete 与磁盘存储

[extundelete](http://extundelete.sourceforge.net/) 是 linux 系统下一个有力的数据恢复工具。extundelete使用存储在分区日志中的信息来尝试恢复已从分区中删除的文件。


我们稍微复习一下计算机基础知识：

### 硬盘

硬盘是一种采用磁介质的数据存储设备，数据『物理意义上的』存储在若干个磁盘片上。在磁盘片的每一面上，以转动轴为轴心、以一定的磁密度为间隔的若干个同心圆就被划分成磁道（track），每个磁道又被划分为若干个扇区（sector）。

### 主引导扇区和分区表

硬盘的0磁道0柱面1扇区是主引导扇区位，包括硬盘主引导记录MBR（Main Boot Record）和**分区表**DPT（Disk Partition Table）。**操作系统通过分区表把硬盘划分为若干个分区，然后再在每个分区里面创建文件系统，写入数据文件。**

### 分区日志

分区日志系统是一个文件系统，用于修复由于计算机关闭不当而导致的任何不一致。这种关闭通常是由于电源中断或软件问题造成的。即记录了分区的数据读写操作。

通过这些日志，可以知道分区中的历史操作。

### 数据存储

具体的数据存储原理内容比较多，这里不做赘述。为了便于理解，我在此理解为：

> 操作系统中的文件在硬盘的表现形式是在硬盘一片数据区域记录二进制信息，并由操作系统的一个指针指向该物理地址
> 而操作系统级别的『删除文件』，即删除这个『指针』
> 原来的物理地址内没有『指针』指向后，相当于被释放，当操作系统需要时，可以被复写上新的数据。

由此观得，extundelete 通过查阅分区日志，找到被删除的指针，告诉用户，可以尝试恢复哪些数据。

但由于操作系统随时可以复写空余磁盘，所以如果要恢复的物理地址已经被改写，数据将无法找回。

### DEMO

为了脱敏，我使用我自己的服务器，具体记录一下文件恢复的过程。

#### 环境

- CentOS 7.4 64位
- 文件目录: /root/Sparrow/db.sqlite3

#### 删除文件

```shell
# rm -rf db.sqlite3
```

#### 安装 extundelete

```shell
# yum install extundelete -y
```

#### 挂载磁盘

首先我们去查询这个文件或上级文件夹所处的磁盘

```shell
# df /root/Sparrow/
文件系统          1K-块    已用     可用 已用% 挂载点
/dev/vda1      41151808 2614640 36423736    7% /
```

找到后首要的是挂载磁盘，**挂载后磁盘将不会被继续写入，保护现场**

```shell
umount /dev/vda1
```

`/dev/vda1` 就是文件所在的磁盘。

不挂载也是可以的，这样会导致磁盘可能会被其他的进程写入数据，从而抹掉原来的数据，所以我最后没有找回文件就是因为没有及时挂载。

#### inode

inode 是 linux 系统下文件或者文件夹的标识。通过 `ls –id` 就可以看到。

读取根目录的 inode 值：

```shell
# ls -id /
2 /
```

现在我们知道磁盘根目录的 inode 是 2。

#### extundelete 查询可恢复的数据信息

先查询磁盘根目录下的可恢复信息

```shell
# extundelete /dev/vda1 --inode 2
```

执行时，如果没有挂载磁盘，会提示。

得到结果：

![](https://raw.githubusercontent.com/summertian4/Images/master/blog/blog_data-restoration-01.png)

重点盘红圈内，看到 `root` 的 inode 是 131073，继续查询 `/dev` 的信息：

```shell
# extundelete /dev/vda1 --inode 131073
```

![](https://raw.githubusercontent.com/summertian4/Images/master/blog/blog_data-restoration-02.png)

看到 `/Sparrow`  是 262194，继续查询 `/dev` 的信息：

```shell
# extundelete /dev/vda1 --inode 262194
```

最终找到了删除信息：

![](https://raw.githubusercontent.com/summertian4/Images/master/blog/blog_data-restoration-03.png)

可以看到 db.sqlite3 被标识为 `Deleted`。

#### extundelete 恢复数据

执行 --restore-directory 恢复指定目录

```shell
# extundelete /dev/vda1 --restore-directory /root/Sparrow/db.sqlite3
```

或执行 --restore-all 恢复所有可恢复的数据

```shell
# extundelete /dev/vda1 --restore-all
```

执行结束后，会在当前路径下，生成一个 `RECOVERED_FILES` 文件夹，里面有所有被恢复的文件。

如果磁盘已经被读写，无法恢复，会提示类似信息：

```shell
Loading filesystem metadata ... 320 groups loaded.
Loading journal descriptors ... 27896 descriptors loaded.
Searching for recoverable inodes in directory /root/Sparrow/db.sqlite3 ...
120 recoverable inodes found.
Looking through the directory structure for deleted files ...
120 recoverable inodes still lost.
No files were undeleted.
```

## 后记

这一次的『删库』事件，给我了一个很好的教训。服务器的数据一定要及时备份或做好容灾，防止丢失。同时，这次也借此机会进行学习了文件恢复。

当然，希望大家都不会出现我这样的悲剧 😂。

----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[Lotty小鱼](http://weibo.com/coderfish/)


