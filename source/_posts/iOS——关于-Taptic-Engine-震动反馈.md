title: iOS——关于-Taptic-Engine-震动反馈
date: 2017-01-16 10:30:17
tags:
  - iOS
categories:
  - iOS
---

# What has Happened？

上周，leader 拿着 iPhone 7 打开了网易新闻，问我：『你看，你这里的下拉刷新是`短震动`，我们的手机数周遥控电视的时候只有`长震动`，产品那边问能不能用短震动』。

然后博主就去查看了一下关于短震动的方式，整个过程可以描述为——『资料真少！』。

不过最后通过一下午的搜集，最终还是总结整理出来了这份文档，也补充了自己对 iPhone 6s 之后对 Taptic Engine 的了解。

<!-- More -->

# Taptic Engine

先了解一个概念——Taptic Engine

Taptic Engine 是苹果产品上推出的全新震动模块，该元件最早出现在 Apple Watch 中。iPhone 6s 和 iPhone 6s Plus 中，也同样内置了Taptic Engine，在设计上有所升级。

Taptic Engine 振动模块为 Apple Watch 以及 iPhone 6s、iPhone 7 提供了 Force Touch 以及 3D Touch，不同的屏幕操作，可以感受到不同的振动触觉效果，带来更好的用户体验。

# 短震方法一 AudioServicesPlaySystemSound

常用调用：

```objc
AudioServicesPlaySystemSound(kSystemSoundID_Vibrate);
```

以上代码在各个型号手机中反应为长震

API 系统版本支持：

```objc
__OSX_AVAILABLE_STARTING(__MAC_10_5,__IPHONE_2_0);
```

APPLE 公开的 `SystemSoundID` 有：

```objc
CF_ENUM(SystemSoundID)
{
    kSystemSoundID_UserPreferredAlert   = 0x00001000,
    kSystemSoundID_FlashScreen          = 0x00000FFE,
        // this has been renamed to be consistent
    kUserPreferredAlert     = kSystemSoundID_UserPreferredAlert
};

CF_ENUM(SystemSoundID)
{
    kSystemSoundID_Vibrate              = 0x00000FFF
};
```

以上类型 _**没有短震动**_ 。

但通过以下代码，可以得到更多类型的震动：

```objc
// 普通短震，3D Touch 中 Peek 震动反馈
AudioServicesPlaySystemSound(1519);
```

```objc
// 普通短震，3D Touch 中 Pop 震动反馈
AudioServicesPlaySystemSound(1520);
```

```objc
// 连续三次短震
AudioServicesPlaySystemSound(1521);
```

但以上 ID 均未在 Apple 的 Documents 中描述。显然，_**这是调用了一些私有一些属性 **_ 。

