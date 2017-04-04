title: iOS学习——页面间的传值
date: 2015-07-20 16:39
tags:
  - iOS
categories:
  - iOS
---


学会ios的页面间传值。

和android不同，iOS传值不是通过Intent，而是更直接的方法，直接在被传值类中定义一个存放值的变量，在上一个页面直接将值放入，达到传值的目的。

## 故事板：

![故事板](http://img.blog.csdn.net/20150720163651231)

-------------------

## 功能描述:

```flow
st=>start: 开始
e=>end: 结束
s1=>operation: 在输入框内输入内容
s2=>operation: 点击下一页按钮
s3=>operation: 在Label中显示上一页输入框内容
st->s1->s2->s3->e
```

-------------------

## 代码:

第一个页面替换自己成自己写的类getDataViewController

- createDataViewController.h

```objc
//
//  createDataViewController.h
//  coderfish003
//
//  Created by apple40 on 15-7-20.
//  Copyright (c) 2015年 coderfish. All rights reserved.
//

#import <UIKit/UIKit.h>

@interface createDataViewController : UIViewController

// 定义输入框
@property (strong, nonatomic) IBOutlet UITextField *tfCreateData;

@end

```

<!--more-->

- createDataViewController.m

```objc
//
//  createDataViewController.m
//  coderfish003
//
//  Created by apple40 on 15-7-20.
//  Copyright (c) 2015年 coderfish. All rights reserved.
//

#import "createDataViewController.h"
#import "getDataViewController.h"

@interface createDataViewController ()

@end

@implementation createDataViewController

- (id)initWithNibName:(NSString *)nibNameOrNil bundle:(NSBundle *)nibBundleOrNil
{
    self = [super initWithNibName:nibNameOrNil bundle:nibBundleOrNil];
    if (self) {
        // Custom initialization
    }
    return self;
}

- (void)viewDidLoad
{
    [super viewDidLoad];
	// Do any additional setup after loading the view.
}

- (void)didReceiveMemoryWarning
{
    [super didReceiveMemoryWarning];
    // Dispose of any resources that can be recreated.
}

// 页面传值函数
- (void)prepareForSegue:(UIStoryboardSegue *)segue sender:(id)sender
{
    NSString *data = self.tfCreateData.text;
    // 获得getDataViewController对象引用
    getDataViewController *controller = (getDataViewController *)[segue destinationViewController];
    // 给getDataViewController的getData标签内容赋值
    controller.getData = data;
}

@end

```

第二个页面替换自己成自己写的类getDataViewController

- getDataViewController.h
```
//
//  getDataViewController.h
//  coderfish003
//
//  Created by apple40 on 15-7-20.
//  Copyright (c) 2015年 coderfish. All rights reserved.
//

#import <UIKit/UIKit.h>

@interface getDataViewController : UIViewController
@property (strong, nonatomic) IBOutlet UILabel *label;
@property(strong,nonatomic)NSString *getData;
@end

```

 - getDataViewController.m
 
```
//
//  getDataViewController.m
//  coderfish003
//
//  Created by apple40 on 15-7-20.
//  Copyright (c) 2015年 coderfish. All rights reserved.
//

#import "getDataViewController.h"

@interface getDataViewController ()

@end

@implementation getDataViewController

- (id)initWithNibName:(NSString *)nibNameOrNil bundle:(NSBundle *)nibBundleOrNil
{
    self = [super initWithNibName:nibNameOrNil bundle:nibBundleOrNil];
    if (self) {
        // Custom initialization
    }
    return self;
}

- (void)viewDidLoad
{
    [super viewDidLoad];
	// Do any additional setup after loading the view.
    self.label.text = self.getData;
}

- (void)didReceiveMemoryWarning
{
    [super didReceiveMemoryWarning];
    // Dispose of any resources that can be recreated.
}

@end

```

-------------------

##结果
在第一页的输入框输入文字，点下一页在下一页中显示文字
![这里写图片描述](http://img.blog.csdn.net/20150720163812732)
<br/ >
![这里写图片描述](http://img.blog.csdn.net/20150720163826866)

----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)

