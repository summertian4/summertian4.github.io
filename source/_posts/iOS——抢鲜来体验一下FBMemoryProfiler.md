title: iOS——抢鲜来体验一下FBMemoryProfiler
date: 2016-04-21 21:32:31
tags:
  - iOS
categories:
  - iOS
---
# FBMemoryProfiler简介
1. 4月刚刚推出的新品~保证中你口味
2. FaceBook荣誉出品~
3. 作者[Gricha](https://github.com/Gricha)
4. 功能及前身：也许你用过[FBRetainCycleDetector](https://github.com/facebook/FBRetainCycleDetector)，这就是[FBMemoryProfiler](https://github.com/facebook/FBMemoryProfiler)的前身了，是由Gricha编写用来检测循环引用的小工具。FBMemoryProfiler比FBRetainCycleDetector要强大的多，提供了非常帮的`交互界面`和更多的功能。

![FBMemoryProfiler一览](https://raw.githubusercontent.com/facebook/FBMemoryProfiler/master/Example/Images/Example2.gif)

<!--more-->

# 初上手
1. **pod安装**

	```objc
	pod 'FBMemoryProfiler'
	```
	
2. **enable FBAllocationTracker:**

	在`mian.m`文件中（注意是`main.m`，不是AppDelegate.m）中加入：
	
    ```objc
	#import <FBAllocationTracker/FBAllocationTrackerManager.h>

	int main(int argc, char * argv[]) {
  	[[FBAllocationTrackerManager sharedManager] startTrackingAllocations];
  	[[FBAllocationTrackerManager sharedManager] enableGenerations];
  	@autoreleasepool {
    	  return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
  	}
	}
    ```
	
3. **enable FBMemoryProfiler**:

	这次是去`AppDelegate.m`了，去加一个变量，保证MemoryProfiler不会被释放：
	
	```objc
	@property (nonatomic , strong) FBMemoryProfiler * memoryProfiler;
	```
	
	在`didFinishLaunchingWithOptions`方法中添加：
	
	```objc
	self.memoryProfiler = [FBMemoryProfiler new];
                               retainCycleDetectorConfiguration:nil];
	[self.memoryProfiler enable];
	```

4. 试一试，运行你的APP是不是能看到小方块了

	![初始样式](http://7xt4xp.com2.z0.glb.clouddn.com/blog_iOS%E2%80%94%E2%80%94%E6%8A%A2%E9%B2%9C%E6%9D%A5%E4%BD%93%E9%AA%8C%E4%B8%80%E4%B8%8BFBMemoryProfiler-01.PNG-w375)
	![展开](http://7xt4xp.com2.z0.glb.clouddn.com/blog_iOS%E2%80%94%E2%80%94%E6%8A%A2%E9%B2%9C%E6%9D%A5%E4%BD%93%E9%AA%8C%E4%B8%80%E4%B8%8BFBMemoryProfiler-02.PNG-w375)
	
5. 展开后点击Expand，可以看到所有的类，你可以通过`Filter`过滤你想看到的结果，比如你的项目是CF开头，就可以输入CF来过滤

	![](http://7xt4xp.com2.z0.glb.clouddn.com/blog_iOS%E2%80%94%E2%80%94%E6%8A%A2%E9%B2%9C%E6%9D%A5%E4%BD%93%E9%AA%8C%E4%B8%80%E4%B8%8BFBMemoryProfiler-03.JPG-w375)
	
6. 还有很多功能，去尝试吧！
	

# 添加自己的pulgin
除了基本功能，FBMemoryProfiler还提供了接口扩展功能

## 1. FBMemoryProfilerPluggable
你可以阅读`FBMemoryProfilerPluggable.h`
FBMemoryProfilerPluggable是一个协议，你可以通过实现这个协议来扩展你想要的功能
提供了如下方法：

```objc
@protocol FBMemoryProfilerPluggable <NSObject>

@optional

/**
 Lifecycle events, when Memory profiler is enabled/disabled
 */
- (void)memoryProfilerDidEnable;
- (void)memoryProfilerDidDisable;

/**
 When memory profiler finds retain cycles plugins can subscribe to get them.
 */
- (void)memoryProfilerDidFindRetainCycles:(nonnull NSSet *)retainCycles;

/**
 If you want to do additional cleanup after marking new generations.
 */
- (void)memoryProfilerDidMarkNewGeneration;

@end
```

## 2. 实现FBMemoryProfilerPluggable协议
来看看`Example`中提供简单的插件

```objc
/**
 * Copyright (c) 2016-present, Facebook, Inc.
 * All rights reserved.
 *
 * This source code is licensed under the license found in the LICENSE file in
 * the root directory of this source tree.
 */

#import <Foundation/Foundation.h>

#import <FBMemoryProfiler/FBMemoryProfiler.h>

/**
 Example of FBMemoryProfiler plugin that will NSLog all retain cycles found
 within FBMemoryProfiler. This could, for instance, send it somewhere to the backend.
 */

@interface RetainCycleLoggerPlugin : NSObject <FBMemoryProfilerPluggable>

@end

```

```objc
/**
 * Copyright (c) 2016-present, Facebook, Inc.
 * All rights reserved.
 *
 * This source code is licensed under the license found in the LICENSE file in
 * the root directory of this source tree.
 */
#import "RetainCycleLoggerPlugin.h"

@implementation RetainCycleLoggerPlugin

- (void)memoryProfilerDidFindRetainCycles:(NSSet *)retainCycles
{
    if (retainCycles.count) {
        DDLogWarn(@"%@", retainCycles);
    }
}
@end
```

其功能就是简单的打印（`DDLogWarn是CocoaLumberjack提供的Log方法`你可以换成你自己的）

## 3. 修改FBMemoryProfiler的初始化方法
写好插件后需要用起来，去改一下AppDelegate中的FBMemoryProfiler初始化方法即可

```objc
self.memoryProfiler = [[FBMemoryProfiler alloc] initWithPlugins:@[[CacheCleanerPlugin new],
                                                                  [RetainCycleLoggerPlugin new]] retainCycleDetectorConfiguration:nil];
[self.memoryProfiler enable];      
```

重新运行项目，点击`Retain Cycles`按钮，是不是能看到控制台有打印了

![](http://7xt4xp.com2.z0.glb.clouddn.com/blog_iOS%E2%80%94%E2%80%94%E6%8A%A2%E9%B2%9C%E6%9D%A5%E4%BD%93%E9%AA%8C%E4%B8%80%E4%B8%8BFBMemoryProfiler-04.png-w500)

# 补充
FBMemoryProfiler目前只是0.1版本，不能算稳定，我在使用的时候也Crash了好几次。但是谁不相信FaceBook的大神呢~嘿嘿



----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)

