title: 快速入门OC/iOS你必须知道的事情
date: 2015-08-07 20:06
tags:
  - iOS
categories:
  - iOS
---


# **.符号的用法**

OC中最刚开始最让人困惑的地方就是.符号，OC中object.property不是一般语言中的引用传递，实际上相当于[object setProperty]或者[object getProperty]，可能一开始就容易搞晕了。这给告诉大家到底什么时候是get方法什么时候是set方法。
当object.property出现在等号右边的时候调用的是get方法
`XXX = object.property`    相当于   `XXX = [object getProperty];`
当object.property出现在等号左边的时候调用的是set方法
`object.property = XXX`    相当于   `[object setProperty : XXX]`;
所以当看到object.property = propertyVal时，如果发生了很多不能理解的变化，那一定是在.setProperty(propertyVal)中加了很多逻辑，比如做一些深拷贝。

----------


# **self代表什么**

JAVA中代表类的对象，在OC中可以代表对象也可以代表这个类。
具体的：当self出现在对象方法（动态方法）中，代表调用方法的对象；当self出现在类方法（静态方法）中，代表类。
典型用法：
`return [[self alloc] init];//返回一个自己的对象`

----------

# **加号(+)减号(-)，并非静态方法**
补充在博文中了：[iOS学习——objective-c 加号减号，并非静态方法](http://blog.csdn.net/u010127917/article/details/47782845)

-------------------


# **@class以及@class#import的区别**
> 1.import会包含这个类的所有信息，包括实体变量和方法，而@class只是告诉编译器，其后面声明的名称是类的名称，至于这些类是如何定义的，暂时不用考虑，后面会再告诉你。

> 2.在头文件中， 一般只需要知道被引用的类的名称就可以了。 不需要知道其内部的实体变量和方法，所以在头文件中一般使用@class来声明这个名称是类的名称。 而在实现类里面，因为会用到这个引用类的内部的实体变量和方法，所以需要使用#import来包含这个被引用类的头文件。

> 3.在编译效率方面考虑，如果你有100个头文件都#import了同一个头文件，或者这些文件是依次引用的，如A–>B, B–>C, C–>D这样的引用关系。当最开始的那个头文件有变化的话，后面所有引用它的类都需要重新编译，如果你的类有很多的话，这将耗费大量的时间。而是用@class则不会。

> 4.如果有循环依赖关系，如:A–>B, B–>A这样的相互依赖关系，如果使用#import来相互包含，那么就会出现编译错误，如果使用@class在两个类的头文件中相互声明，则不会有编译错误出现。

----------


# **group**
经常写java的程序员们都绝对对习惯性的建好包，分好层。并且习惯性的认为包路径是真实存在的（有相应的文件夹），然而xcode中的group看似是文件夹，实际是只是一个逻辑上的分类，是不存在相应文件件的。
xcode中显示看似是文件夹：
![xcode中显示看似是文件夹](http://img.blog.csdn.net/20150807204034235)
实际上却是没有的：
![实际上没有文件夹](http://img.blog.csdn.net/20150807204110753)




----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)


