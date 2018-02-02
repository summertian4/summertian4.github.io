---
title: 关于 Pod 库的资源引用 resource_bundles or resources
date: 2018-02-02 18:29:09
tags:
---


# 1. 资源文件引用的方式

在第一节，先来介绍一下 CocoaPods 两种资源文件引用的方式——`resource_bundles` & `resources`

## 1.1 resource_bundles

`resource_bundles` 允许定义当前 Pod 库的资源包的**名称和文件**。用 hash 的形式来声明，key 是 bundle 的名称，value 是需要包括的文件的通配 patterns。

> We strongly recommend library developers to adopt resource bundles as there can be name collisions using the resources attribute.

CocoaPods 官方强烈推荐使用 `resource_bundles`，因为用 key-value 可以避免相同名称资源的名称冲突。

同时建议 bundle 的名称至少应该包括 Pod 库的名称，可以尽量减少同名冲突


Examples:

```ruby
spec.ios.resource_bundle = { 'MapBox' => 'MapView/Map/Resources/*.png' }
```

```ruby
spec.resource_bundles = {
    'MapBox' => ['MapView/Map/Resources/*.png'],
    'OtherResources' => ['MapView/Map/OtherResources/*.png']
  }
```

<!-- More -->


## 1.2 resources

使用 `resources` 来指定资源，被指定的资源只会简单的被 copy 到目标工程中（主工程）。

> We strongly recommend library developers to adopt resource bundles as there can be name collisions using the resources attribute. Moreover, resources specified with this attribute are copied directly to the client target and therefore they are not optimised by Xcode.

官方认为用 `resources` 是无法避免同名资源文件的冲突的，同时，Xcode 也不会对这些资源做优化。

Examples:

```ruby
spec.resource = 'Resources/HockeySDK.bundle'
```

```ruby
spec.resources = ['Images/*.png', 'Sounds/*']
```

> **FYI：**
> [Podspec Syntax Reference v1.4.0](https://guides.cocoapods.org/syntax/podspec.html)

# 2. 图片资源的管理

我们熟知平常用的 @2x @3x 图片是为了缩小用户最终下载时包的大小，通常我们会将图片放在 `.xcassets` 文件中管理，新建的项目也默认创建：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_pod-resource-reference-01.png)


使用 `.xcassets` 不仅可以方便在 Xcode 查看和拖入图片，同时 `.xcassets` 最终会打包生成为
 `Assets.car` 文件。对于 `Assets.car` 文件，[App Slicing](https://help.apple.com/xcode/mac/current/#/devbbdc5ce4f) 会为切割留下符合目标设备分辨率的图片，可以缩小用户最终下载的包的大小。

> **FYI：**
> [Xcode Ref Asset Catalog Format](https://developer.apple.com/library/content/documentation/Xcode/Reference/xcode_ref-Asset_Catalog_Format/FolderStructure.html#//apple_ref/doc/uid/TP40015170-CH33-SW1)
> [App thinning overview (iOS, tvOS, watchOS)](https://help.apple.com/xcode/mac/current/#/devbbdc5ce4f)

实际上，对于 Pods 库的资源，同样可以使用 `.xcassets` 管理。

# 3. 实际验证

> 不关注的可以直接跳到下面『结论』
 
官文中推荐了 `resource_bundles` 其理由主要是『可以解决同名冲突』和『Xcode为 bundle 提供的一些优化』。

我知道很多人看过 [这篇](http://blog.xianqu.org/2015/08/pod-resources/) 文章，里面提到 resource_bundles 不能使用 .xcassets。

那么到底是不是这样，我们需要亲自动手验证，看看两种引用方式，CocoaPods 到底为我们做了什么。

我们将在『实际验证』一节验证：

1. `resource_bundles` 是否能使用 *.xcassets 指定资源并正确打包
2. 同名冲突是怎样的

## 3.1 resource_bundles 是否能使用 *.xcassets 指定资源并正确打包

写两个 Demo Pod，同时创建好对应的 Example 测试工程。

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_pod-resource-reference-02.png)

对两个 Pod 分别使用不同的方式指定资源。

第一个 Demo Pod：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_pod-resource-reference-04.png)

第二个 Demo Pod：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_pod-resource-reference-03.png)

分别用了 `resource_bundles` 和 `resources` 两种方式引用。

`pod install` 后，观察结果。

### 3.1.1 使用 resources

`pod install` 并编译 Example 工程后，我们可以打开最后生成的 Product 文件下的内容：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_pod-resource-reference-06.png)

可以看到只有 一些 `.a` 文件，`.a` 文件是二进制文件。初次之外只有 `SubModule-Example.app`，打开包内容，可以看到只有一个 `Assets.car`。

