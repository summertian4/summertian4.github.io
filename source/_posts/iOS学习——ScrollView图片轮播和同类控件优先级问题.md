title: iOS学习——ScrollView图片轮播和同类控件优先级问题
date: 2015-08-24 18:13
tags:
  - iOS
categories:
  - iOS
---



## 1. 布置界面
ScrollView的使用非常简单，只有三步
  1.1	添加一个scrollview
  1.2	向scrollview添加内容
  1.3	告诉scrollview中内容的实际大小

首先做第一步，布置界面。
拖拽一个scrollview就可以了
![拖拽一个scrollview](http://img.blog.csdn.net/20150824172017044)
就这么简单

<!--more-->

----------

## 2. 添加内容，并告诉scrollview中内容的实际大小
先给scrollview拖线至viewController的.m文件类扩展中

```objc
@property (weak, nonatomic) IBOutlet UIScrollView *scrollView;
```

再要给scrollview设置代理，让代理处理相应的事件、设置相应的数据。设置代理的方法有两种：拖线和代码设置。

拖线：
![拖线设置代理](http://img.blog.csdn.net/20150824172605867)

代码设置，在`viewDidLoad`方法中设置scrollview的代理为控制器本身。

```objc
self.scrollView.delegate = self;
```

注意，把控制器设为代理需要实现相应代理的接口：
![实现相应代理的接口](http://img.blog.csdn.net/20150824172909552)

现在开始添加内容，向Images.xcassets放入要用的五张图片素材，因为博主做的5张图是白色背景，为了能看清，只能把控制器背景调成其他颜色。
![素材](http://img.blog.csdn.net/20150824173801758)

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
    // 动态创建UIImageView添加到滚动控件
    CGFloat imgW = 340;
    CGFloat imgH = 150;
    CGFloat imgY = 0;
    CGFloat imgX;
    // 加入五张图片
    for (int i = 0; i < 5; i++) {
        UIImageView *imgView = [[UIImageView alloc] init];
        // 计算每张图的x值
        imgView.image = [UIImage imageNamed:[NSString stringWithFormat:@"%d", i + 1]];
        imgX = i * imgW;
        imgView.frame = CGRectMake(imgX, imgY, imgW, imgH);
        [self.scrollView addSubview:imgView];
    }
    // 设置滚动控件内容大小
    self.scrollView.contentSize = CGSizeMake(5 * self.scrollView.frame.size.width, 150);
}
```

解释为什么要计算每张图片的x轴：
图片实际上是这样摆放的：
![这里写图片描述](http://img.blog.csdn.net/20150824174737294)
橘色的是scrollView，每张图片和scrollView长宽一样大，所以只能看到一张图片，五张图片并排放置，这样滚动scrollView时候就能浏览后面的图片，实现滚动效果。

如果不希望有滚动指示器，就是滚动条，可以加入：

```objc
// 设置滚动指示器不可见
self.scrollView.showsHorizontalScrollIndicator = NO;
self.scrollView.showsVerticalScrollIndicator = NO;
```
这时运行后发现可以滚动，但是图片可以滚动到任何位置
<a href="http://i1.tietuku.com/e149d03ede7b0f30.jpg" title="点击显示原始图片"><img src="http://i1.tietuku.com/e149d03ede7b0f30t.jpg"></a>
如果希望能够像平常淘宝看到的图片轮播，轻轻划一下，可以完整的翻到下一张图片，可以加入：

```objc
// 实现分页效果，原理是根据滚动控件的宽度，一个宽度是一页
self.scrollView.pagingEnabled = YES;
```
<a href="http://i1.tietuku.com/fa374a4a1ad8d16b.jpg" title="点击显示原始图片"><img src="http://i1.tietuku.com/fa374a4a1ad8d16bt.jpg"></a>


----------


## 3. 设置自动轮播
通过计时器控件计时

1  先为控制器添加一个timer

```objc
@property (nonatomic, strong) NSTimer *timer;
```

2 `viewDidLoad`中创建计时器控件

```objc
// 创建计时器控件，设定每过两秒执行`scrollImage`方法
self.timer = [NSTimer scheduledTimerWithTimeInterval:2.0 target:self selector:@selector(scrollImage) userInfo:nil repeats:YES];
```

3 实现@selector指定的方法

```objc
- (void) scrollImage {
    NSInteger page = self.scrollView.contentOffset.x / self.scrollView.frame.size.width;
    // 判断是否在最后一页，如果是最后一页设置页码重新设置为第一页
    if (page == 4) {
        page = 0;
    } else {
        page ++;
    }
    CGFloat offsetX = page * self.scrollView.frame.size.width;
    [self.scrollView setContentOffset:CGPointMake(offsetX, 0) animated:YES];
}
```

这时候就能看见图片2秒滚动一次了。


----------


## 4. 同类控件优先级问题
### 问题描述
当界面中有两个scrollview（或其子类控件）时，在滚动其中一个scrollview时候，设置了计时器的scrollview会暂定自动滚动。
比如拽入一个textView:
<a href="http://i1.tietuku.com/c2abd5a14b54c34c.jpg" title="点击显示原始图片"><img src="http://i1.tietuku.com/c2abd5a14b54c34ct.jpg"></a>
如果滚动下面的文字，上面的图片轮播不会继续自动滚动。

### 问题原因
问题的原因是因为计时器的优先级和view（各种控件）的优先级不同，所以会优先执行view的动作。

### 解决方法
将计时器的优先级设置和view的优先级相同

```objc
// 修改优先级
// 获取当前的消息循环
NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
[runLoop addTimer:self.timer forMode:NSRunLoopCommonModes];
```


----------


## 源代码
如果需要源代码，博主已经上传：http://download.csdn.net/detail/u010127917/9042503



----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)


