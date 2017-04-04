title: iOS——教你如何实现瀑布流
date: 2015-12-01 10:07:19
tags:
  - iOS
categories:
  - iOS
---

# 1 GitHub

博主已经写好了一份瀑布流的框架，如果你想直接使用，可以点击进入[CFWaterFlowView](https://github.com/summertian4/iOS-CFWaterFlowView)的项目主页。

[CFWaterFlowView](https://github.com/summertian4/iOS-CFWaterFlowView)是博主已经封装好的瀑布流框架，轻量级、简单、易用，希望你喜欢。

# 2 简介

瀑布流是一种非常常用的UI布局，可以为用户带来沉浸式的体验，不需要打断用户的阅读。非常适合带图片信息的展示。



![CFWaterFlowView展示](http://7xnrog.com1.z0.glb.clouddn.com/github_iOS-CFWaterFlowView-show-02.jpg-w375)

![瀑布流样式示例](http://www.hong16.net/wp-content/uploads/2015/03/v3.jpg)

<!--more-->

# 3 思路与实现

## 3.1 选择什么作为基类

-> 首先你要想到这个能滚动的布局一定是通过UIScrollView实现的

-> 那么现在有三种选择
	
1. 通过UITableView实现
2. 通过UICollectionView实现
3. 通过UIScrollView实现

-> 但是UITableView只能实现每行一个cell，而UICollectionView的每个cell的大小又是相同的
-> 那么最后选择通过最基础的UIScrollView实现瀑布流

## 3.2 如何提供更好的接口

一般来说，为了使你提供的API更易用，可以参考官方是如何构建自己的类的。

既然瀑布流是通过UIScrollView实现的，又类似于UITableView，有cell的概念，那么久可以参考UITableView的API

所以你需要：

1. 定义瀑布流数据源协议`CFWaterFlowViewDataSource`
2. 定义瀑布流代理协议`CFWaterFlowViewDelegate`
3. 定义并实现realoadData方法

仿照UITableView做好数据源和代理协议

```objc
#pragma mark - ========================代理定义=======================
@protocol CFWaterFlowViewDelegate <UIScrollViewDelegate>

@optional
/**
 *  返回对应索引的cell的高度
 *
 *  @param waterFlowView CFWaterFlowView对象
 *  @param index         索引
 *
 *  @return 对应索引的cell的高度
 */
- (CGFloat)waterFlowView:(CFWaterFlowView *)waterFlowView heightAtIndex:(NSUInteger)index;
/**
 *  点击cell回调
 *
 *  @param waterFlowView CFWaterFlowView对象
 *  @param index         索引
 */
- (void)waterFlowView:(CFWaterFlowView *)waterFlowView didSelectCellAtIndex:(NSUInteger)index;
/**
 *  返回对应间距类型的间距
 *
 *  @param waterFlowView CFWaterFlowView对象
 *  @param type          间距类型
 *
 *  @return 对应间距类型的间距
 */
- (CGFloat)waterFlowView:(CFWaterFlowView *)waterFlowView marginForType:(CFWaterFlowViewMarginType)type;


@end
```


```objc
#pragma mark - ========================数据源定义========================
@class CFWaterFlowView;
@protocol CFWaterFlowViewDataSource <NSObject>

@required
/**
 *  一共多少cell
 *
 *  @param waterFlowView CFWaterFlowView对象
 *
 *  @return cell总个数，NSUInteger保证正数
 */
- (NSUInteger)numberOfCellsInWaterFlowView:(CFWaterFlowView *)waterFlowView;
/**
 *  返回对应索引的cell
 *
 *  @param waterFlowView CFWaterFlowView对象
 *  @param index         索引
 *
 *  @return 对应索引的cell
 */
- (CFWaterFlowViewCell *)waterFlowView:(CFWaterFlowView *)waterFlowView cellAtIndex:(NSUInteger)index;

@optional
/**
 *  一共多少列，如果数据源没有设置，默认为2列
 *
 *  @param waterFlowView CFWaterFlowView对象
 *
 *  @return 瀑布流列数
 */
- (NSUInteger)numberOfColumnsInWaterFlowView:(CFWaterFlowView *)waterFlowView;

@end
```

需要注意的是数据源协议中比UITableViewDataSource多出一个方法

```objc
@optional
/**
 *  一共多少列，如果数据源没有设置，默认为2列
 *
 *  @param waterFlowView CFWaterFlowView对象
 *
 *  @return 瀑布流列数
 */
- (NSUInteger)numberOfColumnsInWaterFlowView:(CFWaterFlowView *)waterFlowView;

@end
```
通过这个方法可以向数据源索取各种类型的间距，间距类型是一个枚举类型，定义如下：

```objc
#pragma mark - ========================枚举定义========================
typedef enum {
    CFWaterFlowViewMarginTypeTop,
    CFWaterFlowViewMarginTypeBottom,
    CFWaterFlowViewMarginTypeLeft,
    CFWaterFlowViewMarginTypeRight,
    // 列间距
    CFWaterFlowViewMarginTypeColumn,
    // 上下相邻cell间距
    CFWaterFlowViewMarginTypeRow
} CFWaterFlowViewMarginType;
```
![CFWaterFlowView展示](http://7xnrog.com1.z0.glb.clouddn.com/blog_iOS——教你如何实现瀑布流-01.png-w375)


定义reloadData方法和代理、数据源引用

```objc
#pragma mark - ========================类定义=======================
@interface CFWaterFlowView : UIScrollView

/**
 *  数据源对象
 */
@property (nonatomic, weak) id<CFWaterFlowViewDataSource> dataSource;
/**
 *   代理对象
 */
@property (nonatomic, weak) id<CFWaterFlowViewDelegate> delegate;

/**
 *  刷新数据
 *  调用该方法会重新向数据源和代理发送请求。获取数据
 */
- (void)reloadData;

@end
```

## 3.3 reloadData中应该做什么

reloadData中我们需要：

1. 计算每一个cell的尺寸位置，并存放到一个数组`@property (nonatomic, strong) NSMutableArray *cellFrames;`中
2. 设置contentSize使CFWaterFlowView能够滚动

为了计算每一个cell的frame需要获得：

1. cell总数
2. 瀑布流列数
3. 各个类型的边距
4. 每个cell的高度

因为代理方法并不是强制实现的，所以我们要设定几个默认值，避免在代理没有实现定义的方法时瀑布流能够正常显示：

```objc
// 默认cell的高度为50
#define CFWaterFlowViewDefaultCellH 50
// 默认瀑布流为3列
#define CFWaterFlowViewDefaultColumnsCount 3
// 默认所有间距为10
#define CFWaterFlowViewDefaultMargin 10
```


## 3.4 reloadData的实现

```objc
- (void)reloadData {
    // 1.计算每一个cell的尺寸位置
    // cell总数
    NSUInteger cellsCount = [self.dataSource numberOfCellsInWaterFlowView:self];
    // 瀑布流列数
    NSUInteger columnsCount = [self numberOfColumns];
    
    CGFloat marginTop = [self marginForType:CFWaterFlowViewMarginTypeTop];
    CGFloat marginBottom = [self marginForType:CFWaterFlowViewMarginTypeBottom];
    CGFloat marginLeft = [self marginForType:CFWaterFlowViewMarginTypeLeft];
    CGFloat marginRight = [self marginForType:CFWaterFlowViewMarginTypeRight];
    CGFloat marginRow = [self marginForType:CFWaterFlowViewMarginTypeRow];
    CGFloat marginColumn = [self marginForType:CFWaterFlowViewMarginTypeColumn];
    
    CGFloat cellW = (self.width - marginLeft - marginRight - (columnsCount - 1) * marginColumn) / columnsCount;
    
    CGFloat maxYOfColumns[columnsCount];
    for (int i = 0; i < columnsCount; i++) {
        maxYOfColumns[i] = 0.0;
    }
    
    for (int i = 0; i < cellsCount; i++) {
        NSUInteger cellColumn = 0;
        NSUInteger maxYOfColumn = maxYOfColumns[cellColumn];
        
        // 找到当前最短的一列
        for (int j = 0; j < columnsCount; j++) {
            if (maxYOfColumns[j] < maxYOfColumn) {
                // 这个cell将会加在该列
                cellColumn = j;
                maxYOfColumn = maxYOfColumns[j];
            }
        }
        
        CGFloat cellH = [self heightAtIndex:i];
        CGFloat cellX = marginLeft + cellColumn * (cellW + marginColumn);
        
        CGFloat cellY = 0;
        
        if (maxYOfColumn == 0.0) { //第一行需要有间距
            cellY = marginTop;
        } else {
            cellY = maxYOfColumn + marginRow;
        }
        
        CGRect cellFrame = CGRectMake(cellX, cellY, cellW, cellH);
        [self.cellFrames addObject:[NSValue valueWithCGRect:cellFrame]];
        
        // 更新这一列的最大Y值
        maxYOfColumns[cellColumn] = CGRectGetMaxY(cellFrame);
    }
    
    // 设置contentSize
    CGFloat contentH = maxYOfColumns[0];
    
    // 找到当前最短的一列
    for (int i = 0; i < columnsCount; i++) {
        if (maxYOfColumns[i] > contentH) {
            contentH = maxYOfColumns[i];
        }
    }
    contentH += marginBottom;
    
    self.contentSize = CGSizeMake(0, contentH);
}
```

# 4 缓存池实现
## 4.1 内存浪费(1)

在上一步骤中，计算了每个cell的frame，但是并没有予以显示，那么如何显示呢？

最简单的方法当然是：计算好frame -> 新建一个CFWaterFlowViewCell -> 的frame -> 添加到CFWaterFlowView上

但这显然会造成性能问题：**因为处在屏幕之外的cell不需要显示，如果过早创建cell对象，会造成大量的内存浪费**

正确的这做法是：先判断cell是否在屏幕显示范围内

1. 如果在：创建cell对象 -> 设置frame -> 显示到CFWaterFlowView上
2. 如果不在：暂时不做操作

按照以上思想，可以开始实现显示cell。

那么在什么时候可以`[self addSubview:cell]`呢？

-> 经过上面的分析reloadData中只应该计算每一个cell的frame，不能马上添加。

-> 进一步考虑UIScrollView特性，UIScrollView可以滚动，每次滚动都应该判断当前有哪些cell在屏幕范围内，并予以显示。所以可以考虑在UIScrollView滚动的时候判断、添加cell。

-> UIScrollView在滚动时会调用`- (void)layoutSubviews`方法，所已在`layoutSubviews`中对cell进行布局最合适。

## 4.2 解决内存浪费(1)代码实现：

1. 首先编写一个方法，能够判断一个frame是否在当前屏幕显示范围内

```objc
/**
 *  判断给定frame是否在显示范围内
 *
 *  @param frame
 *
 *  @return 给定frame是否在显示范围内
 */
- (BOOL)isInScreen:(CGRect)frame {
    // contentOffset.y 滚动到的y值
    return (CGRectGetMaxY(frame) > self.contentOffset.y) && (CGRectGetMinY(frame) < self.contentOffset.y + self.height);
}
```

2. 实现`layoutSubviews`方法

```objc
- (void)layoutSubviews {
    
    NSUInteger cellsCount = self.cellFrames.count;
    for (int i = 0; i < cellsCount; i++) {
        // 对应的frame
        CGRect cellFrame = [self.cellFrames[i] CGRectValue];
        // 如果该frame在屏幕显示范围内，加载cell
        CFWaterFlowViewCell *cell = self.displayingCells[@(i)];
        if ([self isInScreen:cellFrame]) { // 在屏幕上
            if (cell == nil) {
                // 向代理索取一个cell
                cell = [self.dataSource waterFlowView:self cellAtIndex:i];
                cell.frame = cellFrame;
                [self addSubview:cell];
            }
            
        } else {// 不在屏幕上
            if (cell != nil) {
                [cell removeFromSuperview];
            }
        }
    }
}
```

## 4.3 内存浪费(2)

上一步骤我们实现了按需求添加cell，但是`layoutSubviews`会在CFWaterFlowView滚动时候不断的调用**（哪怕只有微小的滚动），**这样会导致**cell被疯狂的重复添加**。

-> 为了解决这一问题，我们需要保证在cell从未添加到CFWaterFlowView时才执行`[self addSubview:cell]`

-> 为此我们需要设立一个属性保存已经添加到CFWaterFlowView的cell，为了避免重复，使用一个Dictionary`@property (nonatomic, strong) NSMutableDictionary *displayingCells;`来保存，key值为cell的索引(index)

-> 如果在添加前检索到已经有对应索引cell在Dictionary中，就不再重复添加

-> 更改`layoutSubviews`代码

```objc
- (void)layoutSubviews {
    
    NSUInteger cellsCount = self.cellFrames.count;
    for (int i = 0; i < cellsCount; i++) {
        // 对应的frame
        CGRect cellFrame = [self.cellFrames[i] CGRectValue];
        // 如果该frame在屏幕显示范围内，加载cell
        CFWaterFlowViewCell *cell = self.displayingCells[@(i)];
        if ([self isInScreen:cellFrame]) { // 在屏幕上
            if (cell == nil) {
                // 向代理索取一个cell
                cell = [self.dataSource waterFlowView:self cellAtIndex:i];
                cell.frame = cellFrame;
                [self addSubview:cell];
                self.displayingCells[@(i)] = cell;
            }
            
        } else {// 不在屏幕上
            if (cell != nil) {
                [cell removeFromSuperview];
                [self.displayingCells removeObjectForKey:@(i)];
            }
        }
    }
}

```

## 4.4 缓存池
按照一个步骤，能对性能做了一些优化，但仍然再存在问题：**没有类似UITableView的cell缓存池功能，并提供接口让用户能够使用缓存池中的cell**

`

对于缓存池，我们很容易想到用一个Set来作为缓存池。
建立属性`@property (nonatomic, strong) NSMutableSet *reusableCells;`作为缓存池

仿照UITableView，定义并实现缓存池方法`- (CFWaterFlowViewCell *)dequeueReusableCellWithIdentifier:(NSString *)identifier;

在`dequeueReusableCellWithIdentifier`方法中我们需要：

1. 遍历缓存池，检索符合`(NSString *)identifier`的cell
2. 如果检索到，从缓存池中移除该cell -> 返回该cell
3. 如果没有检索到，返回nil

## 4.5 缓存池代码实现

```objc
/**
 *  根据ID查找可循环利用的cell
 *
 *  @return 可循环利用的cell
 */
- (CFWaterFlowViewCell *)dequeueReusableCellWithIdentifier:(NSString *)identifier {
    
    __block CFWaterFlowViewCell *reusableCell = nil;
    [self.reusableCells enumerateObjectsUsingBlock:^(CFWaterFlowViewCell *cell, BOOL * stop) {
        if ([cell.identifier isEqualToString:identifier]) {
            reusableCell = cell;
            *stop = YES;
        }
    }];
    
    if (reusableCell != nil) { // 如果缓存池中有
        // 从缓存池中移除
        [self.reusableCells removeObject:reusableCell];
    }
    
    return reusableCell;
}
```

# 5 反馈

以上已经完成了瀑布流框架的完整实现

如果你需要直接使用该框架，访问GitHub项目地址：[https://github.com/summertian4/iOS-CFWaterFlowView](https://github.com/summertian4/iOS-CFWaterFlowView)

希望你能够喜欢本套框架以及博主这个萌萌哒大四女程序员^_^

如果有什么修改建议，可以发送邮件到coderfish@163.com，也欢迎到我的[博客](http://zhoulingyu.com)

----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)


