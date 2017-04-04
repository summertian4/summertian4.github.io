title: iOS——objective-c已经支持泛型
date: 2015-10-27 13:31:07
tags:
  - iOS
categories:
  - iOS
---



# 偶然的发现
今天在大家一起做项目的时候，一个小伙伴在从svn上check out项目后报错，发现是这个方法：

```objc
-  (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    
}
```
当时写的时候并没有注意，重写父类方法的时候自动补齐的。
仔细看了以后居然发现用了泛型的写法。

以前一直没有发现oc能用泛型，挺苦恼的，大概是写惯了java。
应该是这次更新了xcode7之后才有的（也有可能小鱼之前眼瞎没看到，但是我应该是试过没有泛型的）。

不知道大家对泛型怎么看，我觉得是很有必要的机制。就像Java SE 1.5带来泛型之后迅速被广泛应用。泛型可以 `避免强制转换` ，`保证类型安全`。

----

# 编写

因为之前也有人通过自己构造，模拟泛型。所以我尝试了一下oc现在是不是真的支持泛型了。


这样写完全没有问题：
```objc
- (void)test {
    NSMutableArray<NSString *> *strArray = [NSMutableArray array];
    [strArray addObject:@"aString"];
    
}
```

而如果这样写，就会报出警告：

```objc
- (void)test {
    NSMutableArray<NSString *> *strArray = [NSMutableArray array];
    [strArray addObject:[NSNumber numberWithFloat:15.0]];
}
```

![类型错误警告](http://img.blog.csdn.net/20151027132341662)


另外尝试了NSSet也是可以的。

点进源码之后看到这样的描述：

```objc
@interface NSMutableArray<ObjectType> : NSArray<ObjectType>
```

```objc
@interface NSSet<__covariant ObjectType> : NSObject <NSCopying, NSMutableCopying, NSSecureCoding, NSFastEnumeration>
```

也就是说objective-c确实已经支持泛型了。

----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)


