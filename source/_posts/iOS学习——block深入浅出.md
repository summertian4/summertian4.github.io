title: iOS学习——block深入浅出
date: 2015-11-20 12:00:58
tags:
  - iOS
categories:
  - iOS
---

其实几乎天天都在用block吧，却没有仔细研究过，这次也当给自己补课啦

# block
这几个基本概念会帮助你理解block

1.	block是一种变量类型
2.	block是C级别的语法和运行机制
3.	block除了包含可执行代码以外，还包含了与堆、栈内存绑定的变量。所以Block对象包含着一组状态数据，这些数据在程序执行时用于对行为产生影响
4.	block的意义在于不仅包含了回调期间的代码，又包含了执行期间需要的数据，并且支持多线程
5.	block类似于C语言的函数指针，不要看的太难

# block的定义和基本结构
放上@传智如意大师的解说图
![block定义](http://img.my.csdn.net/uploads/201208/07/1344323584_7609.png)

现在看是不是有点麻烦？直接上例子，小鱼喜欢把详细的解释放在代码里（程序员风格），相信你一定瞬间弄懂

建立了一个command项目，列出了各种block的定义

```objc
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        /********* 1.1 无参数无返回值-完整写法 *********/
        void (^myBlock1)() = ^(){
            NSLog(@"无参数无返回值的block");
        };
        myBlock1();
        
        
        
        
        /********* 1.2 无参数无返回值-简写 *********/
        void (^myBlock1_simple)() = ^{
            NSLog(@"无参数无返回值的block的简写");
        };
        myBlock1_simple();
        
        
        
        
        /********* 2.1 有参数无返回值 *********/
        /**
         * void (^block变量名)(参数类型及个数) = ^(形参列表){代码块};
         */
        void (^myBlock2)(int, int) = ^(int x, int y){
            NSLog(@"有参数无返回值，两数和为:%d", x + y);
        };
        myBlock2(1,2);
        
        
        
        
        /********* 2.2 对block变量重新赋值 *********/
        myBlock2 = ^(int x, int y) {
            NSLog(@"对block变量重新赋值，两数差为:%d", x - y);
        };
        myBlock2(1,2);
        
        
        
        
        /********* 3 无参数有返回值 *********/
        NSString *(^myBlock3)() = ^ (){
            return @"无参数有返回值";
        };
        NSLog(@"%@", myBlock3());
        
        
        
        
        /********* 4 有参数有返回值 *********/
        int (^myBlock4)(int, int) = ^(int x, int y) {
            return x + y;
        };
        
        int result = myBlock4(1,2);
        NSLog(@"有参数有返回值，两数和为:%d", result);
        
        
        
        
        
    }
    return 0;
}

```

<!--more-->


# block的类型定义/typedef
C语言的typedef，大家应该比较常见，没见过也没关系。typedef用来为复杂的声明定义简单的别名，比如对变量、类型等等取别名。

OC是兼容C的，block又是一个block是C级别的语法，所以我们可以为block进行别称定义。

别称定义是为了减少代码量，并使得代码可读性更强。你可以把『对block的typedef』理解成『对象抽象成类』

```objc
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // 定义、赋值
        void (^myBlock)() = ^{
            NSLog(@"myBlock");
        };
        
        // 调用
        myBlock();
        
        
        
        
        // 定义一个代码块类型
        typedef void (^SimpleBlockType)();
        
        // 创建一个该类型的变量并赋值
        SimpleBlockType simpleBlock;
        simpleBlock = myBlock;
        
        // 调用
        simpleBlock();
        
        
        
        
        // 定义一个代码块类型
        typedef int (^Parameter_Return_BlockType)(int, int);
        
        // 创建一个该类型的变量并赋值
        Parameter_Return_BlockType p_r_Block;
        p_r_Block = ^(int x, int y) {
            return x + y;
        };
        
        // 调用
        int result = p_r_Block(1, 2);
        NSLog(@"x + y = %d", result);
        
    }
   
```

现在你可以看出将block进行typedef可以把一种block当做类的实例化一样使用，非常方便。

# block访问外部变量
其实对很多来说，一开始使用block最头疼的是弄不清block对外部变量的访问。

你只需要记住两点：
1. block只允许对`普通`外部变量进行读操作，不允许写操作
2. block运行对声明为`__block类型`的变量进行读写操作

## 读操作
```objc
        /************** 内存与读访问 **************/
        NSLog(@"========================= 内存与读访问 =========================");
        // 外部变量
        int externalVariable = 0;
        NSLog(@"外部访问：externalVariable = %d", externalVariable);
        NSLog(@"外部访问：externalVariable的地址是：%p", &externalVariable);
        NSLog(@"\n");
        
        void (^myBlock)() = ^{
            NSLog(@"内部访问：externalVariable = %d", externalVariable);
            NSLog(@"内部访问：externalVariable的地址是：%p", &externalVariable);
            NSLog(@"内部访问：当block访问外部变量时，会将变量的值以const方式copy一份到block的所在内存空间");
            NSLog(@"内部访问：所以在block中对外部变量操作对外部变量不会产生影响，编译器也不允许在block内对外部变量修改");
            NSLog(@"\n");
        };
        myBlock();
        
        NSLog(@"回到外部访问：externalVariable的地址是：%p", &externalVariable);
```

以上代码的运行结果：

```objc
block_访问外部变量[34781:2197055] ====================== 内存与读访问 ======================
block_访问外部变量[34781:2197055] 外部访问：externalVariable = 0
block_访问外部变量[34781:2197055] 外部访问：externalVariable的地址是：0x7fff5fbff78c
block_访问外部变量[34781:2197055] 
block_访问外部变量[34781:2197055] 内部访问：externalVariable = 0
block_访问外部变量[34781:2197055] 内部访问：externalVariable的地址是：0x100600020
block_访问外部变量[34781:2197055] 内部访问：当block访问外部变量时，会将变量的值以const方式copy一份到block的所在内存空间
block_访问外部变量[34781:2197055] 内部访问：所以在block中对外部变量操作对外部变量不会产生影响，编译器也不允许在block内对外部变量修改
block_访问外部变量[34781:2197055] 
block_访问外部变量[34781:2197055] 回到外部访问：externalVariable的地址是：0x7fff5fbff78c
```

### 分析
1. 你可以看出block可以对`外部变量externalVariable`读访问
2. 你可以看出block内部访问到的和`外部变量externalVariable`本身地址不同
3. 你可以推测block内部访问到的其实并不是真正的那个`外部变量externalVariable`

### 解释
1. block访问普通外部变量时，会将**变量的值**以**const**方式**copy**一份到**block的所在内存空间**
2. 所以block内部访问的并不是真正的外部变量
3. const是指以常量方式，所以编译器不允许修改copy的变量

## 写操作
如果真的要对外部变量进行写操作呢？
你需要做的是：
1. 将你需要要操作的变量定义成__block

```objc
        /************** 写访问 **************/
        NSLog(@"========================= 写访问 ==========================");
        // 外部变量
        __block int externalVariable2 = 0;
        NSLog(@"外部访问：externalVariable2 = %d", externalVariable2);
        NSLog(@"外部访问：externalVariable2的地址是：%p", &externalVariable2);
        NSLog(@"\n");
        
        void (^myBlock2)() = ^{
            externalVariable2 = 100;
            NSLog(@"内部访问：externalVariable2 = %d", externalVariable2);
            NSLog(@"内部访问：externalVariable2的地址是：%p", &externalVariable2);
            NSLog(@"内部访问：仍然会copy一份到block的所在内存空间，但不是以const方式");
            NSLog(@"\n");
        };
        myBlock2();
        
        NSLog(@"回到外部访问：externalVariable2 = %d", externalVariable2);
        NSLog(@"回到外部访问：externalVariable2的地址是：%p", &externalVariable2);
```

以上代码运行结果：

```objc
block_访问外部变量[34781:2197055] ======================== 写访问 ========================
block_访问外部变量[34781:2197055] 外部访问：externalVariable2 = 0
block_访问外部变量[34781:2197055] 外部访问：externalVariable2的地址是：0x7fff5fbff750
block_访问外部变量[34781:2197055] 
block_访问外部变量[34781:2197055] 内部访问：externalVariable2 = 100
block_访问外部变量[34781:2197055] 内部访问：externalVariable2的地址是：0x100600048
block_访问外部变量[34781:2197055] 内部访问：仍然会copy一份到block的所在内存空间，但不是以const方式
block_访问外部变量[34781:2197055] 
block_访问外部变量[34781:2197055] 回到外部访问：externalVariable2 = 100
block_访问外部变量[34781:2197055] 回到外部访问：externalVariable2的地址是：0x100600048
```

### 分析
1. 你可以看出声明为__block类型的变量可以在block内部进行写访问
2. 你可以看出`外部变量externalVariable2` 在block被访问时和访问前的地址仍然不同
3. 你可以看出`外部变量externalVariable2` 在block被访问后地址改变了

### 解释
1. 对于声明为__block类型的变量在block中访问时，仍然会copy一份到block的所在内存空间，但不是以const方式
2. 所以你可以对变量进行写操作
3. 在block中访问__block类型的变量后，该变量的地址将会更改为block中的copy的地址

# block的回调
通常我们成为回调，在java中也称为钩子函数，spring称为面向切面编程，是一种基于OOP的优秀编程思想

有的时候，我们会重复的写一段逻辑，它可能是这样的：

```
事件A
事件B
**特殊事件X**
事件C
事件D
事件E

```

这一段逻辑的前部分和后部分是固定的，但是中间会发生一段视情况而定的代码。
如果这段逻辑要发生5遍，那么就要重复写5次几乎相同的代码。这时候你会想，如果只要写中间那一段就可以大幅度优化代码。
block可以帮你实现这一段逻辑

你可以这样写：

```objc
void method(BlockType block) {
    NSLog(@"事件A");
    NSLog(@"事件B");
    block();
    NSLog(@"事件C");
    NSLog(@"事件D");
    NSLog(@"事件E");
    NSLog(@"==================");

}


int main(int argc, const char * argv[]) {
    @autoreleasepool {
        method(^ {
            NSLog(@"事件X1");
        });
        method(^ {
            NSLog(@"事件X2");
        });
    }
    return 0;
}
```

运行结果：

```objc
block_回调[34821:2210234] 事件A
block_回调[34821:2210234] 事件B
block_回调[34821:2210234] 事件X1
block_回调[34821:2210234] 事件C
block_回调[34821:2210234] 事件D
block_回调[34821:2210234] 事件E
block_回调[34821:2210234] ==================
block_回调[34821:2210234] 事件A
block_回调[34821:2210234] 事件B
block_回调[34821:2210234] 事件X2
block_回调[34821:2210234] 事件C
block_回调[34821:2210234] 事件D
block_回调[34821:2210234] 事件E
block_回调[34821:2210234] ==================
```

现在你可以看出回调的意思——执行自己的一段代码，再去掉别人实现的一段代码，再回到自己的代码。

block实现的回调非常方便使用。

在JAVA著名的Spring框架中，提出了面向切面编程。也就是利用回调，向代码中插入切面，拦截、过滤、织入自己想添加的逻辑。切面编程可以实现种种不可思议的编程，在每一个方法前、方法后、或者方法中的任意位置，织入你想添加逻辑。这种结合了责任链设计模式的优秀的编程思想被迅速广泛使用。

iOS在加入了block后同样被广泛使用，所以掌握好block是iOS学习之路不可少的一部分。

----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)


