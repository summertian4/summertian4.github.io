---
title: iOS 灵活的 模块化/组件化 工具与规范 Lotusoot 解说
date: 2017-11-29 10:40:41
tags:
	- iOS模块化
	- iOS
categories:
	- iOS
	- iOS模块化
---


# 开篇

上一篇[《iOS 混编 模块化/组件化 经验指北》](http://zhoulingyu.com/2017/11/24/iOS-Modularization/)中介绍到的 [Lotusoot](https://github.com/Vegetarians/Lotusoot) ，将在本篇中做一个更为详细的介绍。

最初 [Lotusoot](https://github.com/Vegetarians/Lotusoot) 简称为『混编路由』，但是随后反而曲解了它的功能，其真正的定位是『模块化工具和规范』。

 [Lotusoot](https://github.com/Vegetarians/Lotusoot) 可以做到：
 
 1. 模块间、模块内服务调用
 2. Swift、OC、或者两者混编项目均可使用
 3. 短链注册、路由调用
 4. 脚本自动注册服务/路由表

> 注：这里的模块化，也就是大家说的『组件化』，不是在主工程用文件夹分模块，而是指将独立模块抽调成 CocoaPods 库、或者其他形式的库文件，成为一个独立工程。
> 下文中的模块就代表一个 CocoaPods 库

<!-- More -->

# 模块化解耦——路由 or 服务调用？

关于模块化，大多数人的第一反应是制作路由、注册短链、调用短链，通过这样的方式来去耦，来实现模块间的页面跳转、服务调用。类似 [MGJRouter](https://github.com/meili/MGJRouter) 一类的库就是基于这样的思想。

但我也非常认可 [casa](https://casatwy.com/) 在反驳使用 URL 作为模块化核心的理由。即：『短链的实质还是通过 URL 来调用服务或者打开页面，反而不如字符串直接，反而增加了 URL 本身的维护成本』。

所以，我们应当回归模块化的本质。我们模块化的最初的目的往往是为了：

1. 代码拆分，将**关联性强的基础服务代码**或者**业务代码**抽调在一起，单独封版，独立开发
2. 防止主工程越来越大，变得臃肿

相对应的，模块化需要的功能是：

1. 提供多个库之间的服务调用
2. 保持库与库之间的独立、非强依赖

所以，总的来说，模块化的重点还是如何**去除多个模块之间的耦合，让每个模块在不强依赖的情况下可以调用其他模块的服务**。

**URL 短链、甚至是路由、都不是模块化重点之处。**路由只要你想，都可以通过服务注册实现。

> 注：『不强依赖』指的是，模块A 调用 模块B 不需要在 Pod 依赖中写出对 B 的依赖，或者简单的认为，模块A 的代码中不出现 `import B`。

# 公共模块和依赖关系

Lotusoot 是这样解耦的：

## 1. Lotus

创建一个 PublicModule，其中存放各个模块的 Lotus，Lotus 其实就是协议，定义了每个模块可以提供的服务（即可以调用的方法），举例如下：

```swift
public protocol AccountLotus {
    func login(username: String, password: String, complete: (Error?) -> Void)
    func register(username: String, password: String, complete: (Error?) -> Void)
    func email(username: String) -> String
    func showLoginVC(username: String, password: String)
}
```

## 2. Lotusoot

在各个模块中，实现 PublicModule 中对应的 Lotus，即具体的服务类，称为 Lotusoot。
Lotusoot 中具体实现了服务的逻辑，并在 **注解** 中表明了模块的 `命名空间-@NameSpace`、`Lotusoot-@Lotusoot`、`Lotus-@Lotus`。举例如下：

```swift
// @NameSpace(TestAccountModule)
// @Lotusoot(AccountLotusoot)
// @Lotus(AccountLotus)
class AccountLotusoot: NSObject, AccountLotus {
    
    func email(username: String) -> String {
        return OtherService.email(username: "zhoulingyu")
    }

    func login(username: String, password: String, complete: (Error?) -> Void) {
        LoginService.login(username: username, password: password, complete: complete)
    }
    
    func register(username: String, password: String, complete: (Error?) -> Void) {
        RegisterService.register(username: username, password: password, complete: complete)
    }
    
    func showLoginVC(username: String, password: String) {
        // 可以用你们喜欢的非耦合方式处理跳转
        // 或者传入 rootvc
        // 更好的方式是自己的非耦合 UI 跳转处理模块
        print("show login view controller")
    }
}
```

注解是非必须的，注解是为了 `Lotusoot.py` 可以扫描 Lotusoot 自动注册，后面一节将会说到。如果你不想使用自动注册，也可以选择手动注册。

> 注：这里做一点解释，『协议-服务类』即『Lotus-Lotusoot』的命名由来纯属卖个萌，因为，协议是暴露外部的，所以叫莲花，而具体实现的服务类自然就是莲藕（Loutsoot）了。

## 3. 自动注册 or 手动注册

如果使用了注解，可以自动注册所有服务，只需要：

```swift
func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplicationLaunchOptionsKey: Any]?) -> Bool {
    LotusootCoordinator.registerAll()
}
```

如果先手动注册，如下所示：

```objc
[LotusootCoordinator registerWithLotusoot:[AppDelegateLotusoot new] lotusName:@"AppDelegateLotus"];
    [LotusootCoordinator registerWithLotusoot:[MainLotusoot new] lotusName:@"MainLotus"];
```

不过手动注册就是去了使用 Lotusoot 的意义了，所以在无法满足条件时再使用手动注册（比如目前 0.0.2 版本的 Lotusoot，主工程如果有多 Target，是无法动态获取 Target 名，导致无法正确获取命名空间，反射到类，当然除工程，各个模块由于使用 CocoaPos 不存在这个问题。下一个版本 Lotusoot 将会重点解决这个问题）

## 4. 关系图

通过 Lotusoot 搭建的工程如下图所示：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS-modularized-tool-Lotusoot-003.png)

所有模块只需要依赖 PublicModule，通过 PublicMoudle 下的 Lotusoot 即可调用其他模块的服务。实例代码如下：


```swift
let loginLotus = s(LoginLotus.self) 
let loginModule: LoginLotus = LotusootCoordinator.lotusoot(lotus: loginLotus) as! LoginLotus
loginModule.login(username: "test", password: "test", complete:nil)
```

OC 中使用：

```objc
id<LoginLotus> loginModule = [LotusootCoordinator lotusootWithLotus:@"LoginLotus"];
[loginModule login:@"test" password:@"test" complete:nil];
```

# Lotusoot 是如何实现自动注册服务的？在什么时机？

这个问题也是在编写 Lotusoot 最初重点考虑的问题。原因是 Swift 没有 `+(void)load`。

大概是每个用 Swift 开发模块路由和解耦工具的人都纠结过的问题。


## 1. OC 中常见的解决方案

先说说通常 OC 的路由或者解耦，都是在 `+(void)load` 注册类服务的，大致的做法类似于：

```objc
+ (void)load {
    @autoreleasepool {
        [[Router shared] map:@"LoginViewController" toController:[self class]];
    }
}
```

或者

```objc
+ (void)load {
    @autoreleasepool {
        [[ServiceManager shared] register:@"LoginService" toService:[self class]];
    }
}
```

这样，即使是在每个模块内，也可以正常注册自己的服务。而主工程和其他模块调用只需要通过字符串调用即可。

## 2. Swift 中的痛点

由于 Swift 是没有 `+(void)load` 的，也没有其他可靠的方法可以替代，那么势必需要在主工程中加入注册路由这一步骤，通常可以放在 `didFinishLaunchingWithOptions`，因为主模块可以调到所有模块的类。那么随之而来的问题就是，你可能会出现这样常常的代码：

```swift
ServiceManager().register("LoginService", toService:LoginService.self)
ServiceManager().register("UserCenterService", toService:UserCenterService.self)
ServiceManager().register("HistoryService", toService:HistoryService.self)
...
```

可能会有一张长长的列表，而且由于服务都分散在各个模块，但是却集中在主模块，往往难以看到路由表和服务类的关联，表征不够明显、关系不够强烈。

## 3. Lotusoot 的解决方案

有什么更好的办法注册？[mmoaay](http://www.jianshu.com/u/2d46948e84e3) 给了我一个灰常棒的建议，参考 [R.swift](https://github.com/mac-cain13/R.swift) 的做法，通过脚本，来完成自动注册。

> [R.swift](https://github.com/mac-cain13/R.swift) 的提供的功能是，可以让使用的 iOSer 可以像开发 Android 的一样调用图片、字符串、音频等等资源文件。在 Project 中插入 Run Script，这个脚本可以在编译阶段扫描整个工程，列算所有的资源文件，最后生成一个 `R.generated.swift` 文件，就像这个样子：
> 
> ![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS-modularized-tool-Lotusoot-001.png)
> 
> 使用的时候就可以：
> 
> ![](https://github.com/mac-cain13/R.swift/raw/master/Documentation/Images/DemoUseImage.gif)


**同样 Lotusoot 通过一个 python 脚本，在『Compile Source』之前扫描工程目录下的文件，找出 Lotusoot 和 Lotus 对应关系，并生成一个 `Lotusoot.plist` 文件**：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS-modularized-tool-Lotusoot-004.png)

如何识别 Lotus？

目前，Lotusoot 使用了很 Low 的方式，在注解中表明了模块的 `命名空间-@NameSpace`、`Lotusoot-@Lotusoot`、`Lotus-@Lotus`，脚本就可以识别。举例如下：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS-modularized-tool-Lotusoot-002.png)

所以，`didFinishLaunchingWithOptions` 只需要一句，即可自动注册路由。

```swift
LotusootCoordinator.registerAll()
```

为什么使用脚本解决？是因为到目前为止，所有的解耦工具或者路由都是通过程序员手动去写代码添加路由，不管是在 `+ (void)load` 中注册也好，在程序启动后注册也好，都是有程序员手动管理的。使用脚本是希望在编译阶段前，就准备好所有的『协议-服务类』对应关系表，在程序启动后通过这张表自动注册，**实现程序员不手动注册、完全无感**。我觉得这才是真正的解耦工具应当具备的功能。

# Lotusoot 的重大缺点和下一版本目标

Lotusoot 的缺点是显而易见的，虽然通过脚本可以在编译阶段创建好『协议-服务类』关系表。但 `Lotusoot.py` 识别『协议-服务类』是通过注解来的，而这里的注解其实就是注释，不能编译检测错误，及时误写错也无法及时检查出问题。如果解决了这一痛点，就可以相对完美的解决了 Swift 的模块化方案。

目前的思路如下：

尝试通过全局方法或是其他语法方式实现真正的注解，可以像 Java 中的注解一样，不仅可以作为一种标识，也可以进行编译检查。

其实 OC 中是可以直接用宏定义做的：

```swift
#define Service(_name_) \
+ (void)load { \
    [self registerService:_name_]; \
}

// 使用
@Service(@"LoginService")
@implementation LoginService
...
@end
```


但 Lotusoot 是提供给 Swift 和 OC 以及混编项目都可以使用的，所以实现方案还需要我继续探索。

另一种方式可以使用 LLVM 是提供了 @annotation 操作的，如果通过这种方式生成 `.plist` 的注册列表文件应该会放到编译结束时。

以上，是以后的一些构思，希望可以完美的解决 Swift 模块化方案。**如果你有什么好的建议，都可以来找我讨论哦~~~**

# Demo 和 Github

如果想更清晰的感受 Lotusoot 带来的模块化改造，请必须下载 [Demo](https://github.com/Vegetarians/Lotusoot/tree/master/Demo) 来看哟。

总项目的地址在[这里](https://github.com/Vegetarians/Lotusoot)。

非常欢迎一起讨论（卖萌~~）


----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)

