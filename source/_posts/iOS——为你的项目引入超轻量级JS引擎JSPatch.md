title: iOS——为你的项目引入超轻量级JS引擎JSPatch
date: 2016-01-16 16:10:11
tags:
  - iOS
categories:
  - iOS
---

# 1. JSPatch
## 简介

JSPatch是由国内一位年轻帅气的大牛bang编写的极小的JS引擎文件

这个开源项目([主页](http://jspatch.com)、[Github连接](https://github.com/bang590/JSPatch))，让你只需要在项目里引入极小的引擎文件，就可以使用 JavaScript 调用任何 Objective-C 的原生接口，替换任意 Objective-C 原生方法。

我了解这个项目的原因是因为项目的大神率先用了这个Mini引擎解决了我们线上BUG的燃眉之急，当我去学习这个引擎的时候，我觉得我的移动编码观被震撼了，这绝对是有重大意义的项目，所以推荐给所有还不了解人

## JSPatch解决了什么问题

JSPatch可以调用一段JS替换OC原生方法。而最关键的是JSPatch调用的JS不仅可以是本地的，还可以是网络请求来的。

试想一下，当你的项目已经上线了，用户反馈给你一堆BUG，作为一个优秀的程序员，你一定会第一时间修正BUG并且在一定修复量之后发布新的版本。但是问题是：你的用户不一定会更新新版本的app。

这里暴露了移动端的一个弊端：不能像服务器端一样实时发布，即使修复了BUG用户却可以不选择更新。

而JSPatch由于可以从网络请求JS替换你原本的代码，你可以在在服务器端设置一个借口获取一个JS文件，当你的项目出现问题是，你可以通过向JS文件中编写代码来实时修复你的BUG。所以JSPatch可以完美的实时修复线上bug。

<!--more-->


# 2. 使用

## 安装
常规的两种：

1. CocoaPods

	```objc
	# Your Podfile
	platform :ios, '6.0'
	pod 'JSPatch'
	```
2. 手动导入
	简历JSPatch/目录，拖入：
	
	```objc
	JSEngine.m
	JSEngine.h
	JSPatch.js
	```
## 使用

非常简单，在`AppDelegate`的`didFinishLaunchingWithOptions`方法中加入JSPatch的初始化代码

```objc
[JPEngine startEngine];
    
// 导入本地存放的js
NSString *sourcePath = [[NSBundle mainBundle] pathForResource:@"sample" ofType:@"js"];
NSString *script = [NSString stringWithContentsOfFile:sourcePath encoding:NSUTF8StringEncoding error:nil];
[JPEngine evaluateScript:script];
```

上面这些代码，是小鱼在本次演示中使用的，sourcePath指向的是项目中的sample.js（sample.js现在是空的）

除了上面我写的这种本地加载JS，更常用的是通过向服务器请求JS，JSPatch提供了非常全面的加载方法：

```objc
[JPEngine startEngine];

// 直接用NSString写一段JS
[JPEngine evaluateScript:@"\
 var alertView = require('UIAlertView').alloc().init();\
 alertView.setTitle('Alert');\
 alertView.setMessage('AlertView from js'); \
 alertView.addButtonWithTitle('OK');\
 alertView.show(); \
"];

// 从网络请求JS
[NSURLConnection sendAsynchronousRequest:[NSURLRequest requestWithURL:[NSURL URLWithString:@"http://cnbang.net/test.js"]] queue:[NSOperationQueue mainQueue] completionHandler:^(NSURLResponse *response, NSData *data, NSError *connectionError) {
    NSString *script = [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
    [JPEngine evaluateScript:script];
}];

// exec local js file
NSString *sourcePath = [[NSBundle mainBundle] pathForResource:@"sample" ofType:@"js"];
NSString *script = [NSString stringWithContentsOfFile:sourcePath encoding:NSUTF8StringEncoding error:nil];
[JPEngine evaluateScript:script];
```

OK，继续，我们来写一段错误代码

1. ViewController的Storyboard中拖入了一个UILabel，并且拖好线
2. 给ViewController加一个数组

	```objc
	// 现在的属性有：
	@property (weak, nonatomic) IBOutlet UILabel *lblText;
	@property (nonatomic, strong) NSMutableArray *array;
	```
3. 给数组加个懒加载

	```objc
	- (NSMutableArray *)array {
   	 if (_array == nil) {
    	    _array = [NSMutableArray array];
   	 }
  	  return _array;
	}
	```
4. 在viewDidLoad中这样写

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    
    [self.array addObject:@"first"];
    [self.array addObject:@"second"];
    [self.array addObject:@"third"];
    
    [self test];
}

- (void)test {
    NSLog(@"%@", self.array[3]);
    self.lblText.text = self.array[3];
}
```

显然会出错是不是？毕竟数组中只有3个元素

OK，运行，然后程序崩了

我们尝试用JSPatch解决
现在我们原来空空如也的sample.js派上了用场，在里面写入

```objc
// sample.js
defineClass('ViewController', {
            test: function() {
            // 获取数组中第一个元素
            var content = self.array().objectAtIndex(0);
            // 获取label控件
            var label = self.lblText();
            // 设置label控件的文字
            label.setText(content);
            
            }
            });
```
再运行，你会发现不会出错，页面中的Label显示了"first"字样

以上的JS起到的作用是：重新编写『ViewController』中的『test』方法

是不是非常简单，关于语法，在[这里](https://github.com/bang590/JSPatch/wiki/基础用法)查看JSPatch的wiki，不要害怕，都是非常易懂的，就算你没有学JS，妹子我觉得这是见过最良心的文档了，简单易读还是中文，bang帅哥棒棒的


## 小结
想想一下，上面的问题，如果这时候你的项目已经上线了，所有用户浏览这个页面时都会崩溃，必须及时修复。Don't worry，如果你已经预先在`AppDelegate`的`didFinishLaunchingWithOptions`中做了网络加载，你只需要去修改那段js，所有线上的用户问题都会被及时解决


# 3. 源码
老规矩，这里奉上上面demo的[源码](http://download.csdn.net/detail/u010127917/9407227)



----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)

