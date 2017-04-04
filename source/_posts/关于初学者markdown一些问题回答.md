title: 关于初学者markdown一些问题回答
date: 2014-06-15 12:17
tags:
  - 随笔
categories:
  - 随笔
---

最近有不少人问到markdown的一些小问题，关于这些问题，我觉得有必要和大家简单的总结一下，这样能迅速掌握markdown。

# 第一 看清markdown是什么

首先要明白，markdown只是一个辅助工具，它其实就是html的另外一种写法，不需要把它当做一种新的标签语言来学，这样你在学习的时候通过类比就可以快速掌握。

# 第二 markdown可以内嵌html

markdown可以内嵌html标签，在有样式问题是可以使用原生html

# 第三 二级List的表示方式

有初学者问到在表示一级list时候可以这样表示

* 一级list
* 一级list
* 一级list

但是却不知道二级list是什么样的

关于这一点，markdown的官方文档中确实也没有提到，一下是markdown参考文档中的内容摘抄

列表

Markdown 支持有序列表和无序列表。

无序列表使用星号、加号或是减号作为列表标记：

\* Red

\* Green

\* Blue

等同于：

\+ Red

\+ Green

\+ Blue

也等同于：

\- Red

\- Green

\- Blue

可以看到在文档中确实没有提到二级list，那么二级list是怎么表示的呢。 首先来看一下上一部分生成的html文档是怎样的


<pre style="margin-top:15px; margin-bottom:15px; padding:6px 10px; border:1px solid rgb(204,204,204); font-size:13px; font-family:Consolas,'Liberation Mono',Courier,monospace; background-color:rgb(248,248,248); line-height:19px; overflow:auto"><code style="margin:0px; padding:0px; font-size:12px; font-family:Consolas,'Liberation Mono',Courier,monospace; background-color:transparent">&lt;ul&gt;
&lt;li&gt;一级list&lt;/li&gt;
&lt;li&gt;一级list&lt;/li&gt;
&lt;li&gt;一级list&lt;/li&gt;
&lt;/ul&gt;
</code></pre>

从这里看以看出，markdown只是将其简单的转换为html标签，那么回到第一点，markdown只是html的另一种书写方式，同时markdown可以内嵌html，所以二级list的表达方法就显而易见了：你可以选择直接内嵌html，表示如下：

<blockquote style="margin-top:15px; margin-right:0px; margin-bottom:15px; margin-left:0px; padding-top:0px; padding-right:15px; padding-bottom:0px; padding-left:15px; border-top-width:0px; border-right-width:0px; border-bottom-width:0px; border-left-width:4px; border-style:initial; border-color:initial; border-left-style:solid; border-left-color:rgb(221,221,221); color:rgb(119,119,119)">
		<ul style="margin-top:0px; margin-right:0px; margin-bottom:0px; margin-left:0px; padding-top:0px; padding-right:0px; padding-bottom:0px; padding-left:30px; border-top-width:0px; border-right-width:0px; border-bottom-width:0px; border-left-width:0px; border-style:initial; border-color:initial">
			<li style="margin-top:0px; margin-right:0px; margin-bottom:0px; margin-left:0px; padding-top:0px; padding-right:0px; padding-bottom:0px; padding-left:0px; border-top-width:0px; border-right-width:0px; border-bottom-width:0px; border-left-width:0px; border-style:initial; border-color:initial">
				一级list
			</li>
			<li style="margin-top:0px; margin-right:0px; margin-bottom:0px; margin-left:0px; padding-top:0px; padding-right:0px; padding-bottom:0px; padding-left:0px; border-top-width:0px; border-right-width:0px; border-bottom-width:0px; border-left-width:0px; border-style:initial; border-color:initial">
				<ul style="margin-top:0px; margin-right:0px; margin-bottom:0px; margin-left:0px; padding-top:0px; padding-right:0px; padding-bottom:0px; padding-left:30px; border-top-width:0px; border-right-width:0px; border-bottom-width:0px; border-left-width:0px; border-style:initial; border-color:initial">
					<li style="margin-top:0px; margin-right:0px; margin-bottom:0px; margin-left:0px; padding-top:0px; padding-right:0px; padding-bottom:0px; padding-left:0px; border-top-width:0px; border-right-width:0px; border-bottom-width:0px; border-left-width:0px; border-style:initial; border-color:initial">
						二级list
					</li>
					<li style="margin-top:0px; margin-right:0px; margin-bottom:0px; margin-left:0px; padding-top:0px; padding-right:0px; padding-bottom:0px; padding-left:0px; border-top-width:0px; border-right-width:0px; border-bottom-width:0px; border-left-width:0px; border-style:initial; border-color:initial">
						二级list
					</li>
				</ul>
			</li>
			<li style="margin-top:0px; margin-right:0px; margin-bottom:0px; margin-left:0px; padding-top:0px; padding-right:0px; padding-bottom:0px; padding-left:0px; border-top-width:0px; border-right-width:0px; border-bottom-width:0px; border-left-width:0px; border-style:initial; border-color:initial">
				一级list
			</li>
			<li style="margin-top:0px; margin-right:0px; margin-bottom:0px; margin-left:0px; padding-top:0px; padding-right:0px; padding-bottom:0px; padding-left:0px; border-top-width:0px; border-right-width:0px; border-bottom-width:0px; border-left-width:0px; border-style:initial; border-color:initial">
				一级list
			</li>
		</ul>
	</blockquote>
	
再来看一下生成的html代码

<pre style="margin-top:15px; margin-bottom:15px; padding:6px 10px; border:1px solid rgb(204,204,204); font-size:13px; font-family:Consolas,'Liberation Mono',Courier,monospace; background-color:rgb(248,248,248); line-height:19px; overflow:auto"><code style="margin:0px; padding:0px; font-size:12px; font-family:Consolas,'Liberation Mono',Courier,monospace; background-color:transparent">&lt;ul&gt;
&lt;li&gt;一级list&lt;/li&gt;
li&gt;&lt;ul&gt;
&lt;li&gt;二级list&lt;/li&gt;
&lt;li&gt;二级list&lt;/li&gt;
&lt;/ul&gt;&lt;/li&gt;
&lt;li&gt;一级list&lt;/li&gt;
&lt;li&gt;一级list&lt;/li&gt;
&lt;/ul&gt;
</code></pre>

# 第四 markdown专注于文本

这一点上，有人问到mardown不能选择文字样式和其他复杂样式。其实markdown的介绍上就已经说明，markdown专注于本文而不是样式，而样式同样是用css

----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)


