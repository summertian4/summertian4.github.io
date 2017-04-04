title: 近日关于 hexo 主题 nexT 博客加载空白问题
date: 2016-11-07 12:42:43
tags:
	- Other
categories:
  - Other
---

周末的时候，突然收到了一些邮件以及知乎的私信，告诉我我的博客似乎挂了。

今天查看的时候，发现确实出现了大片的空白。

以下是排查步骤。

### 1. 用 Chrome 的开发者工具中的 Network 选项查看

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_%E5%85%B3%E4%BA%8E%20hexo%20%E4%B8%BB%E9%A2%98%20nexT%20%E5%8D%9A%E5%AE%A2%E5%8A%A0%E8%BD%BD%E7%A9%BA%E7%99%BD%E9%97%AE%E9%A2%98-01.png)

<!-- More -->

发现很多 js 都加载失败，在加载失败的 js 右键 `Open Link in New Tab` 打开，发现返回 Github 的 404 页面。

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_%E5%85%B3%E4%BA%8E%20hexo%20%E4%B8%BB%E9%A2%98%20nexT%20%E5%8D%9A%E5%AE%A2%E5%8A%A0%E8%BD%BD%E7%A9%BA%E7%99%BD%E9%97%AE%E9%A2%98-02.png)

访问失败的都是 `vendors` 目录下的资源。

> 其实我前端工具用的很不熟，这里隆重感谢[赵鲜华](http://www.jianshu.com/users/86344ec5bfe7/latest_articles)同学的热心帮助。

### 2.查看 Github 博客的 repo

查看 Github 博客的 repo，发现文件确实是存在的。

Travis-CI 也是正常构建的。
 
### 3. 翻阅 nexT 主题 isusses

 最终在 isusse 中找到了原因。
 
 GitHub Pages 禁止了 source/vendors 目录的访问。其原因是 Github 在 11 月 3 日更新了版本。其中包括升级了 Jekyll 到 3.3。Jekyll 为了加快构建速度，忽略 vendor 和 node_modules 文件夹。更新日志在[这里](https://github.com/blog/2277-what-s-new-in-github-pages-with-jekyll-3-3)
 
 ![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_%E5%85%B3%E4%BA%8E%20hexo%20%E4%B8%BB%E9%A2%98%20nexT%20%E5%8D%9A%E5%AE%A2%E5%8A%A0%E8%BD%BD%E7%A9%BA%E7%99%BD%E9%97%AE%E9%A2%98-04.png)

[这是解决问题的 isusse](https://github.com/iissnan/hexo-theme-next/issues/1214)。
 
关键处如下：

![](http://7xt4xp.com1.z0.glb.clouddn.com/blog_%E5%85%B3%E4%BA%8E%20hexo%20%E4%B8%BB%E9%A2%98%20nexT%20%E5%8D%9A%E5%AE%A2%E5%8A%A0%E8%BD%BD%E7%A9%BA%E7%99%BD%E9%97%AE%E9%A2%98-03.png)

除了手动解决方法之外，可以通过在 nexT 主题目录下，`git pull` 更新。

----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)

