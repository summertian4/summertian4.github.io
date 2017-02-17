title: iOS进阶——iOS（Objective-C）内存管理·二
date: 2017-02-15 11:59:33
tags:
	- iOS
	- iOS进阶
categories:
	- iOS
	- iOS进阶
---

在写 『[iOS（Objective-C） 内存管理&Block](http://zhoulingyu.com/2017/02/08/iOS进阶——iOS-Memory-Block/)』 一文时，我并没有发现 NSObject 的代码已经被开源了，所以分析的主要是 GNUStep 的源码，对 Apple 的部分只是通过猜测。

实质上，NSObject 的实现内容已经开源在 [objc4-706](https://opensource.apple.com/tarballs/objc4/) 中。于是我便开始学习 objc4 中的内容。

下面就和大家扒一扒 Apple 的 NSObject 内存管理的一些内容。

# SideTable

找到 NSObject.mm，首先来一些非常重要的信息，以便后面的理解。

**objc4-706 NSObject.mm SideTable:**

```objc
struct SideTable {
    // 保证原子操作的自旋锁
    spinlock_t slock;
    // 引用计数的 hash 表
    RefcountMap refcnts;
    // weak 引用全局 hash 表
    weak_table_t weak_table;
};
```

<!-- More -->

SideTable 结构体重定了几个非常重要的变量。

```objc
// The order of these bits is important.
#define SIDE_TABLE_WEAKLY_REFERENCED (1UL<<0)
#define SIDE_TABLE_DEALLOCATING      (1UL<<1)  // MSB-ward of weak bit
#define SIDE_TABLE_RC_ONE            (1UL<<2)  // MSB-ward of deallocating bit
#define SIDE_TABLE_RC_PINNED         (1UL<<(WORD_BITS-1))

#define SIDE_TABLE_RC_SHIFT 2
#define SIDE_TABLE_FLAG_MASK (SIDE_TABLE_RC_ONE-1)
```

以上定义的是几个重要偏移量。引用计数 retainCount 是保存在一个无符号整形中，也就是有 8 个字节。其结构可以用下图表示：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_Advanced-iOS-Study-objc-Memory-2-01.png)

1. `SIDE_TABLE_WEAKLY_REFERENCED (1UL<<0)`（表示对象所在内存的第 1 位），标识该对象是否有过 weak 对象；
2. `SIDE_TABLE_DEALLOCATING (1UL<<1) `（表示对象所在内存的第 2 位），标识该对象是否正在 dealloc（析构）。
3. `SIDE_TABLE_RC_ONE (1UL<<2)` （表示对象所在内存的第 3 位），存放引用计数数值（其实第三位之后都用来存放引用计数数值）。

# retainCount

找到 retainCount 的实现，一层一层向下看。

**objc4 NSObject.mm retainCount:**

```objc
- (NSUInteger)retainCount {
    return ((id)self)->rootRetainCount();
}
```

**objc4 objc-object.h rootRetainCount:**

```objc
inline uintptr_t 
objc_object::rootRetainCount()
{
    if (isTaggedPointer()) return (uintptr_t)this;
    return sidetable_retainCount();
}
```

**objc4 NSObject.mm sidetable_retainCount:**

```objc
uintptr_t
objc_object::sidetable_retainCount()
{
    SideTable& table = SideTables()[this];

    size_t refcnt_result = 1;
    
    table.lock();
    RefcountMap::iterator it = table.refcnts.find(this);
    if (it != table.refcnts.end()) {
        // this is valid for SIDE_TABLE_RC_PINNED too
        refcnt_result += it->second >> SIDE_TABLE_RC_SHIFT;
    }
    table.unlock();
    return refcnt_result;
}
```

it->second 指向的就是存放引用计数相关的那个 8 位的无符号整型。

上面介绍过 Sidetable 中的几个重要偏移量，通过位移 SIDE_TABLE_RC_SHIFT 可以获取真实的引用计数。

所以，`sidetable_retainCount()` 中的主要内容就是遍历引用计数表，查找对象获取引用计数 +1 并将结果返回。

# retain

找到 retain 的实现。

**objc4 NSObject.mm retain:**

```objc
- (id)retain {
    return ((id)self)->rootRetain();
}
```

**objc4 objc-objc.h rootRetain:**

```objc
// Base retain implementation, ignoring overrides.
// This does not check isa.fast_rr; if there is an RR override then 
// it was already called and it chose to call [super retain].
inline id 
objc_object::rootRetain()
{
    if (isTaggedPointer()) return (id)this;
    return sidetable_retain();
}
```

**objc4 NSObject.mm sidetable_retain:**

```objc
id
objc_object::sidetable_retain()
{
#if SUPPORT_NONPOINTER_ISA
    assert(!isa.nonpointer);
#endif
    SideTable& table = SideTables()[this];
    
    table.lock();
    size_t& refcntStorage = table.refcnts[this];
    if (! (refcntStorage & SIDE_TABLE_RC_PINNED)) {
        refcntStorage += SIDE_TABLE_RC_ONE;
    }
    table.unlock();

    return (id)this;
}
```

`refcntStorage += SIDE_TABLE_RC_ONE` 让人费解，实际是怎么回事呢？我们通过距离说明：

如果 obj 的引用计数数值为 1（二进制 00000100，因为第一位，第二位用来标识其他内容），现在如果进行 retain，需要对引用计数数值增加 1，那么需要由 00000100 => 00001000。所以实际上，从整型的角度，是 `retainCount + 4`，而不是我们理解的 +1。

SIDE_TABLE_RC_ONE 定义是的 1UL<<2，也就是4，所以这里 `refcntStorage += SIDE_TABLE_RC_ONE;`。

# release

**objc4-706 NSObject.mm release:**

```objc
- (oneway void)release {
    ((id)self)->rootRelease();
}
```

**objc4-706 objc-object.h rootRelease:**

```objc
inline bool 
objc_object::rootRelease()
{
    if (isTaggedPointer()) return false;
    return sidetable_release(true);
}
```

**objc4-706 NSObject.mm sidetable_release:**

```objc
// rdar://20206767
// return uintptr_t instead of bool so that the various raw-isa 
// -release paths all return zero in eax
uintptr_t
objc_object::sidetable_release(bool performDealloc)
{
#if SUPPORT_NONPOINTER_ISA
    assert(!isa.nonpointer);
#endif
    SideTable& table = SideTables()[this];

    bool do_dealloc = false;

    table.lock();
    RefcountMap::iterator it = table.refcnts.find(this);
    if (it == table.refcnts.end()) {
        do_dealloc = true;
        table.refcnts[this] = SIDE_TABLE_DEALLOCATING;
    } else if (it->second < SIDE_TABLE_DEALLOCATING) {
        // SIDE_TABLE_WEAKLY_REFERENCED may be set. Don't change it.
        do_dealloc = true;
        it->second |= SIDE_TABLE_DEALLOCATING;
    } else if (! (it->second & SIDE_TABLE_RC_PINNED)) {
        it->second -= SIDE_TABLE_RC_ONE;
    }
    table.unlock();
    if (do_dealloc  &&  performDealloc) {
        ((void(*)(objc_object *, SEL))objc_msgSend)(this, SEL_dealloc);
    }
    return do_dealloc;
}
```

看后面几个判断。

1. 如果对象记录在引用计数表的最后一个：`do_dealloc` 设置为 true，引用计数数值设置为 SIDE_TABLE_DEALLOCATING（二进制 00000010）。
2. 如果 8 位的引用计数小于 SIDE_TABLE_DEALLOCATING（二进制 00000010），也就如果是 00000001 或 00000000：`do_dealloc` 设置为 true，并添加 deallocating 标识位。（但至于有什么用不太理解，希望哪位大神指点一下）。
3. 如果已经 `8 位引用计数 & SIDE_TABLE_RC_PINNED` ，即对象不在 deallocating，且没有被弱引用，且 8 位没有溢出：8 位引用计数减少 4，即真实引用计数数值 -1。
4. 最后，如果 `do_dealloc` 和 `performDealloc`（传入时就已经为 true）都为 ture，执行 SEL_dealloc 释放对象。
5. 方法返回 do_dealloc。

----

如果你只想知道 ARC 引用计数相关，那么只需要看上面的代码就可以了。alloc 和 dealloc 主要是对对象的一些内存分配。

----

# alloc

查看 alloc 相关代码。

**objc4-706 NSObject.mm alloc:**

```objc
+ (id)alloc {
    return _objc_rootAlloc(self);
}
```

**objc4-706 NSObject.mm _objc_rootAlloc:**

```objc
id
_objc_rootAlloc(Class cls)
{
    return callAlloc(cls, false/*checkNil*/, true/*allocWithZone*/);
}
```

**objc4-706 NSObject.mm callAlloc:**

```objc
// Call [cls alloc] or [cls allocWithZone:nil], with appropriate 
// shortcutting optimizations.
static ALWAYS_INLINE id
callAlloc(Class cls, bool checkNil, bool allocWithZone=false)
{
    if (slowpath(checkNil && !cls)) return nil;

#if __OBJC2__
    if (fastpath(!cls->ISA()->hasCustomAWZ())) {
        // No alloc/allocWithZone implementation. Go straight to the allocator.
        // fixme store hasCustomAWZ in the non-meta class and 
        // add it to canAllocFast's summary
        if (fastpath(cls->canAllocFast())) {
            // No ctors, raw isa, etc. Go straight to the metal.
            bool dtor = cls->hasCxxDtor();
            id obj = (id)calloc(1, cls->bits.fastInstanceSize());
            if (slowpath(!obj)) return callBadAllocHandler(cls);
            obj->initInstanceIsa(cls, dtor);
            return obj;
        }
        else {
            // Has ctor or raw isa or something. Use the slower path.
            id obj = class_createInstance(cls, 0);
            if (slowpath(!obj)) return callBadAllocHandler(cls);
            return obj;
        }
    }
#endif

    // No shortcuts available.
    if (allocWithZone) return [cls allocWithZone:nil];
    return [cls alloc];
}
```

进入方法后先进行 `if (slowpath(checkNil && !cls)) return nil;` 判断。

**objc4-706 objc-os.h slowpath:**

```objc
#define fastpath(x) (__builtin_expect(bool(x), 1))
#define slowpath(x) (__builtin_expect(bool(x), 0))
```
`__builtin_expect(exp, n)` 方法表示 exp 很有可能为 0，返回值为 exp。你可以将 `fastpath(x)` 理解成真值判断，`slowpath(x)` 理解成假值判断。

所以，根据传入值，`checkNil` 为 false，`checkNil && !cls` 也为 false。那么这里不会返回 nil。继续向下阅读。

其后是一个 Objective-C 2.0 的条件编译指令。当然我们现在用的都属于 Objctive-C 2.0，会执行其中代码。首先进行一个判断 `if (fastpath(!cls->ISA()->hasCustomAWZ()))... else ...`，这是判断一个类是否有自定义的 `+allocWithZone` 实现。

如果没有自定义的 `+allocWithZone` 实现。进行下一步，又是一个判断：`if (fastpath(cls->canAllocFast()))... else ...`，这里只有对象不存在、没有 isa 等情况才会为真值。所以之间看 else 内容。

else 代码块中调用了 ` id obj = class_createInstance(cls, 0);`。查看内容时注意查看 `objc-runtime-new.h` 中的内容而不是 `objc-runtime-old.mm` 中的内容（你可以注意到 `objc-runtime-new.h` 顶部的 Coptyright 是 Copyright (c) 2005-2007 Apple Inc.  All Rights Reserved.）

在 `objc-runtime-new.mm` 中 `canAllocFast()` 定义如下：

**objc4-706 objc-runtime-new.h canAllocFast:**

```objc
#if FAST_ALLOC
    bool canAllocFast() {
        return bits & FAST_ALLOC;
    }
#else
    bool canAllocFast() {
        return false;
    }
#endif
```

再看 FAST_ALLOC 定义，观察下图：

![FAST_ALLOC](http://7xt4xp.com1.z0.glb.clouddn.com/blog_Advanced-iOS-Study-objc-Memory-2-02.png)

发现，`#elif 1` 直接拦截了下面的 define，所以 `#if FAST_ALLOC` 不起作用（这里我也不是很确定，哪位大神指点一下）。所以，`canAllocFast()` 返回 false，`fastpath(cls->canAllocFast())` 判断为假。

执行

```objc
// Has ctor or raw isa or something. Use the slower path.
id obj = class_createInstance(cls, 0);
if (slowpath(!obj)) return callBadAllocHandler(cls);
return obj;
```

**objc4 objc-runtime-new.mm class_createInstance:**

```objc
id 
class_createInstance(Class cls, size_t extraBytes)
{
    return _class_createInstanceFromZone(cls, extraBytes, nil);
}
```

**objc4 objc-runtime-new.mm _class_createInstanceFromZone:**

```objc
id 
_class_createInstanceFromZone(Class cls, size_t extraBytes, void *zone)
{
    void *bytes;
    size_t size;

    // Can't create something for nothing
    if (!cls) return nil;

    // Allocate and initialize
    size = cls->alignedInstanceSize() + extraBytes;

    // CF requires all objects be at least 16 bytes.
    if (size < 16) size = 16;

    if (zone) {
        bytes = malloc_zone_calloc((malloc_zone_t *)zone, 1, size);
    } else {
        bytes = calloc(1, size);
    }

    return objc_constructInstance(cls, bytes);
}
```

传入的 `extraBytes` 为0，`zone` 为 nil，那么主要执行的语句是 `bytes = calloc(1, size);` 和 `return objc_constructInstance(cls, bytes);`。bytes 是对象所需内存空间。

> **FYI:** 
> `calloc(size_t __count, size_t __size)` 是 C 语言中的方法，用来在内存的动态存储区中分配 n 个长度为size的连续空间，函数返回一个指向分配起始地址的指针。如果分配不成功，返回NULL。

```objc
objc_constructInstance(Class cls, void *bytes) 
{
    if (!cls  ||  !bytes) return nil;

    id obj = (id)bytes;

    obj->initIsa(cls);

    if (cls->hasCxxCtor()) {
        return object_cxxConstructFromClass(obj, cls);
    } else {
        return obj;
    }
}
```

`objc_constructInstance` 方法中，将 `bytes`（指向分对象的指针）定义为 `obj`，并将 `obj` 的 isa 赋值为传入的 cls。最后返回 obj。

> **FYI:**
> hasCxxCtor() 是判断当前 class 或者 superclass 是否有 .cxx_construct 构造方法的实现。
> hasCxxDtor() 是判断判断当前 class 或者 superclass 是否有 .cxx_destruct 析构方法的实现。
> 参考：[Objc 对象的今生今世](http://ios.jobbole.com/90310/)

# dealloc

找到 dealloc 的实现。

**objc4 NSObject.mm dealloc:**

```objc
- (void)dealloc {
    _objc_rootDealloc(self);
}
```

**objc4 NSObject.mm _objc_rootDealloc:**

```objc
void
_objc_rootDealloc(id obj)
{
    assert(obj);

    obj->rootDealloc();
}
```

**objc4 NSObject.mm _objc_rootDealloc:**

```objc
inline void
objc_object::rootDealloc()
{
    if (isTaggedPointer()) return;
    object_dispose((id)this);
}
```

> **FYI:**
> [深入理解Tagged Pointer](http://blog.devtang.com/2014/05/30/understand-tagged-pointer/)

**objc4 objc-runtime-new.mm object_dispose:**

```objc
id 
object_dispose(id obj)
{
    if (!obj) return nil;

    objc_destructInstance(obj);    
    free(obj);

    return nil;
}
```

**objc4 objc-runtime-new.mm objc_destructInstance:**

```objc
/***********************************************************************
* objc_destructInstance
* Destroys an instance without freeing memory. 
* Calls C++ destructors.
* Calls ARC ivar cleanup.
* Removes associative references.
* Returns `obj`. Does nothing if `obj` is nil.
**********************************************************************/
void *objc_destructInstance(id obj) 
{
    if (obj) {
        // Read all of the flags at once for performance.
        bool cxx = obj->hasCxxDtor();
        bool assoc = obj->hasAssociatedObjects();

        // This order is important.
        if (cxx) object_cxxDestruct(obj);
        if (assoc) _object_remove_assocations(obj);
        obj->clearDeallocating();
    }

    return obj;
}
```

**objc4 objc-object.h clearDeallocating:**

```objc
inline void 
objc_object::clearDeallocating()
{
    sidetable_clearDeallocating();
}

```

**objc4 NSObject.mm sidetable_clearDeallocating:**

```objc
void 
objc_object::sidetable_clearDeallocating()
{
    SideTable& table = SideTables()[this];

    // clear any weak table items
    // clear extra retain count and deallocating bit
    // (fixme warn or abort if extra retain count == 0 ?)
    table.lock();
    RefcountMap::iterator it = table.refcnts.find(this);
    if (it != table.refcnts.end()) {
        if (it->second & SIDE_TABLE_WEAKLY_REFERENCED) {
            weak_clear_no_lock(&table.weak_table, (id)this);
        }
        table.refcnts.erase(it);
    }
    table.unlock();
}
```

所以，`objc_destructInstance(obj);` 中进行了销毁实例但不释放内存，调用了 C++ 的析构函数（如果对象有），处理先关的对象（如果有），最后调用 `obj->clearDeallocating();` 清除 weak 引用、清除多余的 retain count。`objc_destructInstance(obj);` 之后是 `free(obj);` 释放 obj 占用的内存空间。


#Other

（在写这边文章的时候，我怎么感觉我最大的感触是，C++ 不懂。。）

写本篇博文时参考的所有资料：

> [Tracy Wang-深入浅出ARC(上)](http://blog.tracyone.com/2015/06/14/%E6%B7%B1%E5%85%A5%E6%B5%85%E5%87%BAARC-%E4%B8%8A/)
> 
> [原来我非不快乐-我们的对象会经历什么](http://www.jianshu.com/p/ff8a7c458c96)
> 
> [desgard-weak 弱引用的实现方式](http://www.desgard.com/weak/)
> 
> [Objc 对象的今生今世](http://ios.jobbole.com/90310/)
> 
> [玉令天下-Objective-C 引用计数原理](http://yulingtianxia.com/blog/2015/12/06/The-Principle-of-Refenrence-Counting/)
> 
> [一缕殇流化隐半边冰霜-神经病院Objective-C Runtime入院第一天——isa和Class](http://www.jianshu.com/p/9d649ce6d0b8)
> 
> [sindrilin-闲聊内存管理](http://sindrilin.com/runtime/2016/12/23/闲聊内存管理)

----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)


