title: 文档生成器 Xcode与Appledoc
date: 2015-10-13 16:33
tags:
  - iOS
categories:
  - iOS
---

# **目录**
[TOC]

博主是不写注释会死星人，以前 java 的时候常用 javadoc ，只要写好注释，然后用 javadoc 生成html 格式的文档。用eclipse和myeclipse都能挂上去。
最近的iOS项目是一个十几个人一起写的项目，很多都初学者，我写主要框架这一块。uml和编码都搞定了，但是为了协作给他人使用，需要一份文档。
除了word和markdown写的reference，api文档也是必不可少的。
所以就开始捣鼓appledoc。在中间遇到不少问题，最后成功的解决了，所以特意分享给大家。

# **安装Appledoc**
**Appledoc的github地址：**https://github.com/tomaz/appledoc

其实不用下载的，在github项目的readme中已经写了安装方法：

## **快速安装**
打开终端，输入： 

```
 git clone git://github.com/tomaz/appledoc.git
```
等待完成后继续输入：

```
sudo sh install-appledoc.sh
```
等待安装完成。如果出现错误，参考后面的错误解决
<br/>

## **brew安装**
如果你装了brew，Appledoc官方文档写的是打开终端输入：

```
brew install appledoc
```
<hr/>

<!--more-->

## **错误解决**
我试了使用brew安装，然而显示错误：

```
Error: No available formula for appledoc 
==> Searching formulae...
==> Searching taps...
```
这个问题让我查了很多资料都没解决，最后发现可能是新版的brew不能这样安装Appledoc（是我的猜测）。

所以决定使用**快速安装**。
要注意的是，使用快速安装要**保证`/usr/local/bin`路径要存在**。如果没有，一定要手动创建相应的文件夹，并且保证bin文件夹是可读可写的（可以在文件夹的『显示简介』里更改）

![设置/usr/local/bin文件夹可读可写](http://img.blog.csdn.net/20151013153313995)

然后就可以放心按照上面『快速安装』安装了，不会出现问题。

-----

# **Appledoc使用**

##**在xcode里使用**
网上找的很多资料都是在很老版本的xcode中使用appledoc的方法，博主用的是xcode6和xcode7。
首先点击file->new->target

![这里写图片描述](http://img.blog.csdn.net/20151013155112543)

然后在弹出的界面中选择Aggregate

![这里写图片描述](http://img.blog.csdn.net/20151013155205606)

填写好名字

![这里写图片描述](http://img.blog.csdn.net/20151013155748077)

这样就添加好了一个Target
然后会弹出一个界面，不同版本长得略有不同
总之选择Build Phases，点击左边的小加号

![这里写图片描述](http://img.blog.csdn.net/20151013160016656)

选择New Run Script Phase

![这里写图片描述](http://img.blog.csdn.net/20151013160128085)

建好了以后打开刚刚建立的Run Script

![这里写图片描述](http://img.blog.csdn.net/20151013160235052)

把红框的地方里面替换成：

```
#appledoc Xcode script
# Start constants
company="ACME";
companyID="com.ACME";
companyURL="http://ACME.com";
target="iphoneos";
#target="macosx";
outputPath="~/help";
# End constants

/usr/local/bin/appledoc \
--project-name "${PROJECT_NAME}" \
--project-company "${company}" \
--company-id "${companyID}" \
--docset-atom-filename "${company}.atom" \
--docset-feed-url "${companyURL}/${company}/%DOCSETATOMFILENAME" \
--docset-package-url "${companyURL}/${company}/%DOCSETPACKAGEFILENAME" \
--docset-fallback-url "${companyURL}/${company}" \
--output "${outputPath}" \
--publish-docset \
--docset-platform-family "${target}" \
--logformat xcode \
--keep-intermediate-files \
--no-repeat-first-par \
--no-warn-invalid-crossref \
--exit-threshold 2 \
"${PROJECT_DIR}"
```

然后点左上角的项目，发现多了一个Document
![这里写图片描述](http://img.blog.csdn.net/20151013160702336)
![这里写图片描述](http://img.blog.csdn.net/20151013160717721)

点Document，然后再运行，只要没报错就OK了
文档已经编译好并且自动安装进Xcode了。重启xcode，打开documentation。就会发现里面有你刚刚生成的文档。
![这里写图片描述](http://img.blog.csdn.net/20151013161028665)

如果你想直接看html

可以用Finder进入`~/Library/Developer/Shared/Documentation/DocSets`
看到你的文档以后可以右键查看包内容，就可以拿到里面的Html文档了


### **终端使用**
**博主还没有试过，可以先尝试上面的方法**
```
appledoc --project-name test     
         --project-company "test"   
         --company-id com.test    
         --output /Users/zhoulingyu/Desktop
         /Users/zhoulingyu/Desktop/Test/Classes        
     
```
从上到下分别代表的是：

 1. 工程名称
 2. 公司名称
 3. 工程ID
 4. 生成结果输出路径
 5. 扫描哪个路径下的类.


----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)


