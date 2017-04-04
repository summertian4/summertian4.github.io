title: 找回丢失的 Github（谨慎保管 2FA 验证）
date: 2017-02-07 14:53:14
tags:
  - Other
  - Github
categories:
  - Other
  - Github
---

# Why 我丢失了 Github 账号

从前天，经过了两天时间与 Github 客服对话，终于找回了我的 Github。

朋友在问我为什么登陆不了 Github 的时候表示不太理解。

原因很简单，因为我丢失了两步验证的 app，也丢失了 recovery code。

<!-- More -->

# 2FA 和基于时间戳的实时验证码

其实，是这样的。Github 也有两步验证（2FA），提供了两种可选方式：

1. 使用手机接收短信
2. 使用由时间戳生成的 2FA 实时验证码生成

由于方法 1 无法支持国内的手机号，所以只能使用方法 2。

方法二的原理是，Github 会提供一个秘钥，通常以二维码的方式显示，用三方软件比如 Autenticator、1password（当时我不知道 1password 也可以） 在扫码之后，将秘钥保存到客户端。

在用户登陆的时候，app 会以秘钥和时间戳为参数，通过固定算法生成一个 6 位数字的一次验证码。

服务端通过同样的算法也会生成一个同样的一次性验证码，两者对比一致，则通过验证。和手机短信接收验证码很类似。

Github 在开启 2FA 的同时，会提供给你一份 recovery code，如果你无法拿到一次验证码，就使用 recovery code 暂时通过验证。recovery code 只能使用一次，使用过一次以后，就会被更新，你需要保存新的 recovery code。

当时我选择验证码生成 app 是 Google 的 Autenticator。就是这个 app 坑了我。这个 app 不需要登陆，我直接扫码就记录了，但是我当时用的 iPhone 6，在更换了 iPhone 7 之后，iPhone 6 借给他人使用前做了抹除，So，我没有了 app，只能通过 recovery code 恢复，但是奇怪的事，我使用三个月内我印象里最新的 recovery code 去验证，但是失败了。

在彻底折腾一番发现没办法后，去联系了 Github 客服。

# Github 客服

当你没有 2FA app，也丢失了 recovery code 之后，你必须去联系客服，请求帮助你关闭 2FA。

这时候客服会要求你运行一段 command，以证明的电脑使用过此账号的公钥。

```
ssh -T git@github.com verify
```

但是我怎样也得不到结果，要么提示 DNS 劫持（和公司的网络翻墙了有关），要么其他的种种问题。

在和 Github 客服一番对话后，对方表示验证的 ssh 公钥是应由 Github Destop 软件生成了。当时很懵逼，问对方『难道我自己生成的秘钥就不是秘钥了？难道不能证明我电脑生成的这个公钥？』。然后客服就说『你 2015 年用一台 Macbook Pro 生成了一个公钥，这是你的电脑吗？』，然后 run 一下：

```
ssh -i ~/.ssh/github_rsa -T git@github.com verify
```

然后 run 之后：

```
Enter passphrase for key '/Users/zhoulingyu/.ssh/github_rsa':
```

当时我想了一下，傻傻的输入了 Github 密码，然而不对。然后我让几个朋友 run 了一下，均是直接显示出了 verification token。我很奇怪，于是想了一下，发现这里要求输入的密码应该是当初生成公钥的时候设置的密码，通常很多人都会选择不设置密码，但是我显然当初设置了，然而我记不起来。

一番搜索之后，得到了一个 happy 的结果，如果你是 windows，那洗洗睡吧，如果是 Mac，这个密码可以在 keychain 中找到，具体方法在[这里](https://help.github.com/articles/recovering-your-ssh-key-passphrase/)。

我从 keychain 中粘出密码后我就惊呆了，是一个 40 多位的密码。显然是自动生成的高复杂度密码。我当时一定是忘了保存。

SO，拿到 verification token 之后，Github 客服就帮我关掉了 2FA。

随后，我发现 1password 是可以生成一次性验证码的，于是使用 1password 保存，不在使用 Google 的 app。

# Other

这篇文章就是做一个记录，自己弄了两天，都去按照客服意见新建了 Gihub 账号 Fork 了原来所有的项目。最终找到的时候也是喜出望外。

也希望这篇记录能帮到其他人。


----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)




