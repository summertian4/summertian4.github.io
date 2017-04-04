title: iOS——教你如何使用ReactiveCocoa和MVVM为代码解耦构建清爽APP
date: 2016-05-20 12:39:44
tags:
  - iOS
categories:
  - iOS
---
# 1. MVVM简介
不过多赘述MVC，用最通俗的方式解说MVVM。

1. 拆解：
	1. **M：** `Model` ，包括数据模型、访问数据库的操作和网络请求等
	2. **V：** `View` ，包括了iOS中的 `View` 和 `controller` 组成，负责 UI 的展示，绑定 viewModel 中的属性
	3. **VM：** `ViewModel` ，负责从 `Model` 中获取 `View` 所需的数据，转换成 `View` 可以展示的数据，并暴露公开的属性和命令供 `View` 进行绑定
	4. **Binder：**这是我最近发现的，在标准MVVM中没有提到的一部分，但是如果使用MVVM + ReactiveCocoa就会自然地写出这一层。这一层主要为了实现响应式编程的功能，实现 `View` 和  `ViewModel` 的同步

<!--more-->

2. MVC——>MVVM：

	MVC是苹果官方推荐的开发模式，但是伴随这这模式产生的问题非常的多，这是随着项目的逐渐扩大、架构的逐渐复杂显示出来的，这也是为什么MVC也被调侃成**Massive View Controller（重量级视图控制器）**。大多数情况下，小型项目MVC开发不会带来太大的负担，即使你将大量的逻辑代码（不包括通用的工具类逻辑）放在了ViewController中，但只要该部分由一个人维护，相对来说还是可以保持逻辑清晰的。
	
	但当项目越来越大时，或者一个模块会有多个人维护时，读代码变成了一件非常困难的事，并且，MVC模式的iOS开发一直存在**难以测试**的问题。博主在做JAVA开发时JUnit的测试就像每天的必修课一样。开始iOS开发后，加上第二家公司一直没有QA，线上发现的BUG简直就是每天的噩梦。MVVM带来的好处是 `VM` 层可以**方便的做测试**，因为 `VM` 层是**独立的逻辑**，脱离对 `View` 和 `Model` 的依赖。
	
	少写字，多写代码，赶紧进入下一部分介绍，尽快去了解如何编码。
	
# 2. ReactiveCocoa
## 2.1 简介
1. **简介：**ReactiveCocoa简称RAC，集合了**函数式编程**和**响应式编程**，这也是为什么ReactiveCocoa被描述为函数响应式编程（FRP）框架。