这说明，使用 `resources` 之后只会简单的将资源文件 copy 到目标工程（Example 工程），最后和目标工程的图片文件以及其他同样使用 `resources` 的 Pod 的图片文件，统一一起打包为了一个 `Assets.car`。

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_pod-resource-reference-07.png)

再为 Pod 写一个 VC 来实验读取图片：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_pod-resource-reference-08.png)

读取图片的方式和平常使用的方式不同，要先获取 Bundle：

```objc
UIImage *image = [UIImage imageNamed:@"some-image"
                                inBundle:[NSBundle bundleForClass:[self class]]
           compatibleWithTraitCollection:nil];
```

在 Example 的 `ViewController` 写一下跳转 `SubModule/SMViewController`：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_pod-resource-reference-09.png)

运行之后，看一下，能正常访问图片：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_pod-resource-reference-10.png)


### 3.1.2 使用 `resource_bundles`

`pod install` 并编译 Example 工程后，同样找到 Product 文件下的内容：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_pod-resource-reference-11.png)

可以看到最终生成了一个 `SubModule_Use_Bundle.bundle`，打开看内部：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_pod-resource-reference-12.png)

发现包含了一个 `Assets.car`

再为 Pod 写一个 VC 来实验读取图片：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_pod-resource-reference-13.png)

由于还需要带上 .bundle 文件的路径，获取的方式又不同：

```objc
NSString *bundlePath = [[NSBundle bundleForClass:[self class]].resourcePath
                            stringByAppendingPathComponent:@"/SubModule_Use_Bundle.bundle"];
NSBundle *resource_bundle = [NSBundle bundleWithPath:bundlePath];
UIImage *image = [UIImage imageNamed:@"some-image"
                                inBundle:resource_bundle
           compatibleWithTraitCollection:nil];
```


在 Example 的 `ViewController` 写一下跳转 `SubModule/SMViewController`：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_pod-resource-reference-14.png)

运行之后，看一下，也能正常访问图片：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_pod-resource-reference-15.png)

	
### 3.1.3 resources 和 resource_bundles 对于 .xcassets 的支持

从 3.1 和 3.2 可以看出 resources 和 resource_bundles 都可以很好的支持 .xcassets 的引用。

所以，[这篇](http://blog.xianqu.org/2015/08/pod-resources/) 文章，里面提到 resource_bundles 不能使用 .xcassets 并不存在。应该说这篇文章已经比较老了，**CocoaPods 随着不断的更新，`resource_bundles` 已经可以很好的支持 `.xcassets` 了。**

## 3.2 同名资源的冲突问题

从上面的分析可以看出：

使用 `resources` 之后只会简单的将资源文件 copy 到目标工程（Example 工程），最后和目标工程的图片文件以及其他同样使用 `resources` 的 Pod 的图片文件，统一一起打包为了一个 `Assets.car`。

使用 `resource_bundles` 之后会为为指定的资源打一个 `.bundle`，`.bundle`包含一个 `Assets.car`，获取图片的时候要严格指定 `.bundle` 的位置，很好的隔离了各个库或者一个库下的资源包。

显然，使用 `resources`，如果出现同名的图片，显然是会出现冲突的，同样使用 `some-image` 名称的两个图片资源，不一定能正确调用到。

#### 3.2.1 简单验证 resources 重名问题

给 Example 文件添加一个同样叫 `some-image` 的图片：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_pod-resource-reference-16.png)

OK，现在的情况是 Example 工程自己有一个 `some-image` 图片资源，SubModule 这个 Pod 库也有一个 `some-image` 图片资源。

还是之前的显示图片的代码，再运行一下：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_pod-resource-reference-17.png)

可以看到，图片显然是用错了，显示了 Example 工程自己的 `some-image`

这就是 resources 的重名资源问题。

而使用 `resource_bundles` 则可以很好的避开这个问题。


# 4. 总结

**resource_bundles 优点：**

1. 可以使用 `.xcassets` 指定资源文件
2. 可以避免每个库和主工程之间的同名资源冲突

**resource_bundles 缺点：**

1. 获取图片时可能需要使用硬编码的形式来获取：`[[NSBundle bundleForClass:[self class]].resourcePath stringByAppendingPathComponent:@"/SubModule_Use_Bundle.bundle"]`

**resources 优点：**

1. 可以使用 `.xcassets` 指定资源文件

**resources 缺点：**

1. 会导致每个库和主工程之间的同名资源冲突
2. 不需要用硬编码方式获取图片：`[NSBundle bundleForClass:[self class]] compatibleWithTraitCollection:nil];`

So，一般来说使用 `resource_bundles` 会更好，不过关于硬编码，还可以再找找别的方式去避免。


# 5. Demo

本文所有的 Demo 代码都在 [这里](https://github.com/summertian4/iOS-ObjectiveC/tree/master/Pod-Resource-Reference)

----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼](http://weibo.com/coderfish/)

