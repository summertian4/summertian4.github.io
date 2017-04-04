title: iOS——分享自己封装的验证码按钮控件CFSecurityCodeButton
date: 2015-11-10 15:07:27
tags:
  - iOS
categories:
  - iOS
---


# CFSecurityCodeButton
写了一个验证码按钮的控件，简约好用，希望大家喜欢。

github地址：https://github.com/summertian4/iOS-CFSecurityCodeButton

# 简介
CFSecurityCodeButton是一个简约的验证码按钮。
![CFSecurityCodeButton演示](http://7xnrog.com1.z0.glb.clouddn.com/github_iOS-CFSecurityCodeButton-show.gif)

<!--more-->

# 功能
1. 自定义Normal状态下文字和Disabled状态下文字
2. 自动根据Normal和Disabled状态下文字设置宽高
3. 自定义定时时间
4. 自动根据按钮的主题色调整文字颜色
5. 提供了代理方法监控按钮开始计时和计时结束
6. 提供了一些好看的颜色供使用者选择

# 安装
将CFSecurityCodeButton.h、CFSecurityCodeButton.m拖入你的项目中

# 使用
1. 创建
	通过主题色创建一个CFSecurityCodeButton

	```objc
	CFSecurityCodeButton *btSecurityCode_Blue = [[CFSecurityCodeButton alloc] initWithColor:CFColorDodgerBlue];
	```
	提供了一些颜色供使用者选择
	
	```
	CFColorCoral
	CFColorDodgerBlue
	CFColorDeepSkyBlue
	CFColorTurquoise
	CFColorWarmYellow
	CFColorMediumPurple
	CFColorSeaGreen
	```

2. 设置文字
	如果没有设置，默认Normal状态会显示"发送验证码"，Disabled状态会显示"再次发送(倒计时)"
	如果需要自定义可以设置`normalTitle`和`disabledTitle`属性
	
	```objc
	btSecurityCode.normalTitle = @"自定义normal状态文字内容";
	btSecurityCode.disabledTitle = @"自定义disabled状态文字内容";
	```
	![CFSecurityCodeButton演示](http://7xnrog.com1.z0.glb.clouddn.com/github_iOS-CFSecurityCodeButton-02.png)
	
3. 设置倒计时
	如果没有设置，默认倒计时为60秒
	如果需要自定义可以设置`time`属性
	
	```objc
	btSecurityCode.time = 60;
	```

4. 自动调节文字颜色
	CFSecurityCodeButton会根据自身的颜色调节文字颜色，当颜色过深时文字将会变成白色，当颜色过浅时文字颜色将会变成黑色
	![CFSecurityCodeButton演示](http://7xnrog.com1.z0.glb.clouddn.com/github_iOS-CFSecurityCodeButton-03.png)

5. 代理
	提供了两个代理方法监控按钮
	
	```objc
	/**
	 *  按钮被点击
	 *
	 *  @param securityCodeButton CFSecurityCodeButton对象
	 */
	- (void)securityCodeButtonDidClicked:(CFSecurityCodeButton *)securityCodeButton;
/**
	 *  按钮倒计时结束
	 *
	 *  @param securityCodeButton CFSecurityCodeButton对象
	 */
	- (void)securityCodeButtonTimingEnded:(CFSecurityCodeButton *)securityCodeButton;
	```
	只需要实现`CFSecurityCodeButtonDelegate`，重写代理方法


# 反馈
如果有什么修改建议，可以发送邮件到coderfish@163.com，也欢迎到[我的博客](http://zhoulingyu.com)

----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)


