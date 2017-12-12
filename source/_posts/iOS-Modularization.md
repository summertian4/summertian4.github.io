---
title: iOS 混编 模块化/组件化 经验指北
date: 2017-11-24 11:03:52
tags:
	- iOS模块化
	- iOS
categories:
	- iOS
	- iOS模块化
---

# 1. 开篇

本文的初衷，是为了给正在做混编或者模块化的同学们一个建议和参考。

因为来饿厂以后做的项目是全公司唯一一个 Swift/OC 混编的 iOS 项目，所以一路上踩坑无数，现在把一些踩坑的过程和经验总结起来，供大家参考。

相信在浏览本文后，一定会有所收获。

我来的时候项目已经开始 Swift 改造了，慢慢的把项目 Swift 化，新代码都是 Swift 的。

先公布七个月成果，下图是我们最终的项目结构：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS-Modularization-03.png)

对于我们混编的情况，在五个月前大家就展开了讨论。

给我们的选择有两种：

1. 慢慢将 OC 代码替换成 Swift
2. 尽快模块化，分离两种语言代码

一开始我们是从 `选择1` 开始做的，但是很快我们就发现，对于我们 74% 都是 OC 代码的项目来说，太痛了，太漫长了，而且期间迭代的过程中还在不断地迭代，不断的耦合。

所以在经过一番利害分析后我们迅速投入到了 `选择2` 中。一方面，模块化本身就是越来越臃肿的项目的最终归宿，一方面可以慢慢将两种语言剥离。

<!-- More -->

> 注：这里的模块化，也就是大家说的『组件化』，不是在主工程用文件夹分模块，而是指将独立模块抽调成 CocoaPods 库、或者其他形式的库文件，成为一个独立工程。

# 2. 模块划分

**刀怎么切，是混编模块化最重要的一步**，完全决定了后续工作的难与否。

不用从业务模块拆分，类似『实时订单模块』、『历史订单模块』、『个人中心』这样直接拆分，保准你后面哭到无法自已。

正确的做法应该从底层部分开始抽离，首先能想到的应该是『类扩展 Extension』、『工具类』、『网络库』、『DB 管理』（当然这个我们没有用到比较重的 DB）。

平常我们看到一些大型库，或者一些公司介绍自己产品架构时候都是什么样的？是不是下层有 OpenGL ES 和 Core Graphics 才有上层 Core Animation，再到 UIKit。下层决定上层，只有把复用率高的部分抽出才能逐步构建上层业务。

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS-Modularization-01.png)

所以首先我们做的就是抽工具类和 Extension，诸如：

1. 各类 Constants 文件
2. `NSTimer`、`NSString`、`UILabel` 等等类的 Extension
3. `RouterHelper`、`JavascripInterface` 等等 Utils 和 Helper

这一块的工作，不仅仅可以抽出 OC 代码，也同时可以抽出 Swift 的代码。我们将 OC 部分的代码新建了库为 `LPDBOCFoundationGarbage`，Swift 部分的代码新建库为 `LPDBPublicModule`。

## 2.1 LPDBOCFoundationGarbage

先说 `LPDBOCFoundationGarbage`，叫这个名字显然不仅仅会放入上面所提到的文件。`LPDBOCFoundationGarbage` 还会**大量放入长期不跟随业务变动的 OC 代码**。这是因为，在实践中，我们发现总是『理想很美好』，虽然大家都抱有把旧代码整理一遍的愿望，但是实际上，我们项目的旧代码已经到了剪不断理还乱的地步，所以期望一边整理、一边分离的想法基本是不可靠的。这时候就要借用 [MM](https://github.com/mmoaay) 大佬给我们传授的一句话『让恶心的代码恶心到一起』，`LPDBOCFoundationGarbage` 正是为此而创建。

**大量放入长期不跟随业务变动的 OC 代码**包括：

1. 自定义的 Customer View，诸如：Refresh 控件、Loading 控件、红点控件等等
2. 自定义的小型控制器，诸如：TextField 和其五六个过滤器 PhoneNumValidator、IDCardValidator 等等
3. 不随业务变动的 Controller，诸如：自定义的 AlertController、自定义的 WebController、自定义的 BaseViewController 等等

最后我们的一级列表看起来就像这样：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS-Modularization-04.png-w375)

> 关于前缀说两句。我们所有抽出的库都带有前缀 `LPDB`，但是针对 Swift 库和 OC 库稍有区分的是，OC 库内的文件也都带有前缀，而 Swift 库是去掉了前缀，这也符合两种语言的规范。

