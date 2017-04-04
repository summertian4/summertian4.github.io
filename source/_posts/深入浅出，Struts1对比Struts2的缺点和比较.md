title: 深入浅出，Struts1对比Struts2的缺点和比较
date: 2014-07-25 21:57
tags:
  - JAVA
categories:
  - JAVA
---


# Action的构造:

1. Struts1要求Action类继承Action或者其他几个基类。
2. Struts2的Action类不需要继承任何类，出于习惯可以继承ActionSupport。
所以任何一个javabean都可以作为Action，在写起来时候没有太多规范。理过程不再是控制死死的流程，而是通过AOP的方式将request和response进行处理。

# 依赖容器:

1. Struts1基于M2开发，所以Action依赖Servlet，导致Struts1中的Action方法必须传入HttpServletRequest和HttpServletResponse等4个参数。依赖Servlet导致依赖容器，因为调用方法必须传入参数，所有Struts1中Action不能单独被测试。
2. Struts2 Action不依赖于容器，允许Action脱离容器单独被测试。在Struts2中需要Session和Request时，可以让Action实现SessionAware和RequestAware接口，其实该接口只需要实现session和requeset的get,set方法，只要Action中有这两个对象和getter,setter就可以直接获得session和request。

# IOC思想:

在获取参数时

1. Struts1 使用ActionForm对象捕获输入，这回使得结构非常的复杂，并且让人觉得多此一举（因为你不但要使用ActionForm对象，还需要写一个ActionForm，同时在xml对其配置）。或者使用session.getAttribute()，每次获得一个参数需要使用大量的重复代码，这样导致了大量的代码冗余。最重要的是，struts1没有将属性当做对象来对待。
2. Struts2在Action中加入需要的属性，通过调用其getter,setter将页面的参数传入其中，最重要的事，它将页面的属性封装好成为对象，这里用到的就是OGNL。大家或许注意到，在页面中动态显示的部分都是一个对象全部属性或者是一个对象的部分属性，比如登陆页面，显示（或者输入）的是user对象的username和password属性。这时候，直接将封装成对象减少了大量的getAttribute，减少了代码冗余。

IOC或者说“依赖注入”的思想虽然由spring提出，但是在Struts2中已经可以体现，Action中的实体对象，就是通过注入的方式获得。

<!--more-->

# 面向切面（AOP）：

1. Struts1基于M2开发，没有太优秀的抽象。
2. Struts2中可以在源代码中看到大量的Filter，这些大量的过滤器体现了Struts2的面向切面思想。
当然大家一定会说AOP也是Spring提出的，这里可以感受到，好的编程思想其实一直在被优秀的程序员使用着，最后也会被人提出，我自己热爱开源的原因就是在于开源可以向全世界分享最新，最好的思想技术。

# 表达式语言：

1. Struts1整合了JSTL，因此使用JSTLEL。这种EL有基本对象图遍历，但是对集合和索引属性的支持很弱。
2. Struts2可以使用JSTL，但是也支持一个更强大和灵活的表达式语言－－"ObjectGraph Notation Language" (OGNL)，同时具有一个强大的s:标签库。
举个简单例子在struts1中判断标签可以使用<c:if>但是没有<c:else>而struts2中有<s:if><s:else>

#视图层的访问:

1. Struts 1使用标准JSP机制把对象绑定到页面中来访问。
2. Struts2 使用"ValueStack"技术，使taglib能够访问值而不需要把你的页面（view）和对象绑定起来。ValueStack策略允许通过一系列名称相同但类型不同的属性重用页面（view）。

# 校验：

1. Struts 1支持在ActionForm的validate方法中手动校验，或者通过CommonsValidator的扩展来校验。同一个类可以有不同的校验内容，但不能校验子对象。
2. Struts2支持通过validate方法和XWork校验框架来进行校验。XWork校验框架使用为属性类类型定义的校验和内容校验，来支持chain校验子属性

# 线程问题:

1. Struts1 Action是单例模式并且必须是线程安全的，因为仅有Action的一个实例来处理所有的请求。Action资源必须是线程安全的或同步的。
2. Struts2 Action对象为每一个请求产生一个实例，因此没有线程安全问题。


----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)

