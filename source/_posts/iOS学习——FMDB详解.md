title: iOS学习——FMDB详解
date: 2015-10-26 18:54:10
tags:
  - iOS
categories:
  - iOS
---

# 介绍
FMDB现在越来越多的被使用了，作为轻量级数据库框架，FMDB对sqlite做了简单的包装，使得使用更简单。

但是小鱼认为FMDB还是不好用，毕竟不能做到像hebernate一样传一个对象直接保存，直接省去sql语句。之后小鱼打算自己动封装一个框架，至少能够对简单的entity进行一键增删改查。

----

# GitHub
FMDB链接：https://github.com/ccgus/fmdb

<!--more-->

----

# 安装
## 1. 使用CocoaPods安装
pods安装当然是最简单的，在Podfile文件中加入

```
pod 'FMDB'
```
然后终端进入项目根目录，输入

```
pod install
```
等待安装完就好了

如果不会使用CocoaPods，可以参考我的这篇博文：[iOS学习——CocoaPods的使用](http://zhoulingyu.com/2015/10/27/iOS学习——CocoaPods的使用/)

## 2. 传统拖文件安装
在github上下载FMDB，把这些文件直接拖入你的项目就可以用了

![FMDB所需要的文件](http://img.blog.csdn.net/20151027094718635)

----

# 使用
首先要在用到的地方

```objc
#import "FMDB.h"
```

## 创建数据库

如果该位置没有此数据库文件将会建立一个新的数据库文件，如果已经有了，将会直接获得该数据库。

```objc
FMDatabase *db = [FMDatabase databaseWithPath:dataBasePath];
```

dataBasePath可以这样设定：

```objc
#define dataBasePath [[(NSSearchPathForDirectoriesInDomains(NSDocumentDirectory,NSUserDomainMask,YES)) lastObject]stringByAppendingPathComponent:dataBaseName]
#define dataBaseName @"test.sqlite"
```

## 打开数据库

返回BOOL类型，代表是否成功

```objc
[db open]
```

## 关闭数据库

返回BOOL类型，代表是否成功

```objc
[db close]
```

## 建表

```objc
if ([self.db open]) {
	NSString *sqlCreateTable = @"CREATE TABLE IF NOT EXISTS Question (UUID TEXT PRIMARY KEY, type TEXT, bigQuetionIndex INTEGER, smallQuetionIndex INTEGER, state TEXT, time INTEGER, elapsedTime INTEGER, studentAnswer TEXT, answer TEXT, studentGrade INTEGER, grade INTEGER, content TEXT, examPaperUUID TEXT)";
	      
	BOOL res = [self.db executeStatements:sqlCreateTable];
	if (!res) {
		CFLog(@"error when creating db table");
	} else {
		CFLog(@"success to creating db table");
	}
	[self.db close];
}
```

## 增加数据

```objc
if ([self.db open]) {
	 NSString *sql = @"INSERT INTO t_entity (UUID, title) VALUES (?, ?)";
        
        BOOL res = [self.db executeUpdate:sql, entity.UUID, entity.title];
        
        if (!res) {
            CFLog(@"error when insert db table");
        } else {
            CFLog(@"success to insert db table");
	}
	[self.db close];
}
```

## 删除数据

```objc
if ([self.db open]) {
        NSString *sql = @"DELETE FROM t_entity WHERE UUID = ?";
        BOOL res = [self.db executeUpdate:sql, UUID];
        
        if (!res) {
            CFLog(@"error when DELETE db table");
        } else {
            CFLog(@"success to DELETE db table");
        }
        [self.db close];
}
```

## 更新数据

```objc
if ([self.db open]) {
        NSString *sql = @"UPDATE t_entity SET title = ?";
        BOOL res = [self.db executeUpdate:sql,
                    examPaper.title,
                    examPaper.UUID];
        if (!res) {
            CFLog(@"error when UPDATE db table");
        } else {
            CFLog(@"success to UPDATE db table");
        }
        [self.db close];
}
```

## 查找数据

```objc
Entity *entity;
if ([self.db open]) {
	//返回数据库中第一条满足条件的结果
	NSString *sql = @"SELECT * FROM t_entity WHERE UUID = ?";
        
	//返回全部查询结果
	FMResultSet *rs=[self.db executeQuery:sql, UUID, nil];
        
	while ([rs next]){
		entity = [[Entity alloc] init];
		entity.UUID = [rs stringForColumn:@"UUID"];
		entity.type = [rs stringForColumn:@"type"];
	}
        
	[rs close];
	[self.db close];
}
```

## 重要提示：使用占位符『?』
我在网上看到很多人是这样用的

`错误代码：`

```objc
NSString *sql = [NSString stringWithFormat: @"INSERT INTO '%@' ('%@', '%@', '%@') VALUES ('%@', '%@', '%@')", TABLENAME, NAME, AGE, ADDRESS, @"value1", @"value2", @"value3"];  
BOOL res = [db sql]; 
```

这是 `错误` 的，在官方文档上已经指出：
> you SHOULD NOT do this (or anything like this):
> 
> ```
> [db executeUpdate:[NSString stringWithFormat:@"INSERT INTO myTable VALUES (%@)", @"this has \" lots of ' bizarre \" quotes '"]];
> ```
> 
> Instead, you SHOULD do:
> 
> ```
> [db executeUpdate:@"INSERT INTO myTable VALUES (?)", @"this has \" lots of ' bizarre \" quotes '"];
> ```
> 

也就是说不应该使用拼串的方式，而应该使用`占位符`

## 重要提示：使用NSNumber保存

官方文档指出，用int保存是 `不起作用` 的：

```objc
[db executeUpdate:@"INSERT INTO myTable VALUES (?)", 42];
```

`正确` 的做法是这样的：

```objc
[db executeUpdate:@"INSERT INTO myTable VALUES (?)", > [NSNumber numberWithInt:42]];
```

----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)

