title: iOS进阶——iOS（Objective-C） 内存管理&Block
date: 2017-02-08 15:33:40
tags:
	- iOS
	- iOS进阶
categories:
	- iOS
	- iOS进阶
---

# 第一篇 iOS 内存管理

## 1 似乎每个人在学习 iOS 过程中都考虑过的问题

1. alloc retain release delloc 做了什么？
2. autoreleasepool 是怎样实现的？
3. __unsafe_unretained 是什么？
4. Block 是怎样实现的
5. 什么时候会引起循环引用，什么时候不会引起循环引用？

所以我将在本篇博文中详细的从 ARC 解释到 iOS 的内存管理，以及 Block 相关的原理、源码。

## 2 从 ARC 说起

说 iOS 的内存管理，就不得不从 ARC（Automatic Reference Counting / 自动引用计数） 说起， ARC 是 WWDC2011 和 iOS5 引入的变化。ARC 是 LLVM 3.0 编译器的特性，用来自动管理内存。

与 Java 中 GC 不同，ARC 是编译器特性，而不是基于运行时的，所以 ARC 其实是在编译阶段自动帮开发者插入了管理内存的代码，而不是实时监控与回收内存。

<!-- More -->

![ARC 管理内存](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E8%BF%9B%E9%98%B6%E2%80%94%E2%80%94iOS%20%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86&Block-01.png)

ARC 的内存管理规则可以简述为：

> 1. 每个对象都有一个『被引用计数』
> 2. 对象被持有，『被引用计数』+1
> 3. 对象被放弃持有，『被引用计数』-1
> 4. 『引用计数』=0，释放对象

## 3 你需要知道

