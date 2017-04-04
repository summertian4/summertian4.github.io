title: iOS进阶学习——React-Native-环境搭建-run-it
date: 2016-04-19 11:08:06
tags:
  - iOS
categories:
  - iOS
---

# 官方参考
1. [react-native github](https://github.com/facebook/react-native)
2. [react-native 中文文档](http://reactnative.cn/docs/0.24/getting-started.html#content)
3. [node.js](https://nodejs.org/en/)
4. [nvm](https://github.com/creationix/nvm#installation)
5. [watchman](https://facebook.github.io/watchman/docs/install.html)
6. [flow](http://www.flowtype.org)

# 环境搭建
## Node.js安装
1. nvm istall
    *  输入命令`curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.31.0/install.sh | bash `
    *  如果你希望卸载nvm，你可以输入命令` $ npm uninstall -g a_module`
2. 输入命令`nvm install node && nvm `
3. 使nvm生效：`. ~/.nvm/nvm.sh` 
4. 安装Node.js`nvm install node && nvm alias default node`

<!--more-->

## watchman安装
1. `brew install watchman`
## flow
`brew install flow`

## 快速使用React Native
1. 安装React Native
    * `npm install -g react-native-cli` 
2. 创建React Native工程
    * 进入你常用的workspace，输入命令`react-native init AwesomeProject`
    * 稍等会自动帮你创建好工程，大概是这样：
    ![图片](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E8%BF%9B%E9%98%B6%E5%AD%A6%E4%B9%A0%E2%80%94%E2%80%94React-Native-%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA-run-it-01.png-w375)
3. 打开`iOS`文件夹，发现里面是熟悉的常规工程目录，打开工程，在模拟器上运行（稍后会说明如何在真机运行），效果如下 

	![图片](http://7xt4xp.com1.z0.glb.clouddn.com/blog_iOS%E8%BF%9B%E9%98%B6%E5%AD%A6%E4%B9%A0%E2%80%94%E2%80%94React-Native-%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA-run-it-02.png-w375)

4. 真机运行
	* 观察项目`AppDelegate.m`你可以发现`didFinishLaunchingWithOptions`中有这样一句
	
	```objc
	jsCodeLocation = [NSURL URLWithString:@"http://localhost:8081/index.ios.bundle?platform=ios&dev=true"];
	```
	所以简单的来说，要保证你的真机和mac在同一个网络，将localhost换成mac的IP地址
    
# 可能遇到的问题
## react-native命令行慢
老问题了……替换npm仓库源，换成国内的源

```objc
npm config set registry https://registry.npm.taobao.org
npm config set disturl https://npm.taobao.org/dist
```
## watchman无法启动
1. 命令台报错：ERROR  watchman--no-pretty get-sockname returned with exit code null dyld: Library not loaded: /usr/local/lib/libpcre.1.dylib
      Referenced from: /usr/local/bin/watchman
      Reason: image not found
2. 解决
    brew uninstall watchman
    brew install --HEAD watchman
    

----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)
    

