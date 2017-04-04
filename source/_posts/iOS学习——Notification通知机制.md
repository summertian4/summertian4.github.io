title: iOS学习——Notification通知机制
date: 2015-09-25 20:37
tags:
  - iOS
categories:
  - iOS
---


## **iOS中的事件有哪几种**
iOS的事件机制一共三种分别是

1. addTarget方式
2. delegate代理方式
3. Notification通知机制

-----

## **通知机制的原理**
1. iOS的通知分为通知发布者和通知监听者，通知将会放在**`NSNotificationCenter`**中。
2. 通知发布者发布带有信息（或者不带有信息）的通知，放置到**`NSNotificationCenter`**中。
3. 通知监听者可以选择需要监听的对象。
4. 要注意的是，在编码中是先放置监听者，再放置通知发布者。保证通知发布时已经有监听者在监听。

-----

## **通知机制的实现**
相信你看了下面的代码一定能理解，博主把能打上的注释全部打上了

```objc
//
//  main.m
//  day34_通知
//
//  Created by 周凌宇 on 15/9/3.
//  Copyright (c) 2015年 周凌宇. All rights reserved.
//

#import <Foundation/Foundation.h>
#import "NotificationListener.h"
#import "NotificationSender.h"

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        // 实例化一个通知发布者
        NotificationSender *sender1 = [[NotificationSender alloc] init];
        // 实例化一个通知监听者
        NotificationListener *listener1 = [[NotificationListener alloc] init];
        
        // 1.获取NSnotificationCenter对象
        NSNotificationCenter *notificationCenter = [NSNotificationCenter defaultCenter];
        
        // 2.先监听
        // 参数1：要监听通知的对象
        // 参数2：该对象的哪个方法监听
        // 参数3：被监听的通知的名称
        // 参数4：发布通知的对象
        // 如果参数3为nil，表示参数4指定的对象发布的所有通知都会被参数2指定的方法监听
        // 如果参数4为nil，表示无论哪个给对象，只要发布的通知名称与参数3相同，都会被监听
        // 如果参数3、4都为nil 那么所有对象发布的所有通知都会被监听
        [notificationCenter addObserver:listener1 selector:@selector(listen:) name:@"通知1" object:sender1];
        
        // 3.向通知中心发布通知
        [notificationCenter postNotificationName:@"通知1" object:sender1 userInfo:@{@"title" : @"阅兵", @"content" : @"阅兵将在9点开始"}];
        
        // 4.移除通知，在监听者的dealloc中移除
        
        
    }
    return 0;
}

```

<!--more-->

```objc
//
//  NotificationSender.h
//  day34_通知
//
//  Created by 周凌宇 on 15/9/3.
//  Copyright (c) 2015年 周凌宇. All rights reserved.
//

#import <Foundation/Foundation.h>

/**
 *  通知发布者
 */
@interface NotificationSender : NSObject

/**
 *  通知发布者名称
 */
@property (nonatomic, copy) NSString *name;

@end

```

```objc
//
//  NotificationSender.m
//  day34_通知
//
//  Created by 周凌宇 on 15/9/3.
//  Copyright (c) 2015年 周凌宇. All rights reserved.
//

#import "NotificationSender.h"

@implementation NotificationSender

@end

```

```objc
//
//  NotificationListener.h
//  day34_通知
//
//  Created by 周凌宇 on 15/9/3.
//  Copyright (c) 2015年 周凌宇. All rights reserved.
//

#import <Foundation/Foundation.h>

/**
 *  通知监听者
 */
@interface NotificationListener : NSObject

/**
 *  通知监听者名称
 */
@property (nonatomic, copy) NSString *name;

/**
 *  监听方法
 */
- (void)listen:(NSNotification *)noteInfo;

@end

```

```objc
//
//  NotificationListener.m
//  day34_通知
//
//  Created by 周凌宇 on 15/9/3.
//  Copyright (c) 2015年 周凌宇. All rights reserved.
//

#import "NotificationListener.h"

@implementation NotificationListener

- (void)listen:(NSNotification *)noteInfo {
    NSLog(@"监听到");
    NSLog(@"%@",noteInfo);
}

- (void)dealloc {
    // 移除监听
    // 在dealloc中移除可以确保监听者被销毁时能够移除监听
    [[NSNotificationCenter defaultCenter] removeObserver:self];
}

@end

```

## **源码**
博主也上传了一份源码http://download.csdn.net/detail/u010127917/9139983
希望同样在学习的同学能和小鱼一起进步~



----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)


