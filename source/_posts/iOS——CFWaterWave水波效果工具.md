title: iOS——CFWaterWave水波效果工具
date: 2016-04-28 16:53:37
tags:
  - iOS
categories:
  - iOS
---


> 由于 `CF` 有 `Core Foundation` 歧义，更名前缀为 `ZLY`

### 最近通过前辈，学着做了一个水波效果工具共享出来，[github连接](https://github.com/summertian4/CFWaterWave)


# 简介
ZLYWaterWave是一个简单好用的iOS水波效果工具，可以让你的APP更加好看有趣

![ZLYWaterWave效果展示](https://raw.githubusercontent.com/summertian4/Images/master/blog/github_CFWaterWave_show_01.gif)

# 原理简介
ZLYWaterWave的原理很简单，我们用Example里的工程做简介。(这里首先要感谢@hy，我敬爱的前辈，最初是从他这里学习的水波效果原理)

![白色图片](https://raw.githubusercontent.com/summertian4/Images/master/blog/github_CFWaterWave_pic_white.png)
![红色图片](https://raw.githubusercontent.com/summertian4/Images/master/blog/github_CFWaterWave_pic_red.png)
![叠加添加遮盖效果](https://raw.githubusercontent.com/summertian4/Images/master/blog/github_CFWaterWave_img_03.png)

1. 首先准备两张图片
2. 将两张图放在重叠的位置
3. 将其中一张图片加上波浪形的遮盖
4. 如果波浪形的遮盖是动态再变化的的，就可以形成动态的波浪
5. ZLYWaterWave就是为你提供好了动态波浪的Path，你只需要在回调中加入遮盖即可
6. 如果你还是晕晕的，那就直接看Example吧，相信你瞬间就会明白的

<!-- More -->

# 安装
直接拽入`ZLYWaterWave.h`和`ZLYWaterWave.m`文件

# 使用ZLYWaterWave
1. 创建ZLYWaterWave对象

```objc
- (ZLYWaterWave *)waterWave {
    if (_waterWave == nil) {
        // 给定的frame和你的图片frame一致即可
        _waterWave = [[ZLYWaterWave alloc] initWithFrame:self.pic_red.frame];
        _waterWave.delegate = self;
    }
    return _waterWave;
}
```

2. 实现好代理，在代理中给你想要实现水波纹的图片加上贝塞尔路径生成的遮盖

```objc
- (void)waterWave:(ZLYWaterWave *)waterWave wavePath:(UIBezierPath *)path {
    CAShapeLayer *maskLayer = [[CAShapeLayer alloc] init];
    maskLayer.path = path.CGPath;
    // 添加遮盖
    self.pic_red.layer.mask = maskLayer;
}
```

# 用例
1. Exmaple中的示例

![CFWaterWave效果展示](https://raw.githubusercontent.com/summertian4/Images/master/blog/github_CFWaterWave_show_01.gif)

2. 最近做的一个小APP——『番茄』的效果展示

![CFWaterWave效果展示](https://raw.githubusercontent.com/summertian4/Images/master/blog/github_CFWaterWave_show_02.gif)



----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼](http://weibo.com/coderfish/)

