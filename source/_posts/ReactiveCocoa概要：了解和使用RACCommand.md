title: ReactiveCocoa概要：了解和使用RACCommand
date: 2016-08-05 10:33:00
tags:
  - iOS
  - 译文
categories:
  - iOS
---

# 原文地址

[点击这里](http://codeblog.shape.dk/blog/2013/12/05/reactivecocoa-essentials-understanding-and-using-raccommand/)

这几天部门的前辈再用RAC的时候问到一个问题，RACCommand在RAC中具体的作用和起到的功能，到底应该如何应用它。

关于RAC的使用文章非常多，但是大多仅限于介绍和基本的使用方法，很少介绍RAC究竟应该如何优雅的嵌入到项目中。

在查阅资料的时候发现了此篇博文，写的非常细致，所以做了一次搬运工。

另，妹子我的英文属于渣渣系列，所以有什么翻译不当，请一定要指教。

<!--more-->

# Code
文章中所有代码在[这里](https://github.com/olegam/RACCommandExample)

# RACCommand是你的新伙伴吗？

`RACCommand`是ReactiveCocoa最精华的部分之一，它可以让你在开发中节约大量的时间并让你的iOS或者OS X app有更强的鲁棒性。

我见过不少刚接触ReactiveCocoa（后文将简写为RAC），还不能完全理解`RACCommand`是如何工作又不知何时应该使用`RACCommand`的同学。所以我认为这个小介绍将会市场实用，可以给他们带来一些启发。官方文档并没有给出多少如何使用RACCommand的Examples，但是[RACCommand头文件的介绍还是很不错的](https://github.com/ReactiveCocoa/ReactiveCocoa/blob/master/ReactiveCocoa/Objective-C/RACCommand.h)，不过这对刚开始用RAC的同学来说还是太难理解了。

`RACCommand`类是用于表示一些操作的执行。通常，是由于UI上的一些事件触发了`RACCommand`的执行。比如当用户按了一个按钮，如果对应`RACCommand`实例可以被执行，就会执行相应的操作。这使得它很容易和UI进行绑定，同时可以保证当`RACCommand`处于`not enabled`时`RACCommand`实例的操作不会被执行。当Command可以执行时，常做的方式是把allowsconcuuent的属性设置为NO，这可以保证Command已经执行完成后不会被重复执行。Command执行的结果是一个RACSignal，因此你可以调用`next:`、`completed:`、或者`error:`。后面将会展示具体使用方式。

# Example App
我们假设我们正在设计一个简单的app，其功能是让用户订阅一个邮件。最简单的方式是，用一个UITextField和一个UIButton。当用户输入email并且点击按钮的时候，email地址将会传给某个web服务。看起来很简单，但是我们应该确保用户有最好的体验。`如果用户按了两次按钮？``如何处理请求出错？``如果email不合法？``RACCommand`可以帮助我们处理这些情况。在这篇文章中将一步步完善这个小app以此来讨论一些概念和工作原理。

![](http://codeblog.shape.dk/images/raccommand_example_app_screenshot.png)

可以从[这里](https://github.com/olegam/RACCommandExample)获得源码。

从一个非常简单的ViewController可以很好的实践MVVM模式。

``` objc
- (void)bindWithViewModel {
  RAC(self.viewModel, email) = self.emailTextField.rac_textSignal;
  self.subscribeButton.rac_command = self.viewModel.subscribeCommand;
  RAC(self.statusLabel, text) = RACObserve(self.viewModel, statusMessage);
}
```

在上面的方法（在`viewDidLoad`中调用），在View和ViewModel中建立了绑定关系。下面是ViewModel的定义：

```objc

@interface SubscribeViewModel : NSObject

@property(nonatomic, strong) RACCommand *subscribeCommand;

// write to this property
@property(nonatomic, strong) NSString *email;

// read from this property
@property(nonatomic, strong) NSString *statusMessage;

@end
```

如上所示，一个暴露出的RACCommand属性。另外两个是字符串属性，它们和View的两个属性绑定在一起。ViewModel的完整实现如下:

```objc
#import "SubscribeViewModel.h"
#import "AFHTTPRequestOperationManager+RACSupport.h"
#import "NSString+EmailAdditions.h"

static NSString *const kSubscribeURL = @"http://reactivetest.apiary.io/subscribers";

@interface SubscribeViewModel ()
@property(nonatomic, strong) RACSignal *emailValidSignal;
@end

@implementation SubscribeViewModel

- (id)init {
  self = [super init];
  if (self) {
      [self mapSubscribeCommandStateToStatusMessage];
  }
  return self;
}

- (void)mapSubscribeCommandStateToStatusMessage {
  RACSignal *startedMessageSource = [self.subscribeCommand.executionSignals map:^id(RACSignal *subscribeSignal) {
      return NSLocalizedString(@"Sending request...", nil);
  }];

  RACSignal *completedMessageSource = [self.subscribeCommand.executionSignals flattenMap:^RACStream *(RACSignal *subscribeSignal) {
      return [[[subscribeSignal materialize] filter:^BOOL(RACEvent *event) {
          return event.eventType == RACEventTypeCompleted;
      }] map:^id(id value) {
          return NSLocalizedString(@"Thanks", nil);
      }];
  }];

  RACSignal *failedMessageSource = [[self.subscribeCommand.errors subscribeOn:[RACScheduler mainThreadScheduler]] map:^id(NSError *error) {
      return NSLocalizedString(@"Error :(", nil);
  }];

  RAC(self, statusMessage) = [RACSignal merge:@[startedMessageSource, completedMessageSource, failedMessageSource]];
}

- (RACCommand *)subscribeCommand {
  if (!_subscribeCommand) {
      NSString *email = self.email;
      _subscribeCommand = [[RACCommand alloc] initWithEnabled:self.emailValidSignal signalBlock:^RACSignal *(id input) {
          return [SubscribeViewModel postEmail:email];
      }];
  }
  return _subscribeCommand;
}

+ (RACSignal *)postEmail:(NSString *)email {
  AFHTTPRequestOperationManager *manager = [AFHTTPRequestOperationManager manager];
  manager.requestSerializer = [AFJSONRequestSerializer new];
  NSDictionary *body = @{@"email": email ?: @""};
  return [[[manager rac_POST:kSubscribeURL parameters:body] logError] replayLazily];
}

- (RACSignal *)emailValidSignal {
  if (!_emailValidSignal) {
      _emailValidSignal = [RACObserve(self, email) map:^id(NSString *email) {
          return @([email isValidEmail]);
      }];
  }
  return _emailValidSignal;
}

@end
```

这看起来真是一大坨~看还是从小的地方来看吧。我们真正感兴趣RACCommand`RACCommand`创建部分是以下代码：

```objc
- (RACCommand *)subscribeCommand {
  if (!_subscribeCommand) {
      NSString *email = self.email;
      _subscribeCommand = [[RACCommand alloc] initWithEnabled:self.emailValidSignal signalBlock:^RACSignal *(id input) {
          return [SubscribeViewModel postEmail:email];
      }];
  }
  return _subscribeCommand;
}
```
Command通过一个`enabledSignal`参数来初始化。这个Signal可以指示Command是否可以被执行。在我们本次的用例中Command应该在用户合法输入email时允许被执行。`self.emailValidSignal`就是用来在email发生变化发送`NO`或者`YES`指示的。

`signalBlock`参数在Command需要执行时被调用。block应该返回一个signal。当我们设置`allowsConcurrentExecution`为`NO`，Command将会看守这个signal并且在本次执行未完成前不允许任何新的执行。

由于本次用例中的Command来自于按钮的`rac_command`（在`UIButtton+RACCommandSupport`分类中定义），根据Command是否可以被执行，按钮会自动切换`enabled`和`disabled`状态。

当然，Command会在按钮被用户点击的时候自动执行。我们可以通过RACCommand自由的实现这一切。如果你需要手动执行你可以调用`-[RACCommand execute:]`，参数是可选的，你可以传递nil。我们的用例里不需要参数，不过这里的参数通常会十分有用（按钮可以将自己当做`-execute:`的参数传入）。`-execute:`方法也是一个你可以监控执行状态的地方，你可以这样写：

```objc
[[self.viewModel.subscribeCommand execute:nil] subscribeCompleted:^{
  NSLog(@"The command executed");
}];
```

在我们的用例中按钮为我们调用Command的执行（所以我们不需要手动调用`-execute:`），所以在Command执行时，为了及时更新UI，我们需要监听Command的另一个属性。我们有几种让人迷惑的地方，`RACCommand`的`executionSignals`属性是一个每当Commands开发执行时就发送`next: `的Signal。问题在于Signal由Command创建，所以Signal中还有一层Signal。每次Command开始执行的时候， 我们会在ViewModel中通过`mapSubscribeCommandStateToStatusMessage`方法里面获取到一个信号。同时在这个信号里面返回了一个字符串：

```objc
RACSignal *startedMessageSource = [self.subscribeCommand.executionSignals map:^id(RACSignal *subscribeSignal) {
  return NSLocalizedString(@"Sending request...", nil);
}];
```

假如我们想用更函数的方式，来在Command执行完成后都能获取string，我们需要做更多的工作：

```objc
RACSignal *completedMessageSource = [self.subscribeCommand.executionSignals flattenMap:^RACStream *(RACSignal *subscribeSignal) {
  return [[[subscribeSignal materialize] filter:^BOOL(RACEvent *event) {
      return event.eventType == RACEventTypeCompleted;
  }] map:^id(id value) {
      return NSLocalizedString(@"Thanks", nil);
  }];
}];
```

当Command执行时，`flattenMap:`方法调用一个带`subscribeSignal`参数的block。这个block返回一个新的Signal并且它的值会被传递到下一个返回信号。`materialize`操作符让我们捕获到一个`RACEvent`（例如 `next:` `complete` 和 `error:`都是RACEvent的实例）。我们可以在信号完成之后过滤这些`event`并且映射成一个string。这些解释让你晕了吗，不过你可以去看一下[flattenMap:](https://github.com/ReactiveCocoa/ReactiveCocoa/blob/d4924b237720afef62c1334437140bb803fb5242/ReactiveCocoaFramework/ReactiveCocoa/RACStream.h#L98)和[materialize](https://github.com/ReactiveCocoa/ReactiveCocoa/blob/2d5f163547f2b9c6ee25cf2c9ff8554faf7929f2/ReactiveCocoaFramework/ReactiveCocoa/RACSignal%2BOperations.h#L558)的文档以助于你的理解。

我们可以用另一种不同但更容易理解的方式来实现：

```objc
@weakify(self);
[self.subscribeCommand.executionSignals subscribeNext:^(RACSignal *subscribeSignal) {
  [subscribeSignal subscribeCompleted:^{
      @strongify(self);
      self.statusMessage = @"Thanks";
  }];
}];
```

但是我并不喜欢上面的写法，因为这样会block中的操作会更多并且会更多的在block中使用到`self`。所以在这里还使用了`@weakify`和`@strongify`（在`libextobjc`中定义）避免循环retain。

关于`executionSignals`属性，有一个重要的细节。在这里的Signal所发送的event不包含`error`，所以对于那些有特殊`errors`属性

```objc
RACSignal *failedMessageSource = [[self.subscribeCommand.errors subscribeOn:[RACScheduler mainThreadScheduler]] map:^id(NSError *error) {
  return NSLocalizedString(@"Error :(", nil);
}];
```

如果我们有三个带有状态消息的Signal，我们可以将他们合并陈搞一个信号并绑定到ViewModel的一个`statusMessage`属性 (`statusMessage`绑定ViewController的statusLabel.text)。

```objc
RAC(self, statusMessage) = [RACSignal merge:@[startedMessageSource, completedMessageSource, failedMessageSource]];
```

那么以上是一个`RACCommand`在iOS app 开发中的一个example。我相信这种实现逻辑比使用`UITextFieldDelegate`有更多的优点，能在属性和变量中体现更多的状态。


# 其他有趣的RACCommand使用细节

`RACCommand`有一个`executing`属性，实际上它是一个当`execute:`时会发送`YES`，终止时发送`NO`的信号。在订阅信号时这个信号将会发送它的当前值，如果你只需要获取当前值而不需要获得信号，你可以通过以下方式：

```objc
BOOL commandIsExecuting = [[command.executing first] boolValue];
```

`enabled`属性也是一个发送`YES`和`NO`的信号。当Command通过发送`NO`的`enabledSignal`信号创建，或者如果信号在执行并且`allowsConcurrentExecutions` 为 `NO`，`enabled`就会发送`NO`。

`-execute:`方法会自动订阅原始Signal并且广播它。这意味着你不需要去订阅`-execute:`返回的信号，但是如果你订阅了也不需要担心它会被执行两次。

----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)