~~1. 包含 NSObject 类的 Foundation 框架并没有公开~~
（此处错误，感谢 [酷酷的哀殿](http://www.jianshu.com/u/486bf26e8dce) 的指出）

1. Foundation 框架是非开源的，但是 NSObject 被包含在 [obj4](https://opensource.apple.com/source/objc4/objc4-706/runtime/NSObject.mm) 中，该库已开源。
2. Core Foundation 框架源代码，以及通过 NSObject 进行内存管理的部分源代码是公开的。
3. GNUstep 是 Foundation 框架的互换框架

> GNUstep 也是 GNU 计划之一。将 Cocoa Objective-C 软件库以自由软件方式重新实现
> 某种意义上，GNUstep 和 Foundation 框架的实现是相似的
> 通过 GNUstep 的源码来分析 Foundation 的内存管理

## 4 alloc retain release dealloc 的实现

### 4.1 GNU - alloc

查看 GNUStep 中的 alloc 函数。

**GNUstep/modules/core/base/Source/NSObject.m alloc:**

```objc
+ (id) alloc
{
  return [self allocWithZone: NSDefaultMallocZone()];
}

+ (id) allocWithZone: (NSZone*)z
{
  return NSAllocateObject (self, 0, z);
}
```

**GNUstep/modules/core/base/Source/NSObject.m NSAllocateObject:**

```objc
struct obj_layout {
    NSUInteger retained;
};

NSAllocateObject(Class aClass, NSUInteger extraBytes, NSZone *zone)
{
    int	size = 计算容纳对象所需内存大小;
    id	new = NSZoneCalloc(zone, 1, size);
    memset (new, 0, size);
    new = (id)&((obj)new)[1];
}
```

`NSAllocateObject` 函数通过调用 `NSZoneCalloc` 函数来分配存放对象所需的空间，之后将该内存空间置为 nil，最后返回作为对象而使用的指针。

我们将上面的代码做简化整理：

**GNUstep/modules/core/base/Source/NSObject.m alloc 简化版本:**

```objc
struct obj_layout {
    NSUInteger retained;
};

+ (id) alloc
{
    int size = sizeof(struct obj_layout) + 对象大小;
    struct obj_layout *p = (struct obj_layout *)calloc(1, size);
    return (id)(p+1)
    return [self allocWithZone: NSDefaultMallocZone()];
}
```

alloc 类方法用 struct obj_layout 中的 `retained` 整数来保存引用计数，并将其写入对象的内存头部，该对象内存块全部置为 0 后返回。

一个对象的表示便如下图：

![GNU 中的对象存储空间](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E8%BF%9B%E9%98%B6%E2%80%94%E2%80%94iOS%20%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86&Block-02.png)

### 4.2 GNU - retain

**GNUstep/modules/core/base/Source/NSObject.m retainCount:**

```objc
- (NSUInteger) retainCount
{
  return NSExtraRefCount(self) + 1;
}

inline NSUInteger
NSExtraRefCount(id anObject)
{
  return ((obj_layout)anObject)[-1].retained;
}
```

**GNUstep/modules/core/base/Source/NSObject.m retain:**

```objc
- (id) retain
{
  NSIncrementExtraRefCount(self);
  return self;
}

inline void
NSIncrementExtraRefCount(id anObject)
{
  if (((obj)anObject)[-1].retained == UINT_MAX - 1)
    [NSException raise: NSInternalInconsistencyException
      format: @"NSIncrementExtraRefCount() asked to increment too far”];
  ((obj_layout)anObject)[-1].retained++;
}

```

以上代码中， `NSIncrementExtraRefCount` 方法首先写入了当 `retained` 变量超出最大值时发生异常的代码（因为 `retained` 是 NSUInteger 变量），然后进行 `retain ++` 代码。

### 4.3 GNU - release

和 retain 相应的，release 方法做的就是 `retain --`。

**GNUstep/modules/core/base/Source/NSObject.m release**

```objc
- (oneway void) release
{
  if (NSDecrementExtraRefCountWasZero(self))
    {
      [self dealloc];
    }
}

BOOL
NSDecrementExtraRefCountWasZero(id anObject)
{
  if (((obj)anObject)[-1].retained == 0)
  {
	  return YES;
	}
  ((obj)anObject)[-1].retained--;
	return NO;
}
```

### 4.4 GNU - dealloc

dealloc 将会对对象进行释放。

**GNUstep/modules/core/base/Source/NSObject.m dealloc:**

```objc
- (void) dealloc
{
  NSDeallocateObject (self);
}

inline void
NSDeallocateObject(id anObject)
{
  obj_layout o = &((obj_layout)anObject)[-1];
  free(o);
}
```

### 4.5 Apple 实现

在 Xcode 中 设置 `Debug` -> `Debug Workflow` -> `Always Show Disassenbly` 打开。这样在打断点后，可以看到更详细的方法调用。

通过在 NSObject 类的 alloc 等方法上设置断点追踪可以看到几个方法内部分别调用了：

**retainCount**

> __CFdoExternRefOperation
> CFBasicHashGetCountOfKey

**retain**

> __CFdoExternRefOperation
> CFBasicHashAddValue

**release**

> __CFdoExternRefOperation
> CFBasicHashRemoveValue

可以看到他们都调用了一个共同的 `__CFdoExternRefOperation` 方法。

该方法从前缀可以看到是包含在 Core Foundation，在 CFRuntime.c 中可以找到，做简化后列出源码：

**CFRuntime.c __CFDoExternRefOperation:**

```objc
int __CFDoExternRefOperation(uintptr_t op, id obj) {
    CFBasicHashRef table = 取得对象的散列表(obj);
    int count;
    
    switch (op) {
        case OPERATION_retainCount:
        count = CFBasicHashGetCountOfKey(table, obj);
        return count;
        break;
        case OPERATION_retain:
        count = CFBasicHashAddValue(table, obj);
        return obj;
        case OPERATION_release:
        count = CFBasicHashRemoveValue(table, obj);
        return 0 == count;
    }
}
```

所以 `__CFDoExternRefOperation` 是针对不同的操作，进行具体的方法调用，如果 op 是 `OPERATION_retain`，就去掉用具体实现 retain 的方法。

从 `BasicHash` 这样的方法名可以看出，其实引用计数表就是散列表。

key 为 hash(对象的地址) value 为 引用计数。

下图是 Apple 和 GNU 的实现对比：

![Apple 和 GNU 内存管理的实现对比](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E8%BF%9B%E9%98%B6%E2%80%94%E2%80%94iOS%20%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86&Block-03.png)

## 5 autorelease 和 autorelaesepool

在苹果对于 NSAutoreleasePool 的[文档](https://developer.apple.com/reference/foundation/nsautoreleasepool)中表示：

> 每个线程（包括主线程），都维护了一个管理 NSAutoreleasePool 的栈。当创先新的 Pool 时，他们会被添加到栈顶。当 Pool 被销毁时，他们会被从栈中移除。
> autorelease 的对象会被添加到当前线程的栈顶的 Pool 中。当 Pool 被销毁，其中的对象也会被释放。
> 当线程结束时，所有的 Pool 被销毁释放。

对 NSAutoreleasePool 类方法和 autorelease 方法打断点，查看其运行过程，可以看到调用了以下函数：

```objc
NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];
// 等同于 objc_autoreleasePoolPush
    
id obj = [[NSObject alloc] init];
[obj autorelease];
//  等同于 objc_autorelease(obj)
    
[NSAutoreleasePool showPools];
// 查看 NSAutoreleasePool 状况
    
[pool drain];
// 等同于 objc_autoreleasePoolPop(pool)
```

`[NSAutoreleasePool showPools]` 可以看到当前线程所有 pool 的情况：

```
objc[21536]: ##############
objc[21536]: AUTORELEASE POOLS for thread 0x10011e3c0
objc[21536]: 2 releases pending.
objc[21536]: [0x101802000]  ................  PAGE  (hot) (cold)
objc[21536]: [0x101802038]  ################  POOL 0x101802038
objc[21536]: [0x101802040]       0x1003062e0  NSObject
objc[21536]: ##############
Program ended with exit code: 0
```

在 [objc4](https://github.com/opensource-apple/objc4) 中可以查看到 AutoreleasePoolPage：

```objc
objc4/NSObject.mm AutoreleasePoolPage

class AutoreleasePoolPage 
{
    static inline void *push() 
    {
        生成或者持有 NSAutoreleasePool 类对象
    }
    static inline void pop(void *token) 
    {
        废弃 NSAutoreleasePool 类对象
        releaseAll();
    }
    static inline id autorelease(id obj)
    {
        相当于 NSAutoreleasePool 类的 addObject 类方法
        AutoreleasePoolPage *page = 取得正在使用的 AutoreleasePoolPage 实例;
    }
    id *add(id obj)
    {
        将对象追加到内部数组
    }
    void releaseAll() 
    {
        调用内部数组中对象的 release 方法
    }
};

void *
objc_autoreleasePoolPush(void)
{
    if (UseGC) return nil;
    return AutoreleasePoolPage::push();
}

void
objc_autoreleasePoolPop(void *ctxt)
{
    if (UseGC) return;
    AutoreleasePoolPage::pop(ctxt);
}
```

AutoreleasePoolPage 以双向链表的形式组合而成（分别对应结构中的 parent 指针和 child 指针）。
thread 指针指向当前线程。
每个 AutoreleasePoolPage 对象会开辟4096字节内存（也就是虚拟内存一页的大小），除了上面的实例变量所占空间，剩下的空间全部用来储存autorelease对象的地址。
next 指针指向下一个 add 进来的 autorelease 的对象即将存放的位置。
一个 Page 的空间被占满时，会新建一个 AutoreleasePoolPage 对象，连接链表。

![AutoreleasePoolPage](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E8%BF%9B%E9%98%B6%E2%80%94%E2%80%94iOS%20%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86&Block-04.png)

## 6 __unsafe_unretained

有时候我们除了 `__weak` 和 `__strong` 之外也会用到 `__unsafe_unretained` 这个修饰符，那么我们对 `__unsafe_unretained` 了解多少？

`__unsafe_unretained` 是不安全的所有权修饰符，尽管 ARC 的内存管理是编译器的工作，但附有 `__unsafe_unretained` 修饰符的变量不属于编译器的内存管理对象。**赋值时即不获得强引用也不获得弱引用**。

来运行一段代码：

```objc
id __unsafe_unretained obj1 = nil;
{
    id __strong obj0 = [[NSObject alloc] init];
            
    obj1 = obj0;
            
    NSLog(@"A: %@", obj1);
}
        
NSLog(@"B: %@", obj1);
```

运行结果：

```
2017-01-12 19:24:47.245220 __unsafe_unretained[55726:4408416] A: <NSObject: 0x100304800>
2017-01-12 19:24:47.246670 __unsafe_unretained[55726:4408416] B: <NSObject: 0x100304800>
Program ended with exit code: 0
```

对代码进行详细分析：

```objc
id __unsafe_unretained obj1 = nil;
{
    // 自己生成并持有对象
    id __strong obj0 = [[NSObject alloc] init];
            
    // 因为 obj0 变量为强引用，
    // 所以自己持有对象
    obj1 = obj0;
            
    // 虽然 obj0 变量赋值给 obj1
    // 但是 obj1 变量既不持有对象的强引用，也不持有对象的弱引用
    NSLog(@"A: %@", obj1);
    // 输出 obj1 变量所表示的对象
}
        
    NSLog(@"B: %@", obj1);
    // 输出 obj1 变量所表示的对象
    // obj1 变量表示的对象已经被废弃
    // 所以此时获得的是悬垂指针
    // 错误访问
```

所以，最后的 NSLog 只是碰巧正常运行，如果错误访问，会造成 crash
在使用 `__unsafe_unretained` 修饰符时，赋值给附有 `__strong` 修饰符变量时，要确保对象确实存在

# 第二篇 Block

花几分钟时间看下面三个小题目，写下你的答案。

![Block 的三道测试题](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E8%BF%9B%E9%98%B6%E2%80%94%E2%80%94iOS%20%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86&Block-05.png)

这个三个小题目，我在整理此片博文之前给了三位朋友去解答，最后的结果，除了一位朋友 3 题全部正确，其他两个朋友均只答中 1 题。

说明还是有很多 iOS 的朋友对于 Block 并没有透彻理解。本篇博文会对 Block 进行详细的解说。

## 1 Block 使用的简单规则

先了解简单规则，再去分析原理和实现：

> Block 中，Block **表达式截获**所使用的自动变量的值，即保存该自动变量的**瞬间值**。
> 修饰为 `__block` 的变量，在捕获时，获取的**不再是瞬间值**。

至于 Why，后面将会继续说。

## 2 Block 的实现

Block 是带有自动变量（局部变量）的匿名函数。
Block 表达式很简单，总体可以描述为：『`^ 返回值类型 参数列表 表达式`』。
但是 Block 并不是 Objective-C 中才有的语法，这是怎么一回事？

clang 编译器提供给程序员了解 Objective-C 背后机制的方法，通过 clang 的转换可以看到 Block 的实现原理。

通过 `clang -rewrite-objc yourfile.m` clang 将会把 Objective-C 的代码转换成 C 语言的代码。

### 2.1 Block 基本实现剖析

用 Xcode 创建 Command Line 项目，写如下代码：

```objc
int main(int argc, const char * argv[]) {
    void (^blk)(void) = ^{NSLog(@"Block")};
    blk();
    return 0;
}
```

用 clang 转换：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E8%BF%9B%E9%98%B6%E2%80%94%E2%80%94iOS%20%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86&Block-06.png)

以上是转换后的代码，不要方，一段一段看。

可以看到，Block 的实现内容，**被转换成了一个普通的静态函数 `__main_func_0`**。

再看其他部分：

**main.cpp __block_impl:**

```c
struct __block_impl {
    void *isa;
    int Flags;
    int Reserved;
    void *FuncPtr;
};
```

`__block_impl` 结构体包括了一些标志、今后版本升级**预留的变量**、**函数指针**。

----

**main.cpp __main_block_desc_0:**

```c
static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
```

`__main_block_desc_0` 结构体包括了今后版本升级预留的变量、block 大小。

----

**main.cpp __main_block_impl_0:**

```c
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;

  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

`__main_block_impl_0` 结构体含有两个成员变量，分别是 `__block_impl` 和 `__main_block_desc_0 `实例变量。

此外，还含有一个构造方法。该构造方法在 main 函数中被如下调用：

**main.cpp __main_block_impl_0 构造函数的调用:**

```c
void (*blk)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0,
                                              &__main_block_desc_0_DATA));
```

去掉各种强制转换，做简化：

**main.cpp __main_block_impl_0 构造函数的调用 简化:**

```c
struct __main_block_impl_0 tmp = __main_block_impl_0(__main_block_func_0, &__main_block_desc_0_DATA);
struct __main_block_impl_0 *blk = &tmp;
```

以上代码即：将 `__main_block_impl_0` 结构体实例的指针，赋值给 `__main_block_impl_0` 结构体指针类型的变量 `blk`。也就是我们最初的结构体定义：

```objc
 void (^blk)(void) = ^{NSLog(@"Block");};
```

另外，main 函数中还有另外一段：

```c
((void (*)(__block_impl *))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk);
```

去掉各种转换：

```c
(*blk->impl.FuncPtr)(blk);
```

实际就是最初的：

```objc
blk();
```

> 本节所有代码在 [block_implementation](https://github.com/summertian4/iOS-ObjectiveC/tree/master/ObjcMemory/ObjcMemory-Test-Code/block_implementation) 中

### 2.2 Block 截获外部变量瞬间值的实现剖析

2.1 中对最简单的 _无参数 Block 声明、调用_ 进行了 clang 转换。接下来再看一段『截获自动变量』的代码(可以使用命令 `clang -rewrite-objc -fobjc-arc -fobjc-runtime=macosx-10.7 main.m`)：

```objc
int main(int argc, const char * argv[]) {
    
    int val = 10;
    const char *fmt = "val = %d\n";
    void (^blk)(void) = ^{printf(fmt, val);};
    
    val = 2;
    fmt = "These values were changed, val = %d\n";
    
    blk();
    
    return 0;
}
```

clang 转换之后：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E8%BF%9B%E9%98%B6%E2%80%94%E2%80%94iOS%20%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86&Block-07.png)

和 2.1 节中的转换代码对比，可以发现多了一些代码。

首先，`__main_block_impl_0` 多了一个变量 `val`，并在构造函数的参数中加入了 `val` 的赋值：

**main.cpp __main_block_impl_0:**

```objc
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  const char *fmt;
  int val;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, const char *_fmt, int _val, int flags=0) : fmt(_fmt), val(_val) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

