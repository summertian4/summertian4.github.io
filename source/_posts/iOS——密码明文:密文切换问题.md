title: iOS——密码明文/密文切换问题
date: 2016-03-25 15:35:27
tags:
  - iOS
categories:
  - iOS
---
前段时间根据产品经理的要求给我们输入密码的部分加了明文/密文切换，中间也遇到了一些颇有意思的问题。其中也有些很难查到资料。

在这里记录下来，也供大家参考，避免大家重复踩坑。

# 情景描述
明文/密文切换，就是输入密码的时候可以选择`明文显示`还是`**`这样的显示。

![](http://7xnrog.com1.z0.glb.clouddn.com/blog_iOS%E2%80%94%E2%80%94%E5%AF%86%E7%A0%81%E6%98%8E%E6%96%87%3A%E5%AF%86%E6%96%87%E5%88%87%E6%8D%A2%E9%97%AE%E9%A2%98-01.jpg-w375)

![](http://7xnrog.com1.z0.glb.clouddn.com/blog_iOS%E2%80%94%E2%80%94%E5%AF%86%E7%A0%81%E6%98%8E%E6%96%87%3A%E5%AF%86%E6%96%87%E5%88%87%E6%8D%A2%E9%97%AE%E9%A2%98-02.jpg-w375)

右侧的按钮可以切换明文、密文模式

<!--more-->

# UITextField明文\密文切换属性的属性

```objc
@property(nonatomic,getter=isSecureTextEntry) BOOL secureTextEntry;       // default is NO
```

# Q1：光标位置错乱
一般来说密文的时候*号要比字母更宽，当密文切换成明文的时候光标的位置居然没有变化，出现了这样的情况。
![嗯 没有用我们自己的app了，写了个demo，样式很简约](http://7xnrog.com1.z0.glb.clouddn.com/blog_iOS%E2%80%94%E2%80%94%E5%AF%86%E7%A0%81%E6%98%8E%E6%96%87%3A%E5%AF%86%E6%96%87%E5%88%87%E6%8D%A2%E9%97%AE%E9%A2%98-03.png-w375)

这个问题在查了一些资料之后发现可能是苹果自己的BUG，当然，对应方法是有的。我们可以在切换代码前将textfiled的enable设为NO，切换后在设置YES。当然，这回让textfiled退出编辑模式。

如果你有更好的方式，欢迎交流，或者在博文后留言😘
```objc
self.tfPassword.enabled = NO;
self.tfPassword.secureTextEntry = !self.tfPassword.secureTextEntry;
self.tfPassword.enabled = YES;
```

# Q2：光标位置错乱
当UITextField经历 `明文->密文->明文` 时，再次输入，无论你输入什么，都会将所有输入清空。

嗯，这确实是个头疼的问题，也没有任何理由，因为UITextField本身如此，而且当时真的想不到任何办法。

最后终于解决。

思路是这样的：我们都只到UITextField的代理UITextFieldDelegate中有方法`- (BOOL)textField:(UITextField *)textField shouldChangeCharactersInRange:(NSRange)range replacementString:(NSString *)string;` 相信每个人都会常用，通常我们用来抓用户输入的文字，在每次textfield发生字符改变的时候。

但是我们忽略了这个方法的本身作用，注意返回值，这个方法本身是用来返回`『是否允许改变textfield字符』`。

所以只要在这里做判断：

```objc
- (BOOL)textField:(UITextField *)textField shouldChangeCharactersInRange:(NSRange)range replacementString:(NSString *)string {
    
    //string就是此时输入的那个字符
    //得到输入框的内容
    NSString * toBeString = [textField.text stringByReplacingCharactersInRange:range withString:string];
    if (textField == _tfPassword && textField.isSecureTextEntry ) {
        textField.text = toBeString;
        return NO;
    }
    
    return YES;
}
```
完美解决


# 代码
唔，这篇也给个代码吧，其实只有几行。
http://download.csdn.net/detail/u010127917/9472703

# 其他

其实很多奇怪的问题只有在实际开发的时候发现，这时候你就会认识到自己的经验不足。所以啦，学无止境~。

过段时间妹子我会奉上自己的照片哦~（好久没自拍啦~~😲）

----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)


