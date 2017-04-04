title: iOS学习——StatusBar状态栏的简单设置
date: 2015-08-19 16:29
tags:
  - iOS
categories:
  - iOS
---

## 引入
默认情况下，状态栏的文字、图标颜色是黑色
![这里写图片描述](http://img.blog.csdn.net/20150819161929310)
但有的时候，比如我们的应用程序背景是深色的，就会看不到状态栏，这时候，我们希望状态栏文字、图标变成白色。
或者，你干脆不让状态栏显示出来。

## 分析
状态的样式，是交由当前的控制器（viewController、tableViewController等），所以更改状态栏样式需要在控制器的代码里设定。
## 实现
**改变状态栏的颜色**：重写`- (UIStatusBarStyle)preferredStatusBarStyle`方法：

```objc
/**
 *  改变状态栏的文字颜色
 *
 *  @return 状态栏风格
 */
- (UIStatusBarStyle)preferredStatusBarStyle {
    return UIStatusBarStyleLightContent;
}
```

**隐藏状态栏**：重写- (BOOL)prefersStatusBarHidden方法：

```objc
/**
 *  隐藏状态栏
 *
 *  @return
 */
- (BOOL)prefersStatusBarHidden {
    return YES;
}
```

<!--more-->

----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)


