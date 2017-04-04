title: 快速又深入的了解OC中的nil、Nil、NULL、NSNull
date: 2015-08-10 16:29
tags:
  - iOS
categories:
  - iOS
---

在网上看过很多介绍，但是还是会晕掉，一大堆绕口的定义，比如下面这些：
>nil：		指向oc中对象的空指针，计数器0
>Nil：		指向oc中类的空指针
>NULL：		指向其他类型的空指针，如一个c类型的内存指针
>NSNull：	在集合对象中，表示空值的对象。

这样的显然不能帮助更好的理解，那么结合源码再去理解：
> nil：		#define nil ((id)0)-------->指向oc中对象的空指针
>NULL：#define NULL ((void *)0)-------->NULL其实是C语言中的定义
>NSNull：	用在不能使用nil的场合。-------->下面会进行详细描述

## **NSNull补充**
对于NSNull，如果还是困难，那就记住这一句话：NSNull是nil的对象形式。

为什么要这样呢？举个例子，OC中的数组内可以保存所有类型对象，但是不能保存基本类型。
NSArray里不能放入int、double这样类型的，只能放入对象。那么这时候如果你想向数组存入一个空对象，存nil是不合理的，这时就可以用NSNull代替。所以说NSNull是nil的对象形式。

## **nil和NSNull除了类型完全没有区别吗**？
实际上是有的：
若obj为nil：
`［obj message］`将返回NO,而不是NSException
若obj为NSNull:
`［obj message］`将抛出异常NSException，即NSNull类似于Java的null，可以报空指针错误



----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)

