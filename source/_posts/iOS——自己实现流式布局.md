title: iOS——自己实现流式布局
date: 2015-10-28 13:51:33
tags:
  - iOS
categories:
  - iOS
---


# 描述
因为做项目的时候，需要用到这样布局，也就是通常所说的流式布局

![流式布局目标图](http://img.blog.csdn.net/20151028135937412)

----------

# 思路
最先想到的是用UICollectionView做，因为UICollectionView也可以做成瀑布流的样子，虽然平常看到的都是纵向的，但做成横向应该也不是什么问题。而且UICollectionView可以重用Cell。

但是考虑了实际需求，我们项目需要的是一个条件选择栏，像这样，类似于淘宝：

![条件选择栏](http://img.blog.csdn.net/20151028142014475)

实际上，每栏的按钮大约5~8个，而且本来每栏都是一个TableViewCell，所以重用的问题不需要考虑了。

所以最后决定自己封装一个FlowView来实现流式布局的效果。

<!--more-->
----------


# 核心代码

当你看完下面这些代码的时候，你就会明白，布局本身是非常简单的，基本如下：

 1. 设置一个`row`变量，记录当前按钮排到了第几行
 2. 设置一个数组`rowFirstButtons` 放置每一个行第一个按钮
 3. 安置第一个按钮的位置（居左、居右），并将按钮加入`rowFirstButtons` 中
 4. 在安置其后的按钮位置：
	 4.1  计算从本行第一个按钮到该按钮所有的宽度及间距和，如果超过了父控件的宽度，`row++` ，并将其加入`rowFirstButtons` 中。计算相应的x、y值
	 4.2 如果没有超过过父控件的宽度，算相应的x、y值

```objc
- (void)layoutSubviews {

    CGFloat margin = 10;

    // 存放每行的第一个Button
    NSMutableArray *rowFirstButtons = [NSMutableArray array];
    
    // 对第一个Button进行设置
    UIButton *button0 = self.buttonList[0];
    button0.x = margin;
    button0.y = margin;
    [rowFirstButtons addObject:self.buttonList[0]];
    
    // 对其他Button进行设置
    int row = 0;
    for (int i = 1; i < self.buttonList.count; i++) {
        UIButton *button = self.buttonList[i];
        
        int sumWidth = 0;
        int start = (int)[self.buttonList indexOfObject:rowFirstButtons[row]];
        for (int j = start; j <= i; j++) {
            UIButton *button = self.buttonList[j];
                sumWidth += (button.width + margin);
        }
        sumWidth += 10;
        
        UIButton *lastButton = self.buttonList[i - 1];
        if (sumWidth >= self.width) {
            button.x = margin;
            button.y = lastButton.y + margin + button.height;
            [rowFirstButtons addObject:button];
            row ++;
        } else {
            button.x = sumWidth - margin - button.width;
            button.y = lastButton.y;
        }
    }
    
    
    UIButton *lastButton = self.buttonList.lastObject;
    self.height = CGRectGetMaxY(lastButton.frame) + 10;
    

}

```

因为将计算布局的代码放在了layoutSubviews方法中，所以适应横竖屏转换，不管是父控件如何变化都能正确设置布局。

![竖屏显示](http://7xnrog.com1.z0.glb.clouddn.com/iOS-CFFlowButtonView-01.png-w500)

![横屏显示](http://7xnrog.com1.z0.glb.clouddn.com/iOS-CFFlowButtonView-02.png-h500)


----------


# 问题
小鱼已经把FlowView封装成控件，小伙伴们可以直接使用了，只要给FlowView对象塞入一个按钮数组即可。

现在的问题主要有这几点：

 1. 目前FlowView只能放入按钮
 2. 按钮本身需要已经有宽高
 3. 按钮的高度要相同

对于这三点问题

 1. 对于第一点：
 在稍后的版本中将更新为可以放入所有UIView控件，这里改动并不大，控件的共性本身就很大。
 
 2. 对于第二点：
 首先是控件本身确实应该是有宽高之后FlowView再能计算。但是对于UILabel和UIButton两个控件，在稍后的版本将会在其没有设置宽高的时候由FlowView通过计算按钮和标签文字的宽高来自动设置。
 
 3. 对于第三点：


----------


# 使用

小鱼已经将源码上传。

[github链接](https://github.com/summertian4/iOS-CFFlowButtonView)

使用的时候拖入CFFlowButtonView文件夹下的文件至项目中。

实例化一个CFFlowButtonView对象

```objc
CFFlowButtonView *flowButtonView = [[CFFlowButtonView alloc] initWithButtonList:buttonList];
```

设置CFFlowButtonView的约束或者frame，`注意` ，对CFFlowButtonView设置约束时`不需要` 设置高度相关的约束。

更多用法可以查看DEMO。

----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)

