title: iOS学习——Object-C模拟类变量
date: 2015-08-17 11:18
tags:
  - iOS
categories:
  - iOS
---


Object-C中并没有JAVA中的类变量，博主在刚开始时，又是给@property加static修饰，又是给字段加static修饰，都会报错。查阅文档得到以下信息：
>  1.  static关键字不能用于修饰成员变量
>  2.  static关键字只能用于修饰**局部变量**、**全局变量**、**函数**
>  3.  static修饰局部变量，代表将该局部变量存放到静态存储区
>  4.  static修饰全局变量，可以限制该全局变量只能被当前源文件访问
>  5.  static修饰函数，可以限制该函数只能在当前源文件调用

那么现在可以明白OC中的static修饰符跟不少语言中的static不是一回事。

但事实上，类变量确实是面向对象中很有必要的。现在要用另一种方式模拟类变量。

解决方案是：**建立static的全局变量，并在实现public的get、set方法，供外部调用**

下面给出代码示例，相信你能很快明白其用法。

```objc
//
//  Person.h
//  test_command
//
//  Created by 周凌宇 on 15/8/11.
//  Copyright (c) 2015年 周凌宇. All rights reserved.
//

#import <Foundation/Foundation.h>

@interface Person : NSObject

+(void) setPlanetName: (NSString *)planetName;
+(NSString *) planetName;

@end
```

<!--more-->

```objc
//
//  Person.m
//  test_command
//
//  Created by 周凌宇 on 15/8/11.
//  Copyright (c) 2015年 周凌宇. All rights reserved.
//

#import "Person.h"


static NSString *_planetName;

@implementation Person

+(void) setPlanetName: (NSString *)planetName {
    _planetName = planetName;
}

+(NSString *) planetName {
    return _planetName;
}

@end
```

```objc
//
//  main.m
//  test_command
//
//  Created by 周凌宇 on 15/8/11.
//  Copyright (c) 2015年 周凌宇. All rights reserved.
//

#import <Foundation/Foundation.h>
#import "Person.h"

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        [Person setPlanetName:@"地球"];
        NSLog(@"%@",[Person planetName]);
    }
    return 0;
}
```



----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)