关于是否调用了私有 API，也有一些讨论，可以查看[这里](https://forums.developer.apple.com/thread/45628)。


# 短震方法二 获取 _tapticEngine

这种方法是从[这里](https://unifiedsense.com/development/using-taptic-engine-on-ios.html)搜集到的。

```objc
id tapticEngine = [[UIDevice currentDevice] performSelector: NSSelectorFromString(@"_tapticEngine")
                                                     withObject:nil];
[tapticEngine performSelector: NSSelectorFromString(@"actuateFeedback:")
                       withObject:@(0)];
```

或者：

```objc
id tapticEngine = [[UIDevice currentDevice] performSelector: NSSelectorFromString(@"_tapticEngine")
                                                     withObject:nil];

SEL selector = NSSelectorFromString(@"actuateFeedback:");
int32_t arg = 1001;
    
NSInvocation *inv = [NSInvocation invocationWithMethodSignature:[tapticEngine methodSignatureForSelector:selector]];
[inv setTarget:tapticEngine];
[inv setSelector:selector];
[inv setArgument:&arg atIndex:2];
[inv invoke];
```

显然， _**这是调用了私有 API**_ 。

这些方法，在实际测试的时候发现，在 iPhone 7 上调用没有震动反馈，在 iPhone 6S Plus 上调用有震动反馈，在 iPhone 6 上调用 无反馈。

# 短震方法三 UIImpactFeedbackGenerator

iOS10 引入了一种新的、产生触觉反馈的方式， _**帮助用户认识到不同的震动反馈有不同的含义**_ 。这个功能的核心就是由 `UIFeedbackGenerator` 提供。Apple 对于 `UIImpactFeedbackGenerator` 有一篇[介绍文档](https://developer.apple.com/reference/uikit/uifeedbackgenerator#2555399)。

UIFeedbackGenerator 可以帮助你实现 haptic feedback。它的要求是：

1. 支持 Taptic Engine 机型 (iPhone 7 以及 iPhone 7 Plus).
2. app 需要在前台运行
3. 系统 Haptics setting 需要开启

Apple 曾表示公开了 Taptic Engine 的 API，但是鲜有文档。在搜罗了各种资料后，可以认为 `UIImpactFeedbackGenerator` 即 Taptic Engine 的 公开 API。

它的调用方式是：

```objc
UIImpactFeedbackGenerator *generator = [[UIImpactFeedbackGenerator alloc] initWithStyle: UIImpactFeedbackStyleLight];
[generator prepare];
[generator impactOccurred];
```

# Others 

观察 `UIImpactFeedbackGenerator` 你会发现它继承于 `UIFeedbackGenerator`。除了 `UIImpactFeedbackGenerator` 还有三种 FeedbackGenerator：

1. UIImpactFeedbackGenerator
2. UISelectionFeedbackGenerator
3. UINotificationFeedbackGenerator

详情可参考 Apple 的 [这篇 Reference](https://developer.apple.com/reference/uikit/uifeedbackgenerator?language=objc) 。

对于震动反馈的应用，Apple 也给出了示例场景：

```objc
- (IBAction)gestureHandler:(UIPanGestureRecognizer *)sender {
    
    switch (sender.state) {
        case UIGestureRecognizerStateBegan:
            
            // Instantiate a new generator.
            self.feedbackGenerator = [[UISelectionFeedbackGenerator alloc] init];
            
            // Prepare the generator when the gesture begins.
            [self.feedbackGenerator prepare];
            
            break;
            
        case UIGestureRecognizerStateChanged:
            
            // Check to see if the selection has changed...
            if ([self myCustomHasSelectionChangedMethodWithTranslation:[sender translationInView: self.view]]) {
                
                // Trigger selection feedback.
                [self.feedbackGenerator selectionChanged];
                
                // Keep the generator in a prepared state.
                [self.feedbackGenerator prepare];
    
            }
            
            break;
            
        case UIGestureRecognizerStateCancelled:
        case UIGestureRecognizerStateEnded:
        case UIGestureRecognizerStateFailed:
            
            // Release the current generator.
            self.feedbackGenerator = nil;
            
            break;
            
        default:
            
            // Do nothing.
            break;
    }
}
```

# 三种方法在测试机上不同的反馈结果


| AudioServicesPlaySystemSound | 1519 | 1520 | 1521 |
| ---------------------------- | ---- | ---- | ---- |
| iPhone 7（iOS 10）      | peek 触感 | pop 触感 | 三次连续短振 |
| iPhone 6s Puls（iOS 9） | peek 触感 | pop 触感 | 三次连续短振 |
| iPhone 6（iOS 10）      | 无振动 | 无振动| 无振动 |

| 获取 _tapticEngine |  |
| ----------------- | ---------------- | 
| iPhone 7（iOS 10）      | 无振动 | 
| iPhone 6s Puls（iOS 9） | 长振 | 
| iPhone 6（iOS 10）      | 无振动 |

| UIImpactFeedbackGenerator | .Light | .Medium | .Heavy |
| ------------------------- | ------ | ------- | ------ |
| iPhone 7（iOS 10）      | 微弱短振 | 中等短振 | 明显短振 |
| iPhone 6s Puls（iOS 9） | 长振 | 长振 | 长振 |
| iPhone 6（iOS 10）      | 无振动 | 无振动 | 无振动 |

总结一下，希望同样的代码能在更多的机型上实现短振，建议使用 AudioServicesPlaySystemSound(1519)。不过可能会涉及到调用私有 API。安全起见，可以使用 `UIImpactFeedbackGenerator`。

# 代码

测试代码在[这里](https://github.com/summertian4/iOS-ObjectiveC/tree/master/iPhoneShakeDemo)。

----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)