2. ReactiveCocoa解决的问题：iOS开发中有多种事件处理方式，相信你一定也曾想过这些坑爹的地方，通常有这些事件处理方式：Action、delegate、Notification、KVO。并且通常这些代码总是散落在代码的各个角落，几度分散。ReactiveCocoa为事件提供了很多处理方法，可以把要处理的事情，和监听的事情的代码放在一起，非常方便管理。
3. 关于ReactiveCocoa的基本用法，希望你能认真的阅读[这篇博文](http://www.jianshu.com/p/87ef6720a096#)

## 2.2 常用宏
1. **`RAC(TARGET, [KEYPATH, [NIL_VALUE]])：`**用于给某个对象的某个属性绑定。

```objc
// 文本框文字改变时修改label的文字
    RAC(self.labelView,text) = _textField.rac_textSignal;
```

2. `**RACObserve(self, name)：**`监听某个对象的某个属性,返回的是信号，可以用来代替KVO

```objc
[RACObserve(self.view, center) subscribeNext:^(id x) {
        NSLog(@"%@",x);
}];
```
# 3. 实践
## 3.1 实现内容
做一个简单的登陆功能，两个输入框，一个登陆按钮。
简单的用户名密码验证，要求都在6位数以上即可，不符合要求时禁用登陆按钮。
## 3.2 Coding
### 界面大概是这样的感觉，简单的拉一个即可：
![禁用登录](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E2%80%94%E2%80%94%E6%95%99%E4%BD%A0%E5%A6%82%E4%BD%95%E4%BD%BF%E7%94%A8ReactiveCocoa%E5%92%8CMVVM%E4%B8%BA%E4%BB%A3%E7%A0%81%E8%A7%A3%E8%80%A6%E6%9E%84%E5%BB%BA%E6%B8%85%E7%88%BDAPP-01.PNG-w375)
![启用登录](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E2%80%94%E2%80%94%E6%95%99%E4%BD%A0%E5%A6%82%E4%BD%95%E4%BD%BF%E7%94%A8ReactiveCocoa%E5%92%8CMVVM%E4%B8%BA%E4%BB%A3%E7%A0%81%E8%A7%A3%E8%80%A6%E6%9E%84%E5%BB%BA%E6%B8%85%E7%88%BDAPP-01.PNG-w375)

### M层
抽出简单的User模型，Thin Model，不包含功能型方法：

```objc
#import <Foundation/Foundation.h>

@interface User : NSObject
@property (nonatomic, copy) NSString *username;
@property (nonatomic, copy) NSString *password;
@end

```

### VM层

```objc
#import <Foundation/Foundation.h>
#import "ReactiveCocoa.h"

@interface LoginViewModel : NSObject

@property (nonatomic, strong) NSString   *userName;
@property (nonatomic, strong) NSString   *password;
// 成功信号
@property (nonatomic, strong) RACSubject *successSubject;
// 失败信号
@property (nonatomic, strong) RACSubject *failureSubject;
// 错误信号
@property (nonatomic, strong) RACSubject *errorSubject;

/**
 *  按钮是否可点信号
 *
 *  @return
 */
- (RACSignal *)validSignal;
/**
 *  登陆操作
 */
- (void)login;

@end
```
----

```objc
#import "LoginViewModel.h"
#import "User.h"

@interface LoginViewModel ()

/** 用户名改变信号 */
@property (nonatomic, strong) RACSignal *userNameSignal;
/** 密码改变信号 */
@property (nonatomic, strong) RACSignal *passwordSignal;
/** 请求数据（模拟） */
@property (nonatomic, strong) NSArray *requestData;

@end


@implementation LoginViewModel

- (instancetype)init {
    if (self = [super init]) {
        // RACObserve(self, name):监听某个对象的某个属性,返回的是信号。
        _userNameSignal = RACObserve(self, userName);
        _passwordSignal = RACObserve(self, password);
        _successSubject = [RACSubject subject];
        _failureSubject = [RACSubject subject];
        _errorSubject = [RACSubject subject];
    }
    return self;
}

/**
 *  按钮是否可点信息
 *
 *  @return
 */
- (RACSignal *)validSignal {
    RACSignal *validSignal = [RACSignal combineLatest:@[_userNameSignal, _passwordSignal] reduce:^id(NSString *userName, NSString *password){
        // 要求用户名和密码大于6位数
        return @(userName.length >= 6 && password.length >= 6);
    }];
    return validSignal;
}

/**
 *  登陆操作
 */
- (void)login{
    // 网络请求进行登录，当然这里只是模拟一下
    User *user = [[User alloc] init];
    user.username = self.userName;
    user.password = self.password;
    _requestData = @[user];
    // 成功发送成功的信号
    [_successSubject sendNext:_requestData];
    // 如果失败发送失败的信息号
}

@end
```

通过这一层，你是否发现，VM是可以单独测试的，也是可以单独编写的，即使你没有写 `View` 层，你可以编写针对VM的Unit进行功能测试，确保VM无误后继续编写后续代码。

### V层
首先我们要通过RAC实现一部分UI的功能——输入文字的时候同步将文字保存起来&&控制按钮的禁用状态。

想想看我们通常是怎么做的？

1. 通过实现UITextField的代理
2. 在 `- (BOOL)textField:(UITextField *)textField shouldChangeCharactersInRange:(NSRange)range replacementString:(NSString *)string;` 方法中获取输入的文字，赋值给username属性和password属性
3. 再判断username和password是否符合要求
4. 再设置按钮的enabled属性

是不是看一看就觉得乱糟糟的，按钮的addTarget在一个地方，代理又在一个地方，再加上判断用户名密码合法逻辑单独抽出的方法。OMG。

ReactiveCocoa是怎么做的？

```objc
RAC(self.viewModel, userName) = self.tfUserName.rac_textSignal;
RAC(self.viewModel, password) = self.tfPassword.rac_textSignal;
RAC(self.btLogin, enabled) = [self.viewModel validSignal];
```

三句话，清清爽爽。

再加上 `ViewModel` 的信号绑定，将上面的代码放到一个方法中，命名为`bindModel`

最后的代码如下：

```objc
#import "ViewController.h"
#import "ReactiveCocoa.h"
#import "LoginViewModel.h"
#import "User.h"

@interface ViewController ()
@property (nonatomic, strong) LoginViewModel *viewModel;
@property (weak, nonatomic  ) IBOutlet UITextField    *tfUserName;
@property (weak, nonatomic  ) IBOutlet UITextField    *tfPassword;
@property (weak, nonatomic  ) IBOutlet UIButton       *btLogin;

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self bindModel];
}

/**
 *  绑定Model中的各种事件
 */
- (void)bindModel {
    self.viewModel = [[LoginViewModel alloc] init];
    RAC(self.viewModel, userName) = self.tfUserName.rac_textSignal;
    RAC(self.viewModel, password) = self.tfPassword.rac_textSignal;
    RAC(self.btLogin, enabled) = [self.viewModel validSignal];
    
//    @weakify(self);
    // 订阅登录成功信号并作出处理
    [self.viewModel.successSubject subscribeNext:^(NSArray * x) {
//        @strongify(self);
        User *user = x[0];
        NSLog(@"username:%@\tpassword:%@", user.username, user.password);
        NSLog(@"登陆成功");
    }];
    
    // 订阅登录失败信号并作出处理
    [self.viewModel.failureSubject subscribeNext:^(id x) {
        NSLog(@"登陆失败");
    }];
    
    // 订阅登录错误信号并作出处理
    [self.viewModel.errorSubject subscribeNext:^(id x) {
        NSLog(@"登陆错误");
    }];
    
    // 添加按钮点击事件
    [[self.btLogin rac_signalForControlEvents:UIControlEventTouchUpInside] subscribeNext:^(id x) {
        [self.viewModel login];
    }];
    
}

@end
```

这样，ViewController中事件处理的所有代码被集中在一起，方便管理，你的代码变得如此清爽、低耦合。

# 代码
以上代码地址请点[这里](http://download.csdn.net/detail/u010127917/9528911)。

# 补充
小鱼最近在换工作哟~各路的朋友有推荐的请务必介绍我哟~
简历在[这里](http://zhoulingyu.com/about/)

----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)


