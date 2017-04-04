title: iOS攻防——（三）Cycript攻·防
date: 2016-07-19 14:09:36
tags:
  - iOS
  - iOS攻防
categories:
  - iOS
  - iOS攻防
---

# 简介
> Cycript允许开发人员探讨和修改iOS和Mac OS X上运行的应用程序。
> Cycript是一个理解Objective-C语法的javascript解释器，它能够挂钩正在运行的进程，能够在> 运行时修改应用的很多东西。
> 
> 1. 能够挂钩正在运行的进程，并且找出正被使用的类信息，例如view controllers，内部和第三方库，甚至程序的delegate的名称。
> 2. 对于一个特定的类，例如View Controller, App delegate或者任何其他的类，我们能够得到所有被使用的方法名称。
> 3. 能够得到所有实例变量的名称和在程序运行的任意时刻实例变量的值。
> 4. 能够在运行时修改实例变量的值。
> 5. 能够执行Method Swizzling，例如替换一个特定方法的实现。
> 6. 可以在运行时调用任意方法，即使这个方法目前并不在应用的实际代码当中。

<!--more-->

# Cycript安装
在[这里下载](http://www.cycript.org)
在[这里阅读]()所有Cycript诡计

# 攻
先看看怎么用Cycript干点坏事吧

## 1. 给应用弹一个莫名其妙的alert

### ssh登陆你的手机（如果不会，上一篇有~）
### 找一个app
这里我找的是以前做的一个app

```
ps aux | grep blackwidow
```
	
print

```
mobile 466 6.6 7.0 508416 36204 ?? Ss 11:22AM 0:09.65 /xxxx/blackwidow
```
	
这样知道进程号是466
	
### hock住

```
cycript -p 466
```
	
如果你看到出现了cy#，说明你可以开始编写Cycript代码了
	
### alert

```objc
// 找到widnow
var window = [UIApplication sharedApplication].keyWindow;
// 初始化一个alert
var alert = [[UIAlertView alloc] initWithTitle:@"hack you" message:@"hack you" window cancelButtonTitle:@"cancel" otherButtonTitles:@"yes", nil];
// 弹出来吧
[alert show];
```
	
![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E6%94%BB%E9%98%B2%E2%80%94%E2%80%94%EF%BC%88%E4%B8%89%EF%BC%89Cycript%E6%94%BB%C2%B7%E9%98%B2-01.PNG-w375)
	
	
## 2. 探索一个app

```javascript
function printMethods(className) {
  var count = new new Type("I");
  var methods = class_copyMethodList(objc_getClass(className), count);
  var methodsArray = [];
  for(var i = 0; i < *count; i++) {
    var method = methods[i];
    methodsArray.push({selector:method_getName(method), implementation:method_getImplementation(method)});
  }
  free(methods);
  return methodsArray;
}
```

调用一下：
```javascript
printMethods(AppDelegate)
```

输出结果：
![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E6%94%BB%E9%98%B2%E2%80%94%E2%80%94%EF%BC%88%E4%B8%89%EF%BC%89Cycript%E6%94%BB%C2%B7%E9%98%B2-02.png)

是不是觉得发生了很可怕的事情？该有的都被打印出来了。

你还可以通过试探的方式找出每一个Controller的名字，例如：

insert

```javascript
var homeVC= [[[[[UIApplication sharedApplication] keyWindow] subviews] objectAtIndex:0] nextResponder];
```

print

```
#"<HomePageTabBarViewController: 0x156cf200>"
```

insert

```javascript
var page0VC = [homeVC.childViewControllers objectAtIndex:3]
```

print

```
#"<BaseNavigationController: 0x156decc0>"
```

insert
```
var meVC = page0VC.topViewController
```

print
```
#"<MeViewController: 0x156de940>"
```

这样，我们就找到了『我的』页面所属Controller。

查看所有的方法：

```javascript
printMethods(MeViewController)
```

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E6%94%BB%E9%98%B2%E2%80%94%E2%80%94%EF%BC%88%E4%B8%89%EF%BC%89Cycript%E6%94%BB%C2%B7%E9%98%B2-02.png)

改个标题试试：

```objc
[meVC setCurrentTitle:@"hack you"];
```

效果如下：
![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E6%94%BB%E9%98%B2%E2%80%94%E2%80%94%EF%BC%88%E4%B8%89%EF%BC%89Cycript%E6%94%BB%C2%B7%E9%98%B2-04.PNG-w375)

试想一下，如果MeViewController中或者LoginViewController中有一个方法叫getUserInfo，那么通过Cycript就可以轻而易举的拿到用户信息。

不过Cycript在这里最主要的作用还是偷窥APP和调试APP。
当然，好玩的方法还有很多。


# 防
知道了Cycript的可怕，在有重要信息藏在代码中的时候，我们也得学会如何放置Cycript修改运行时。

你可以参考[这篇文章](http://www.cocoachina.com/ios/20150511/11801.html)


----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)


