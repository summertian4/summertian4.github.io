title: SSH2如何用Cookie做两周免登陆效果
date: 2014-07-25 20:00
tags:
  - JAVA
categories:
  - JAVA
---


# 简介

今天将会做一个Cookie的Demo，也是我今天在写项目时刚刚学的小技术，这里将会展示最简单的实现Cookie的方法，以及JAVAEE中Cookie的使用。

# Demo

需要注意的时，这里贴出的源码是我项目中的一部分，不需要大家具体理解其中一些变量的意思，只需要关注Cookie部分。

## 流程

点击页面中登录，链接到登录action
登录时选中“两周免登陆”选框，输入用户名密码，点击登录
存入Cookie
下次无需登录

## 登录页面表单


<pre style="margin-top:15px; margin-bottom:15px; padding:6px 10px; border:1px solid rgb(204,204,204); font-size:13px; font-family:Consolas,'Liberation Mono',Courier,monospace; background-color:rgb(248,248,248); line-height:19px; overflow:auto"><code style="margin:0px; padding:0px; font-size:12px; font-family:Consolas,'Liberation Mono',Courier,monospace; background-color:transparent">&lt;form class=&quot;registed&quot; action=&quot;User_login&quot; method=&quot;post&quot;&gt;
&lt;h2&gt;登陆&lt;/h2&gt;
&lt;div class=&quot;email&quot;&gt;
    &lt;strong&gt;用户名&lt;/strong&gt;&lt;sup class=&quot;surely&quot;&gt;*&lt;/sup&gt;&lt;br /&gt;
&lt;input type=&quot;text&quot; name=&quot;username&quot; value=&quot;&quot; /&gt;
&lt;/div&gt;
&lt;div class=&quot;password&quot;&gt;
    &lt;strong&gt;密码&lt;/strong&gt;&lt;sup class=&quot;surely&quot;&gt;*&lt;/sup&gt;&lt;br /&gt;
&lt;input type=&quot;password&quot; name=&quot;password&quot; value=&quot;&quot; /&gt;
    &lt;a class=&quot;forgot&quot; href=&quot;#&quot;&gt;忘记密码?&lt;/a&gt;
&lt;/div&gt;
&lt;!-- .password --&gt;
&lt;div class=&quot;remember&quot;&gt;
&lt;input class=&quot;niceCheck&quot; type=&quot;checkbox&quot; name=&quot;useCookie&quot;  value=&quot;true&quot;/&gt;
    &lt;span class=&quot;rem&quot;&gt;两周免登陆&lt;/span&gt;
&lt;/div&gt;
&lt;!-- .remember --&gt;

&lt;div class=&quot;submit&quot;&gt;
    &lt;input type=&quot;submit&quot; value=&quot;登陆&quot; /&gt;
&lt;/div&gt;
    &lt;!-- .submit --&gt;
&lt;/form&gt;
</code></pre>

注意action为User_login

<!--more-->

## action代码

```java
/**
 * 用户登陆
 * 
 * @return
 */
public String login() {
    /**
     * 获取response
     */
    HttpServletResponse response = ServletActionContext.getResponse();
    //如果查询为空
    if (userDao.login(username, password) == null) {
        return ERROR;
    }
    //如果用户存在 
    else {
        //将用户放到session中
        session.put("user", userDao.login(username, password));
        //如果useCookie不为空
        if(useCookie != null){
            //新建一个Cookie，存放用户名和密码，用&做分隔符
            Cookie cookie = new Cookie("userCookie", username+"&"+password);
            //设置Cookie的存活时间（这里是两周）
            cookie.setMaxAge(2*7*24*60*60);
            //添加Cookie
            response.addCookie(cookie);
        }
        return SUCCESS;
    }
}
```

注意这里useCookie是一个字符串，就是从页面表单中接收的单选框的value。 这里做的就是如果勾了选框，则保存Cookie，不选中就不保存。

相应的struts.xml中配置

```xml
<action name="User_login" class="com.webstore.action.UserAction"
        method="login">
    <result name="success">/index.jsp</result>
    <result name="error">/login.jsp</result>
</action>
```

这里附加一个方法，用于在点击登录连接时进行Cookie检测

```java
/**
 * 登录前检查cookie
 * 
 * @return
 */
public String cookieDetection() {
    if (session.get("user") != null) {
        return SUCCESS;
    } else {
        return LOGIN;
    }
}
```

相应的struts.xml中配置

```xml
<action name="User_cookieDetection" class="com.webstore.action.UserAction"
        method="cookieDetection">
    <result name="success">/index.jsp</result>
    <result name="login">/login.jsp</result>
</action>
```

## 首页过滤器

Cookie在什么时候检查呢，正常的设想是，在首页时就检查好是否有用户已经“两周免登陆”，那么需要有一个东西能够做到这一点，一般能想到的两个机制，过滤器和监听器。 监听器智能监听相应的动作，然后同时进行操作，而过滤器可以直接对request和response进行过滤，在Cookie的使用中会用到response和request，所以最好选择过滤器。

```java
import java.io.IOException;
import java.util.Map;

import javax.servlet.Filter;
import javax.servlet.FilterChain;
import javax.servlet.FilterConfig;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.http.Cookie;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.apache.struts2.ServletActionContext;

import com.webstore.dao.IUserDao;
import com.webstore.imp.UserImp;

/**
 * 作者：周凌宇 时间：2014-7-4 描述：Cookie过滤器
 */
public class CookieFilter implements Filter {

    /**
     * 配置对象
     */
    protected FilterConfig config;

    /**
     * 初始化过滤器
     */
    public void init(FilterConfig config) {
        this.config = config;
    }

    /**
     * 重写doFilter方法，过滤所有请求和回复，加入或取出Cookie
     */
    public void doFilter(ServletRequest req, ServletResponse resp,
            FilterChain chain) throws IOException, ServletException {

        HttpServletRequest request = (HttpServletRequest) req;
        HttpServletResponse response = (HttpServletResponse) resp;
        Cookie[] cookies = request.getCookies();
        IUserDao userDao = new UserImp();
        String[] info = null;
        if (cookies != null) {
            for (Cookie c : cookies) {
                info = c.getValue().split("&");
                if (info.length == 2) {
                    String username = info[0];
                    String password = info[1];
                    request.getSession().setAttribute("user", userDao.login(username, password));
                }
            }
        }
        chain.doFilter(request, response);
    }

    /**
     * 销毁过滤器
     */
    public void destroy() {
        this.config = null;
    }
}
```

## 过滤器配置

```xml
<filter>
    <filter-name>CookieFilter</filter-name>
    <filter-class>com.webstore.util.CookieFilter</filter-class>
</filter>
<filter-mapping>
    <filter-name>CookieFilter</filter-name>
    <url-pattern>/index.jsp</url-pattern>
</filter-mapping>
```

注意这里配置了只对首页过滤


----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)


