title: 初次整合Hibernate4和Spring4中出现的细节问题
date: 2014-06-15 11:44
tags:
  - JAVA
categories:
  - JAVA
---

# 一、第一个错误
首先出现的是
`org.springframework.beans.factory.NoSuchBeanDefinitionException: No bean named 'userService' is defined`


在UerDAOImpl中如下配置

```java
@Component("userDao")
public class UserDAOImpl implements UserDAO {
 
   private SessionFactory sessionFactory;
 
   public SessionFactory getSessionFactory() {
      return sessionFactory;
   }
 
   @Resource
   public void setSessionFactory(SessionFactory sessionFactory) {
      this.sessionFactory = sessionFactory;
   }
       Connection conn = null;
 
      Session s = sessionFactory.openSession();
      s.beginTransaction();
      s.save(user);
      s.getTransaction().commit();
      s.close();
      System.out.println("user saved!");
   }
}
```

发现没有配置错误的地方
最后发现原因是没有在beans.xml中配置扫描器
 
因为使用的Annotation所以必须要要扫描所有的源码

<!--more-->

# 二、第二个错误
修改完成后还是出现同样的错误
查了一下相关资料，原来发现hibernate4已经将hibernate3的一些功能改掉了，在hibernate4已经不使用CacheProvider了，所以做了以下修改，
原先：
 `class="org.springframework.orm.hibernate3.annotation.AnnotationSessionFactoryBean">`
改成：
  `class="org.springframework.orm.hibernate4.LocalSessionFactoryBean">`
问题解决，发现可以正常使用了
参考链接http://stackoverflow.com/questions/8565051/spring-3-1-hibernate-4-sessionfactory

----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[小鱼周凌宇](http://weibo.com/coderfish/)