而在 main 函数中，对 Block 的声明变为此句：

**main.cpp __main_block_impl_0 构造函数的调用:**

```objc 
void (*blk)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, fmt, val));
```

去掉转换：

**main.cpp __main_block_impl_0 构造函数的调用 简化:**

```objc
struct __main_block_impl_0 tmp = __main_block_impl_0(__main_block_func_0, &__main_block_desc_0_DATA, val);
    struct __main_block_impl_0 *blk = &tmp;
```

_**所以，在 Block 被声明时，Block 已经将 `val` 作为 `__main_block_impl_0` 的内部变量保存下来了。无论在在声明之后怎样更改 val 的值，都不会影响，Block 调用时访问的内部 val 值。这就是 Block 捕获变量瞬间值的原理。**_

> 本节所有代码在 [EX05](https://github.com/summertian4/iOS-ObjectiveC/tree/master/ObjcMemory/ObjcMemory-Test-Code/EX05) 中

### 2.3 __block 变量的访问实现剖析

我们知道，Block 中能够读取，但是不能更改一个局部变量，如果去更改，Xcode 会提示你无法在 Block 内部更改变量。

Block 内部只是对局部变量只读，但是 Block 能读写以下几种变量：

1. 静态变量
2. 静态全局变量
3. 全局变量

也就是说以下代码是没有问题的：

```
int global_val = 1;
static int static_global_val = 2;

int main(int argc, const char * argv[]) {
    static int static_val = 3;
    
    void (^blk)(void) = ^ {
        global_val = 1 * 2;
        static_global_val = 2 * 2;
        static_val = 3 * 2;
    }
    
    return 0;
}
```

如果想在 Block 内部写局部变量，需要对访问的局部变量增加 __block 修饰。

__block 修饰符其实类似于 C 语言中 static、auto、register 修饰符。用于指定将变量值设置到哪个存储域中。

具体 __block 之后究竟做了哪些变化我们可以写代码测试：

**EX07:**

```
int main(int argc, const char * argv[]) {
    
    __block int val = 10;
    void (^blk)(void) = ^{val = 1;};
    
    return 0;
}
```


clang 转换之后：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_blog_iOS%E8%BF%9B%E9%98%B6%E2%80%94%E2%80%94iOS%20%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86&Block-08.png)

