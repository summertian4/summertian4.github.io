---
title: iOS——Swift 中的 strongSelf，你以为不需要了？
date: 2017-08-14 15:19:35
tags:
	- iOS
	- Swift
	- 内存与引用
categories:
	- iOS
	- Swift
---

# 开端

Objective-C 中，有一段重复写到你不得不加入 `snippets` 的代码块。就是下面这段

```objc
__weak __typeof__(self) weakSelf = self;
[self.aButton touchUpInside:^{
    __strong __typeof(weakSelf)strongSelf = weakSelf;
    if (strongSelf) {
        strongSelf.title = @"按钮被点击";
    }
}];
```

一般，由于重复写的次数过多，就加到了 `snippets` 快捷代码块，以下是我的：

<!-- More -->

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_swift-strongself-01.png)

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_swift-strongself-02.png)

当然，如果用了 RAC，你可以写成如下代码：

```objc
@weakify(self);
[self.aButton touchUpInside:^{
	 @strongify(self);
    if (strongSelf) {
        self = @"按钮被点击";
    }
}];
```

当然，如果你阅读过 RAC 对 `@weakify` 和 `@strongify` 语法糖的实现，你会知道，实现方式不过是用宏简写了 `__weak __typeof__(self) weakSelf = self;` 和 `__strong __typeof(weakSelf)strongSelf = weakSelf;`

重点是为什么要写这两句代码？

weakSelf 是为了避免循环引用（如果你对循环引用和内存管理不了解的话，请调跳到下方 [Other 部分](#Other)）。

strongSelf 是为了防止 self 被提前释放。具体可以参考 [深入研究Block用weakSelf、strongSelf、@weakify、@strongify解决循环引用](http://www.jianshu.com/p/701da54bd78c)。

# 你以为 Swift 中不需要 strongSelf 了吗？

Swift 中有了 `[weak self]` 语法糖后，再也不用谢一长串 `__weak __typeof__(self) weakSelf = self;`，相应的，在闭包中会调用到已经被弱引用的 `self`，但使用 `self` 时需要加上 `?`。

这里就已经可以初见端倪了，和 Objective-C 一样，weak 引用的 `self` 是可以被释放的，也就是说 `self` 可以为 `nil`。在 Objective-C 中，没有可选性概念，所以对此并不感知，在 Swift 中问题就很容易暴露出来。如果 `self` 被提前释放了会如何？

我们做如下代码：

```swift
class ViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()

        var anObject: SomeClass? = SomeClass()  // Optional, so we can set it to nil
        anObject?.doClosure()

        DispatchQueue.global(qos: .userInitiated).async {
            usleep(100)
            anObject = nil  // This will dealloc c
        }
    }
}

class SomeClass {
    deinit { print("Destroying C") }
    func log(_ msg: String) { print(msg) }
    func doClosure() {
        DispatchQueue.global(qos: .userInitiated).async { [weak self] in
            self?.log("before sleep")
            usleep(500)
            self?.log("after sleep")
        }
    }
}
```

这段代码运行结果如下：

这就是问题所在了。代码中可以清晰的看到，在闭包中代码运行的过程中，`anObject` 被设置成了 `nil`，此时没有任何对象强引用它，自然被释放了，释放后闭包中代码继续运行，`self` 此时为 `nil`，但由于 Swfit 的安全性，`self?.log("after sleep")` 没有解包成功便不会执行，所以不会造成闪退。但问题依然有。

由于后续代码没有执行，造成的其他逻辑错误是可想而知的，如果闭包中正在执行存磁盘操作，`self` 被释放，后续还有更新 UI 显示等等操作，便不会执行，造成各种各样的问题。

所以，你以为 Swift 中就不需要 `strongSelf` 了？不，以后你的代码仍然需要：

```swift
DispatchQueue.global(qos: .userInitiated).async { [weak self] in
    if let strongSelf = self {
        strongSelf.log("before sleep")
        usleep(500)
        strongSelf.log("after sleep")
    }
}
```

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_swift-strongself-05.png-w375)

当然，你可以加到 `snippets`，这样就可以快速插入代码了：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_swift-strongself-04.png)

# 总结

**Swift 和 Objective-C 大不同，但是 iOS 内存管理仍然是 iOS 内存管理，ARC 仍然是 ARC，所以不管语法怎么变，关于内存和引用依旧还是原来的样子，`strongSelf` 也好，循环引用也好，千万别忘了去注意。**

# Other
 
如果以下这幅图中的三个问题，你不能清晰的答出答案和其理由，建议你去读一读 Objective-C 关于内存管理的源码哟。这里是我的两篇博文，希望能帮到你：
 
1. [iOS进阶——iOS（Objective-C） 内存管理&Block](http://zhoulingyu.com/2017/02/08/iOS%E8%BF%9B%E9%98%B6%E2%80%94%E2%80%94iOS-Memory-Block/)
2. [iOS进阶——iOS（Objective-C）内存管理·二](http://zhoulingyu.com/2017/02/15/Advanced-iOS-Study-objc-Memory-2/)

以下是三个问题：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_swift-strongself-03.jpeg)

----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼](http://weibo.com/coderfish/)

