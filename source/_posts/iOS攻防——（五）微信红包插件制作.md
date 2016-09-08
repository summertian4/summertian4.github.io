title: iOS攻防——（五）微信红包插件制作
date: 2016-08-31 16:08:31
tags:
  - iOS
  - iOS攻防
categories:
  - iOS
---

# 观察
[上篇](http://zhoulingyu.com/2016/08/29/iOS%E6%94%BB%E9%98%B2%E2%80%94%E2%80%94%EF%BC%88%E5%9B%9B%EF%BC%89class-dump%20%E4%B8%8E%20Dumpdecrypted%20%E4%BD%BF%E7%94%A8/)中，我们已经成功将微信的壳砸开，并且 dump 出了微信的头文件。

首先明确本篇的目的——制作一个微信红包插件，并且能够让非越狱手机使用。

通过观察微信头文件，注意到以下两个文件

1. `CMessageMgr.h`
2. `WCRedEnvelopesLogicMgr.h`

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E6%94%BB%E9%98%B2%E2%80%94%E2%80%94%EF%BC%88%E4%BA%94%EF%BC%89%E5%BE%AE%E4%BF%A1%E7%BA%A2%E5%8C%85%E6%8F%92%E4%BB%B6%E5%88%B6%E4%BD%9C-01.png)

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E6%94%BB%E9%98%B2%E2%80%94%E2%80%94%EF%BC%88%E4%BA%94%EF%BC%89%E5%BE%AE%E4%BF%A1%E7%BA%A2%E5%8C%85%E6%8F%92%E4%BB%B6%E5%88%B6%E4%BD%9C-02.png)

观察到有这两个方法：

```objc
- (void)AsyncOnAddMsg:(id)arg1 MsgWrap:(id)arg2;
- (void)OpenRedEnvelopesRequest:(id)arg1;
```

<!--more-->

名字上去猜测第一个方法是异步异步接收消息，第二方法是请求打开红包。

简单的理一下思路：
1. hook微信的 `- (void)AsyncOnAddMsg:(id)arg1 MsgWrap:(id)arg2` 方法，滤出红包消息
2. 调用 `(void)OpenRedEnvelopesRequest:(id)arg1` 打开红包

# Do it
要做到上一节的目标，我们需要通过动态链接库实现。

在 Xcode 中新建一个 dylib 工程，我取名叫 `RedEnvelopesGenius`。

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E6%94%BB%E9%98%B2%E2%80%94%E2%80%94%EF%BC%88%E4%BA%94%EF%BC%89%E5%BE%AE%E4%BF%A1%E7%BA%A2%E5%8C%85%E6%8F%92%E4%BB%B6%E5%88%B6%E4%BD%9C-03.png)