跟 2.2 对比，似乎又加了非常代码。发现多了两个结构体。

**main.cpp __Block_byref_val_0:**

```objc
struct __Block_byref_val_0 {
  void *__isa;
__Block_byref_val_0 *__forwarding;
 int __flags;
 int __size;
 int val;
};
```

很惊奇的发现，__block 类型的 `val` 变成了结构体 `__Block_byref_val_0` 的实例。这个实例内，包含了 `__isa` 指针、一个标志位 `__flags` 、一个记录大小的 `__size` 。最最重要的，多了一个 `__forwarding` 指针和 `val` 变量。这是怎么回事？

在 main 函数部分，实例化了该结构体：

**main.cpp main.m 部分:**

```
__Block_byref_val_0 val = {(void*)0,
                            (__Block_byref_val_0 *)&val,
                            0,
                            sizeof(__Block_byref_val_0),
                            10};
```

我们可以看出该结构体对象初始化时：

1. **__forwarding 指向了结构体实例本身在内存中的地址**
2. val = 10

而在 main 函数中，`val = 1` 这句赋值语句变成了：

**main.cpp `val = 1;` 对应的函数:**

```
(val->__forwarding->val) = 1;
```

这里就可以看出其精髓，val = 1，实际上更改的是 `__Block_byref_val_0` 结构体实例 val 中的 `__forwarding` 指针（也就是本身）指向的 `val` 变量。