## 2.2 LPDBPublicModule

`LPDBPublicModule` 情况很简单，主要是新业务迭代时候产生的一些复用性高的代码，但是这显然和 OC 那个垃圾桶库不一样，要干净整洁的多。主要存放的是：

1. Swift Extension
2. [Lotusoot](https://github.com/Vegetarians/Lotusoot) 及其他公开协议

> [Lotusoot](https://github.com/Vegetarians/Lotusoot) 是个由我开发的模块化工具和规范，一开始我叫它『路由』，但是随后发现部门这边因为叫它『路由库』而曲解了它的意思，所以后来我就叫『模块化工具』了。关于 Lotusoot 可以查看[这篇](http://zhoulingyu.com/2017/11/29/iOS-modularized-tool-Lotusoot/)。

## 2.3 LPDBNetwork

这块毋庸置疑，不管什么项目都基本有的一块，基本上我们项目中网络相关的旧代码都是 OC 的，唯一比较麻烦的是，我们的网络层，早期人员写的比较粗糙，甚至和 UI 层代码有很多耦合，比如网络请求中和网络请求失败有一些 HUD 显示，转转菊花什么的。所以导致在从主工程抽离的时候有很多恶心的地方。

**所以对于这种强耦合，最后解决的方式是分成了两遍代码改造，第一遍先通过反射先将 OC 代码抽出，保证代码可用，通过基础测试。第二遍是通过协议来代替原先的反射。第三遍是使用 [Lotusoot](https://github.com/Vegetarians/Lotusoot) 彻底规范服务调用。在后面一节『过程中的一些难点总结』中会介绍**

## 2.4 LPDBUIKit

这块是 Swift 的 UI 库，一些比较常用到的控件等等。

## 2.5 LPDBEnvironment

这块是用于环境控制的，切换要访问的服务器环境，这块本身可以不抽出的，但是由于有其他基础模块，比如 `LPDBNetwork` 依赖，而且其中相关代码比较多，环境相关的代码也比较独立，所以单独抽出。

# 3. 业务模块抽离

到这里为止，比较底层的代码就基本抽出结束了，剩下的就可以较为轻松一些的抽取业务库了。

抽取业务库的重点在于：

1. 抽取的业务库不会经常改动，以防止在抽取、重构过程中由于业务需求发生更动
2. 抽取的业务库可以高度独立，抽取后应当和积木一样，如 `LPDBLoginModule`，抽取后快速被集成在任何模块，并能保证登录功能，更好的服务其他模块

我们目前抽出的三个业务模块分别是： `LPDBHistoryModule`、`LPDBUserCenterModule`、`LPDBLoginModule`。

# 4. 过程中的一些重难点

剩下的就是，来说一下在这个过程中的疑难问题。


## 4.1 处理模块耦合代码-反射调用

抽取代码第一遍使用反射的原因主要是，通常你在递归某个文件的依赖的时候，会递归出非常多的东西（尤其是我们的蜜汁旧代码），往往就是 **A->B->C->D->F**，中间有各种依赖，甚至到最后一层的时候还引用了 Swift 的类。直到最后你看 `#import` 就想吐。给个图感受一下：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS-Modularization-05.png)

为什么没有办法一步到位，通过协议解决耦合？

这主要是因为单个 Pod 库开发时使用开发模式是很容易调试的，但是两个 Pod 库同时在不发版本的情况下使用开发模式是比较难处理的（可以参考[这篇文章](http://www.pluto-y.com/cocoapod-private-pods-and-module-manager/)中『使用私有库』一节）。这种情况下，反复操作两个或者两个以上的库是麻烦的，所以优先考虑将代码尽快分离开来，并能通过基本测试，不影响功能。

所以在这一遍处理结束后，子库中出现了很多 `NSClassFromString` 等等。

以 `LPDBLoginMoudle` 为例：

```objc
NSString *className = [NSString stringWithFormat:@"%@.`AuthLoginManager", [NSString targetName]];
id authLoginManager = NSClassFromString(className);
if (![authLoginManager conformsToProtocol:@protocol(authLoginSuccess)]) {
    return;
}
[authLoginManager authLoginSuccess];
```

```objc
id delegate = [[UIApplication sharedApplication] delegate];
[delegate jumpToShopListVC:shops];
```

## 4.2 处理模块耦合代码-协议调用

保持第一遍中充满 `NSClassFromString`  是不可取的，因为这类代码往往属于硬编码，不能在类名出现改动、或者方法名出现改动的时候及时在编译阶段抛出 error。

在这里引出一段讨论。

之前跟大神们讨论组件化（模块化）的具体实践时候，说到了主流的组件化可能都借用了 `+ (void)load` 方法和 rumtime 操作来注册路由和服务。这时候 [casa](https://casatwy.com/) 大神提出了一种说法『组件化的根本目的是隔离、隔离问题影响域、隔离业务、隔离开发时的依赖。所以让两个本来有关系的人变得没有关系，就需要一个中间人，如果不用 runtime 能省掉不少事，但是用 URL 是一件相对来说比较多余的事，一个包含了 target-action 的字符串就足够了，URL 是字符串的更复杂表征，target-action 的意义体现的更明显。同时 URL 应该仅限于 H5 调度和跨 App 的 URL Scheme 调度』。

> 这里要向 [casa](https://casatwy.com/) 大神非常非常郑重的道歉，上面一段，原来在第一版的时候是预留修改的片段，本想再读一遍大神 [《 [iOS应用架构谈 组件化方案]》](https://casatwy.com/iOS-Modulization.html) 仔细理解以后再次修改这块，本来是悄咪咪的发了文章，没想到被推送出去了，有引导大家曲解大神的愿意。非常非常抱歉！现在已经修改。
> 下面在贴上大佬自己对 URL 的见解：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS-Modularization-09.png)


那个时候听了 casa 大神的说法觉得『哎？有道理』，但是在后期的实践中，我觉得就我个人的代码习惯，是希望尽可能的将问题暴露在编译阶段，能让它抛出 error 就抛出 error，纵使使用字符串可以定义常量，但由于大家不是独立负责项目，在其他人看到你的方法参数时，比如：`+ (void)callService:(NSString *)sUrl` 或者 `+ (void)openURL:(NSString *)url` ，对方发现你的参数是 NSStrring，很有可能直接出现硬编码字符串而不去查阅常量列表，这是习惯性编码很容易出现的问题。但我对 casa 『URL 没有 target-action 表征明显』是非常仍可的，所以 Lotusoot 的重点只在于解耦的服务调用，URL 只是为了更好的为 H5 页面提供外部调用服务，在工程内部大可使用更加简洁的方式。


最后一点原因是，反射或者通过类/方法字符串字典的方式实在太 OC 了，不管怎么样我们是一个尽量 Swift 化的项目，应该尽量吸取其优点，虽然抽出的 OC 库可以使用反射，那 Swift 库咋办？目前 Swift3 与 4 都没有很好的支持反射。

所以，第二遍处理使用协议替换反射是很有必要的。但实质上，处理的并不是很好。大致如下（我们以 `LPDBLoginModule` 为例）：

### 4.2.1 在 LPDBLoginModule 整理用到的服务，归类整理

如我们的 `LPDBLoginModule` 用到了 AppDelegate 中的一些方法，同事用到了 AuthLogin 相关类中的一些方法

### 4.2.2 在 LPDBLoginModule 中建立相应的协议

即建立 `AuthLoginDelegate.h` 和 `AppDelegateProtocol`

大致的代码如下：

```objc
@protocol AppDelegateProtocol <NSObject>

- (void)jumpToHomeVC;
- (void)jumpToShopListVC:(NSArray *)shops;
- (CLLocationCoordinate2D)getCoordinate;

@end
```

```objc
@protocol AuthLoginDelegate <NSObject>[Pods](media/Pods.)
+ (void)authLoginSuccess;
@end
```

### 4.2.3 在主工程中去实现协议

AppDelegateProtocol 由 AppDelegate 扩展实现：

```objc
@import LPDBLoginModule;
@interface AppDelegate (Protocol)  <AppDelegateProtocol>
@end

@implementation AppDelegate (Protocol)
- (CLLocationCoordinate2D)getCoordinate {
    ...
}
- (void)jumpToHomeVC {
    ...
}
- (void)jumpToShopListVC:(NSArray *)shops {
    ...
}
@end
```

AuthLoginDelegate 由 AuthLoginManager(这个 Manager 在主工程中是 swift 编写的) 实现：

```swift
extension AuthLoginManager: AuthLoginDelegate {
    static func authLoginSuccess() {
        ...
    }
}
```

### 4.2.4 在 LPDBLoginModule 调用服务

```objc
id delegate = [[UIApplication sharedApplication] delegate];

if (![delegate conformsToProtocol:@protocol(AppDelegateProtocol)]) {
    return;
}
CLLocationCoordinate2D coordinate = [delegate coordinate];
```

```objc
NSString *className = [NSString stringWithFormat:@"%@.AuthLoginManager", [NSString targetName]];
id authLoginManager = NSClassFromString(className);
if (![authLoginManager conformsToProtocol:@protocol(LPDBAuthLoginDelegate)]) {
     return;
}
[authLoginManager authLoginSuccess];
[self jumpToSelectShopView:shops];
```

经过这些改造之后，模块间的状态如图所示：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS-Modularization-06.png)

但是，可以很明显感受到，这次的改变并不彻底：

1. 还是存在大量的 `![delegate conformsToProtocol:@protocol(AppDelegateProtocol)]` 这样的判断，仅仅是起到了容错，保证不会 crash，但是却不能将问题暴露在编译阶段。
2. `AppDelegateProtocol` 明明是一个公共的，多个模块使用的协议，却被定义到了 `LPDBLoginModule`
3. 概念颠倒，理想状态下，应该是各个子模块提供协议和实现，告知其他模块可以调用该模块哪些功能。而目前是子模块告知其他模块需要调用哪些方法，由其他模块实现。

那么为了彻底解决问题，我们引入了 [Lotusoot —— 组件通信和工具](https://github.com/Vegetarians/Lotusoot)。

## 4.3 处理模块耦合代码-Lotusoot

 [Lotusoot](https://github.com/Vegetarians/Lotusoot) 的最初目的就是为了解决模块间的耦合，并且同时支持 OC 和 Swift 使用，也是这几个月中去做的一个比较重要的东西，库本身小巧灵活，包含的东西也很少，但是起到的规范作用却是我非常满意的一点。

Lotusoot 规范的核心思想主要是以下几步，我们同样使用上面的 `LPDBLoginModule 为例`：

### 4.3.1 建立共用模块——LPDBPublicModule

`LPDBPublicModule`中定义了各个模块可以提供的服务，做成协议，称为 Lotus，一个 Lotus 协议包含了一个模块的所有的能调用的方法的列表。举例如下：

```swift
@objc public protocol AppDelegateLotus {
    func jumpToHomeVC()
    func jumpToSelectShopVC(shops: [Any], isNapos: Bool)
    func getCoordinate() -> CLLocationCoordinate2D
}
```

```swift
@objc public protocol MainLotus {
    func authLoginSuccess()
}
```

### 4.3.2 各个模块中，实现 LPDBPublicModule 中对应的 Lotus 协议

实现协议的 Class 称为 Lotusoot。举例如下：

```swift
class AppDelegateLotusoot: NSObject, AppDelegateLotus {

    func jumpToHomeVC() {
        ...
    }
    
    func jumpToSelectShopVC(shops: [Any], isNapos: Bool) {
        ...
    }

    func getCoordinate() -> CLLocationCoordinate2D {
        ...
    }
}
```

```swift
class MainLotusoot: NSObject, MainLotus {
    func authLoginSuccess() {
        ...
    }
}
```

### 4.3.3 注册服务

**需要着重说明的是，这一步是可以省略的，通过 Lotusoot 提供的脚本和注解，可以自动为所有的路由进行注册。请移步 [Lotusoot](https://github.com/Vegetarians/Lotusoot)参考『3. 注解与规范』部分。**

`didFinishLaunchingWithOptions` 中注册服务：

```objc
[LotusootCoordinator registerWithLotusoot:[AppDelegateLotusoot new] lotusName:@"AppDelegateLotus"];
    [LotusootCoordinator registerWithLotusoot:[MainLotusoot new] lotusName:@"MainLotus"];
```

### 4.3.3 在其他模块中调用服务

现在只需要 `import Lotusoot`、`import ModulePublic`

```objc
id<MainLotus> mainModule = [LotusootCoordinator lotusootWithLotus:@"MainLotus"];
[mainModule authLoginSuccess];
```

```objc
// 如果使用字符串 @"AppDelegateLotus" 注册，建议定义在 LPDBPublicModule
// 也可以使用 NSStirngFromClass(AppDelegateLotus.class)
id<AppDelegateLotus> appDelegateLotus = [LotusootCoordinator lotusootWithLotus:@"AppDelegateLotus"];
[appDelegateLotus goToHomeVC];
```

无论 OC 还是 Swift，都可以顺畅调用

```swift
// 或者使用类似字符串 "AccountLotus"，但需要你管理好 kAccountLotus，尽量不要硬编码
let appDelegateLotus = s(AppDelegateLotus.self) 
let appDelegateLotusoot: AppDelegateLotus = LotusootCoordinator.lotusoot(lotus: appDelegateLotus) as! AppDelegateLotus
accountModule.goToHomeVC()
```

```swift
let mainLotus = s(MainLotus.self) 
let mainModule: MainLotus = LotusootCoordinator.lotusoot(lotus: mainLotus) as! MainLotus
mainModule.authLoginSuccess()
```

到此为止，就比较完整的解决了模块间耦合。清爽的风格用一张图表示就是这样（这是我在做 Lotusoot 解说时候用的一张配图）：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS-Modularization-07.png)

`LPDBPublicModule` 中的 `Lotus` 协议，像一张清单列出了所有模块提供的服务声明，而在各个模块中，直接通过这些公共协议就可以调用想要的服务。很多问题都可以在编译前和编译阶段显示出来（如果模块不提供服务，是不能通过编译的；如果没有一项服务没有声明，是不能通过编译的）。


## 4.4 语言耦合

我们抽模块中一个重要的目的就是『分割两种语言』，但是实践过程中，会发现，分割语言比分割业务还要难。

一个 Pod 库中只能包含一种语言，但往往，在抽离代码的最后，会发现有无数的**基础 Model 耦合**，如：

```objc
@interface ShopInfo : LPDBModel

...
@property (nullable, nonatomic, strong) DeliveryService *workingProduct;
@property (nullable, nonatomic, strong) DeliveryService *preEffectiveProduct;

@end
```

```swift
class DeliveryService: BaseModel {
    ...
}
```

如果需要将 `ShopInfo` 和 `DeliveryService` 抽出到一个模块时，必须要『有舍有得』，在涉及到基础 Model 语言不同时，可以适当的重写，因为 Model 的代码量是极小的，Model 通常也只包含属性声明，作为数据传输的中介，即使更改，产生的不可预支错误的可能性也较低。

如果要抽出的模块主体使用 OC，那么可以将 `DeliveryService` 重新用 OC 编写。

但要注意，要先尽量通过拆分更基础的服务模块，在考虑重新编写文件，保证项目的稳定性。

## 4.5 模块的积木化

模块化的最终目的，不仅仅是去耦，还应当让每个模块像积木一样，随意拼接，最后达到主工程完全没有代码，通过 Pod 集成各个模块，组成完整的功能。而每个模块也应当可以独立测试，独立开发。

还是以 `LPDBLoginModule` 和 `LPDBNetWort` 为例。

登录模块是一个非常特殊的模块，所有的子模块如果想独立测试和开发，一般都需要通过登录验证，比如订单模块，必须要登录后，该业务模块内能才能正确的拉取订单信息。

由于 `LPDBLoginModule` 依赖基础库 `LPDBNetWort`，`LPDBNetWort` 需要做的有：

1. 包含 cer 文件，可以正确的提供给其他模块正常的 https 接口访问
2. 便利的网络服务调用

而 `LPDBLoginModule` 至少要做的事有：

1. 可以正确的保存登录信息，完成登录操作
2. 提供登录的 UI 界面，可以直接调用 LoginVC

在具备以上功能后，`LPDBLoginModule` 就可以快速的集成进其他模块，为其他模块提供独立开发、独立测试的功能。

## 4.6 资源打包

上一小结提到『 `LPDBLoginModule` 要提供登录的 UI 界面』。对于 UI 界面，需要做的是资源打包，在模块拆分中，要非常注意资源分割。

_**因为业务模块的划分，不仅仅是是代码抽出，也有资源抽出。**_

资源库包括但不仅限于：

1. `.xib` 文件
2. 声音资源
3. 图片资源
4. 纯文本文件
5. 视频资源

所以，所有的资源文件，应当单独创立 `Res` 文件夹，放入其中，并在 `.podspec` 中表明资源文件路径

```shell
s.resources 	 = ["Source/**/*.xib", "Source/Res/*.xcassets"]
```

> 注意图片资源，如果想保留 @2x、@3x，是可以按照 xcassets 的格式直接 copy 过来的。

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS-Modularization-08.png)


# 5 结尾

以上是我在混编项目中进行 模块化/ 组件化的经验总结，写成了指导的模式，希望这篇文章能对走同样路的人有所帮助，希望你们会有所收获，么么哒。

----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)

