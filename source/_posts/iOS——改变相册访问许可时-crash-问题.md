title: iOS——改变相册访问许可时的 crash
date: 2016-11-17 14:39:25
tags:
	- iOS
categories:
	- iOS
---

# 问题描述

这几天有注意到一个问题。我在做相册一块的时候，如果用户没有打开相册访问权限，会跳转到系统的设置界面，接着如果改动了权限回到 app，就会发现 app crash 了，并且重新加载了。

大概的步骤如下：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E2%80%94%E2%80%94%E6%94%B9%E5%8F%98%E7%9B%B8%E5%86%8C%E8%AE%BF%E9%97%AE%E8%AE%B8%E5%8F%AF%E6%97%B6%20crash%20%E9%97%AE%E9%A2%98-01.PNG-w375)

点击设置后代码如下：

<!-- More -->

```objc
[[UIApplication sharedApplication] openURL:[NSURL URLWithString:UIApplicationOpenSettingsURLString]];
```

成功跳转后：
![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E2%80%94%E2%80%94%E6%94%B9%E5%8F%98%E7%9B%B8%E5%86%8C%E8%AE%BF%E9%97%AE%E8%AE%B8%E5%8F%AF%E6%97%B6%20crash%20%E9%97%AE%E9%A2%98-02.png-w375)

改变一下照片权限。

然后华丽丽的 crash 了：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E2%80%94%E2%80%94%E6%94%B9%E5%8F%98%E7%9B%B8%E5%86%8C%E8%AE%BF%E9%97%AE%E8%AE%B8%E5%8F%AF%E6%97%B6%20crash%20%E9%97%AE%E9%A2%98-03.png)

没有任何输出，没有被 All Exceptions 断点拦截到。这真是一个悲伤的故事。

# 问题解决

当我发现这个问题时，仔细观察发现这种 crash 和一般的 crash 不太一样，app 会自动重启，但是没有经过 LauchScreen 界面。

然后尝试去用『大众点评』、『支付宝』一类常用的 app 做了同样的尝试。发现均有此问题。

又经过一番查找，在 **stackoverflow** 上找到 [这样](http://stackoverflow.com/questions/25611537/how-to-detect-changes-to-phauthorizationstatus) 一个问题，该问下有这样的一个回答。

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E2%80%94%E2%80%94%E6%94%B9%E5%8F%98%E7%9B%B8%E5%86%8C%E8%AE%BF%E9%97%AE%E8%AE%B8%E5%8F%AF%E6%97%B6%20crash%20%E9%97%AE%E9%A2%98-04.png)

该问题无人解答，这真是一个悲伤的故事。

随后又发现 [这样](http://stackoverflow.com/questions/26115265/app-crashes-on-enabling-camera-access-from-settings-ios-8/) 一个问题——**App crashes on enabling Camera Access from Settings iOS 8**。

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E2%80%94%E2%80%94%E6%94%B9%E5%8F%98%E7%9B%B8%E5%86%8C%E8%AE%BF%E9%97%AE%E8%AE%B8%E5%8F%AF%E6%97%B6%20crash%20%E9%97%AE%E9%A2%98-05.png)

当首次请求访问相册时，系统会自动提示你在 plist 文件中配置的请求许可信息。
无论用户是否允许你的 app 访问相册，如果用户跳出应用改变了通讯簿、日历、提醒、相册的许可开关。iOS 将会 `SIGKILL（无条件终止）` 你的 app，以便确保你的 app 不再拿到任何过时的授权信息。当用户回到你的 app 时，你的 app 将重新加载。

综上所述，这是一个可以放任它不用管的问题。这果然是个悲伤的故事。

----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)