![__Block_byref_val_0 实例示意图](http://7xt4xp.com1.z0.glb.clouddn.com/blog_blog_iOS%E8%BF%9B%E9%98%B6%E2%80%94%E2%80%94iOS%20%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86&Block-12.png)

而对 `val` 访问也是如此。你可以理解为通过取地址改变变量的值，这和 C 语言中取地址改变变量类似。

所以，声明 __block 的变量可以被改变。至于 `__forwarding` 的其他巨大作用，会继续分析。

> 本节代码在 [EX05](https://github.com/summertian4/iOS-ObjectiveC/tree/master/ObjcMemory/ObjcMemory-Test-Code/EX07) 中

## 3 Block 的存储域

Block 有三种类型，分别是：

> 1. __NSConcreteStackBlock         ————————栈中
> 2. __NSConcreteGlobalBlock        ————————数据区域中
> 3. __NSConcreteMallocBlock        ————————堆中

**__NSConcreteGlobalBlock 出现的地方有：**

1. 设置全局变量的地方有 Block 语法时
2. Block 语法的表达式中不使用任何外部变量时

设置在栈上的 Block，如果所属的变量作用域结束，Block 就会被废弃。如果其中用到了 __block，__block 所属的变量作用域结束也会被废弃。

为了解决这个问题，Block 在必要的时候就需要从栈中移到堆中。ARC 有效时，很多情况下，编译器会帮助完成 Block 的 copy，但很多情况下，我们需要手动 copy Block。

对不同存储域的 Block copy 时，影响如下：

![对不同存储域的 Block copy 影响](http://7xt4xp.com1.z0.glb.clouddn.com/blog_blog_iOS%E8%BF%9B%E9%98%B6%E2%80%94%E2%80%94iOS%20%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86&Block-09.png)

copy 时，对访问到的 __block 类型对象影响如下：

![Block copy 时对 __block 对象的影响](http://7xt4xp.com1.z0.glb.clouddn.com/blog_blog_iOS%E8%BF%9B%E9%98%B6%E2%80%94%E2%80%94iOS%20%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86&Block-10.png)

> 此时可以看出 `__forwarding` 的巨大作用——无论 Block 此时在堆中还是在栈中，由于 `__forwarding` 指向局部变量转换成的结构体实例的真是地址，所以都能确保正确的访问。

具体的来说：

1. 当 __block 变量被一个 Block 使用时，Block 从栈复制到堆，__block 变量也会被复制到，并被该 Block 持有。
2. 在 __block 变量被多个 Block 使用时，在任何一个 Block 从栈复制到堆时， __block 变量也会被复制到堆，并被该 Block 持有。但由于 `__forwarding` 指针的存在，无论 __block 变量和 Block 在不在同一个存储域，都可以正确的访问 __block 变量。
3. 如果堆上的 Block 被废弃，那么它所使用的 __block 变量也会被释放。

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_blog_iOS%E8%BF%9B%E9%98%B6%E2%80%94%E2%80%94iOS%20%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86&Block-11.png)

前面说到编译器会帮助完成一些 Block 的 copy，也有手动 copy Block。那么 Block 被复制到堆上的情况有（此段摘自于『Objective-C高级编程 iOS与OS X多线程和内存管理』）：

1. 调用 Block 的 copy 方法时
2. Block 作为返回值时
3. 将 Block 赋值给附有 `__strong` 修饰符的成员变量时（id类型或 Block 类型）时
4. 在方法名中含有 `usingBlock` 的 Cocoa 框架方法或 GCD 的 API 中传递 Block 时

## 4 Block 循环引用

Block 循环引用，是在编程中非常常见的问题，甚至很多时候，我们并不知道发生了循环引用，直到我们突然某一天发现『怎么这个对象没有调用 delloc』，才意识到有问题存在。

在『Block 存储域』中也说明了 Block 在 copy 后对 __block 对象会 retain 一次。

那么对于如下情况就会发生循环引用：

**block_retain_cycle:**

```objc
@interface MyObject : NSObject

@property (nonatomic, copy) blk_t blk;
@property (nonatomic, strong) NSObject *obj;

@end

@implementation MyObject

- (instancetype)init {
    self = [super init];
    _blk = ^{NSLog(@"self = %@", self);};
    return self;
}

- (void)dealloc {
    NSLog(@"%@ dealloc", self.class);
}

@end

int main(int argc, const char * argv[]) {
    id myobj = [[MyObject alloc] init];
    NSLog(@"%@", myobj);
    return 0;
}
```

由于 self -> blk，blk -> self，双方都无法释放。

但要注意的是，对于以下情况，同样会发生循环引用：

```objc
block_retain_cycle

@interface MyObject : NSObject

@property (nonatomic, copy) blk_t blk;

// 下面是多加的一句
@property (nonatomic, strong) NSObject *obj;

@end

@implementation MyObject

- (instancetype)init {
    self = [super init];
    
    // 下面是多加的一句
    _blk = ^{NSLog(@"self = %@", _obj);};
    
    return self;
}

- (void)dealloc {
    NSLog(@"%@ dealloc", self.class);
}

@end

int main(int argc, const char * argv[]) {
    id myobj = [[MyObject alloc] init];
    NSLog(@"%@", myobj);
    return 0;
}
```

这是由于 self -> obj，self -> blk，blk -> obj。这种情况是非常容易被忽视的。

## 5 重审问题

我们再来看看最初的几个小题目：

![Block 的三道测试题](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E8%BF%9B%E9%98%B6%E2%80%94%E2%80%94iOS%20%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86&Block-05.png)

1.  第一题：

    由于 Block 捕获瞬间值，所以输出为 `in block val = 0`

2.  第二题：

    由于 `val` 为 __block，外部更改会影响到内部访问，所以输出为 `in block val = 1`

3.  第三题：

    和第二题类似，`val = 1` 能影响到 Block 内部访问，所以先输出 `in block val = 1`，之后在 	Block 内部更改 `val` 值，再次访问时输出 `after block val = 2`。

# Other

我写这篇文章是在我阅读了『Objective-C高级编程 iOS与OS X多线程和内存管理』一书之后，博文中也有很内容源于『Objective-C高级编程 iOS与OS X多线程和内存管理』。

非常向大家推荐此书。这本书里记录了关于 iOS 内存管理的深入内容。但要注意的是，此书中的多处知识点并不是很详细，需要你以拓展的心态去学习。在有解释不详细的地方，自己主动去探索，去拓展，找更多的资料，最后，你会发现你对 iOS 内存管理有了更多的深入的理解。

对于文章中的测试代码，全部在[这里](https://github.com/summertian4/iOS-ObjectiveC/tree/master/ObjcMemory)。

----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)

