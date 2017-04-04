title: iOS命名与编码规范文档
date: 2015-11-03 13:30
tags:
  - iOS
categories:
  - iOS
---

项目经理的要求，给成员写一份简单的命名和编码规范，因为成员很多都是新人，甚至还有刚开始接触规范编码的，所以我就写了一份简单的。

还有很多不完善的地方，而且只是针对这次项目的，只是给一个参考。


----------


# XXXiOS项目-命名及编码规范
2015年11月3日星期二
周凌宇

<br/ >
## **1 类**
1.1	遵守大驼峰命名法
1.2	Controller命名：
&nbsp;&nbsp;&nbsp;&nbsp;1.1.1	ViewController子类命名规范：XXXController

![ViewController子类命名规范](http://7xnrog.com1.z0.glb.clouddn.com/blog_iOS%E5%91%BD%E5%90%8D%E4%B8%8E%E7%BC%96%E7%A0%81%E8%A7%84%E8%8C%83%E6%96%87%E6%A1%A3001.png)

&nbsp;&nbsp;&nbsp;&nbsp;1.1.2	TableViewController子类命名规范：XXXTableController
&nbsp;&nbsp;&nbsp;&nbsp;1.1.3	其他Controller命名规范：与ViewController子类命名规范相同

1.1	View命名：
&nbsp;&nbsp;&nbsp;&nbsp;1.1.1	View子类命名规范：XXXView
&nbsp;&nbsp;&nbsp;&nbsp;1.1.2	View的Xib与相应的类同名

![View的Xib与相应的类同名](http://7xnrog.com1.z0.glb.clouddn.com/blog_iOS%E5%91%BD%E5%90%8D%E4%B8%8E%E7%BC%96%E7%A0%81%E8%A7%84%E8%8C%83%E6%96%87%E6%A1%A3002.png)

<br/ >

## **2	协议**
2.1	遵守大驼峰命名法
2.2	代理：
&nbsp;&nbsp;&nbsp;&nbsp;2.2.1	代理写在相应类型的.h文件中，不需要单独建立文件
&nbsp;&nbsp;&nbsp;&nbsp;2.2.2	命名规范：XXXDelegate，XXX和相应类名一致

![代理命名规范](http://7xnrog.com1.z0.glb.clouddn.com/blog_iOS%E5%91%BD%E5%90%8D%E4%B8%8E%E7%BC%96%E7%A0%81%E8%A7%84%E8%8C%83%E6%96%87%E6%A1%A3003.png)

<br/ >

## **3 宏定义**
3.1	避免在程序中直接出现常数，常数宏定义变量全部大写
3.2	使用超过一次的小段代码应以宏定义的形式来替代，遵循大驼峰命名法
3.3	常量的命名应当能够表达出它的用途

![宏定义示例](http://7xnrog.com1.z0.glb.clouddn.com/blog_iOS%E5%91%BD%E5%90%8D%E4%B8%8E%E7%BC%96%E7%A0%81%E8%A7%84%E8%8C%83%E6%96%87%E6%A1%A3004.png)

<br/ >

## **4 枚举**
4.1	遵循大驼峰命名法
4.2	枚举写在相应的.h文件中，不需要单独建立文件
4.3	枚举中的变量以枚举名开头，如：ExamPaperType枚举中包含的变量是ExamPaperTypeExam、ExamPaperTypeCheck、ExamPaperTypeGrade

![枚举命名规范](http://7xnrog.com1.z0.glb.clouddn.com/blog_iOS%E5%91%BD%E5%90%8D%E4%B8%8E%E7%BC%96%E7%A0%81%E8%A7%84%E8%8C%83%E6%96%87%E6%A1%A3005.png)

<br/ >

## **5 方法**
5.1	遵循小驼峰命名法
5.2	方法的名称应全部使用有意义的单词组成
5.3	set、get方法：
&nbsp;&nbsp;&nbsp;&nbsp;5.3.1	set方法命名规范: `- (void)setUUID:(NSString *)UUID`
&nbsp;&nbsp;&nbsp;&nbsp;5.3.2	get方法命名规范：`- (NSString *)UUID`
5.4	init方法：
&nbsp;&nbsp;&nbsp;&nbsp;5.4.1	必须在`if (self = [super init]) {}` 中做初始化操作
![init方法](http://7xnrog.com1.z0.glb.clouddn.com/blog_iOS%E5%91%BD%E5%90%8D%E4%B8%8E%E7%BC%96%E7%A0%81%E8%A7%84%E8%8C%83%E6%96%87%E6%A1%A3006.png)

<br/ >

## **6 变量**
6.1	遵循小驼峰命名法
6.2	变量必须起有意义的名字
6.3	NSString类型属性：参数必须为(nonatomic, copy)
6.4	NSArray/NSMutableArray类型变量：变量名可以使用加后缀s或Array

![NSArray/NSMutableArray类型变量](http://7xnrog.com1.z0.glb.clouddn.com/blog_iOS%E5%91%BD%E5%90%8D%E4%B8%8E%E7%BC%96%E7%A0%81%E8%A7%84%E8%8C%83%E6%96%87%E6%A1%A3007.png)

6.5	NSDictionary/ NSMutableDictionary类型变量：变量名使用后缀Dic
6.6	控件类型变量命名规范：
&nbsp;&nbsp;&nbsp;&nbsp;6.6.1	UIView类型变量：XXXView
&nbsp;&nbsp;&nbsp;&nbsp;6.6.2	UIScrollView类型变量：XXXScrollView
&nbsp;&nbsp;&nbsp;&nbsp;6.6.3	UIImageView类型变量：imgXXX
&nbsp;&nbsp;&nbsp;&nbsp;6.6.4	UIButton类型变量：btXXX
&nbsp;&nbsp;&nbsp;&nbsp;6.6.5	UILabel类型变量：lblXXX
&nbsp;&nbsp;&nbsp;&nbsp;以此类推

<br/ >

## **7 注释**
7.1	变量、方法、枚举使用标准**文档**注释
&nbsp;&nbsp;&nbsp;&nbsp;7.1.1	变量注释应详细描述变量用途
&nbsp;&nbsp;&nbsp;&nbsp;7.1.2	枚举注释应详细描述枚举和每一个元素用途
&nbsp;&nbsp;&nbsp;&nbsp;7.1.3	方法注释应详细描述方法作用、参数意义、返回值意义

![注释](http://7xnrog.com1.z0.glb.clouddn.com/blog_iOS%E5%91%BD%E5%90%8D%E4%B8%8E%E7%BC%96%E7%A0%81%E8%A7%84%E8%8C%83%E6%96%87%E6%A1%A3008.png)

7.2	其他使用单行注释

<br/ >

## **8	Storyboard ID**
8.1	自定义的并只在Storyboard 出现一次的Controller，其Storyboard ID与类型名相同
8.2	其他情况必须起有意义的名字

<br/ >

## **9 资源文件规范**
1.1	资源文件全部放入Supporting Files文件夹下
1.2	图片资源放入Assets.xcassets。可以建立自己的Folder


----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)

